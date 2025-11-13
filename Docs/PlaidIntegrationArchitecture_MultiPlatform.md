# Plaid Integration Architecture - Multi-Platform (iOS & macOS)

## Overview

This document outlines the revised architecture for integrating Plaid to support importing bank transactions from all U.S. banks, with full support for both **iOS** and **macOS** platforms.

## Platform Strategy: WebView for Both

### Decision: WebView for iOS and macOS

We've decided to use **WebView-based Plaid Link for both iOS and macOS** for:
- **Simplicity**: Single implementation, easier to maintain
- **Consistency**: Same user experience across all platforms
- **Maintainability**: One code path to test and debug
- **Future-proof**: Works on visionOS, iPadOS, etc. without changes
- **No SDK dependency**: Smaller app size, simpler builds

### Implementation
- **iOS**: Plaid Link in WKWebView (SwiftUI WebView)
- **macOS**: Plaid Link in WKWebView (NSViewRepresentable)
- **Same code**: Minor platform-specific differences (frame sizing)

**Key Insight**: Plaid's API is platform-agnostic. The Link UI uses WebView on both platforms for consistency and simplicity. All core API calls, token management, and transaction sync are identical.

---

## Revised Architecture Design

### 1. Feature Structure (Updated)

```
MoneyFlow/
├── Features/
│   ├── Plaid/                          
│   │   ├── Provider/
│   │   │   ├── PlaidAccountProvider.swift
│   │   │   └── PlaidAccountFeatureProvider.swift
│   │   ├── Services/
│   │   │   ├── PlaidService.swift              # Core API (platform-agnostic)
│   │   │   ├── PlaidLinkService.swift          # WebView-based Link integration (both platforms)
│   │   │   ├── PlaidTransactionSyncService.swift # Sync logic (platform-agnostic)
│   │   │   └── PlaidTokenManager.swift          # Token storage (platform-agnostic)
│   │   ├── Data/
│   │   │   ├── PlaidRepository.swift
│   │   │   └── PlaidModels.swift
│   │   ├── Domain/
│   │   │   └── PlaidImportParameters.swift
│   │   └── Import/
│   │       ├── PlaidLinkView.swift             # Unified WebView Link UI (both platforms)
│   │       └── PlaidAccountConfigurationView.swift
│   └── CMB/
│       └── ...
```

### 2. WebView-Based Link Service (Unified for Both Platforms)

```swift
/// Platform-agnostic protocol for Plaid Link integration
protocol PlaidLinkServiceProtocol {
    /// Create a link token for initiating Link flow
    func createLinkToken(config: PlaidLinkConfig) async throws -> String
    
    /// Open Link flow (platform-specific implementation)
    func openLink(
        linkToken: String,
        onSuccess: @escaping (String) -> Void,  // public_token
        onExit: @escaping (PlaidLinkExit?) -> Void
    )
    
    /// Check if Link is available on current platform
    static var isAvailable: Bool { get }
}
```

#### 2.2 iOS Implementation

```swift
#if os(iOS)
import Plaid

struct PlaidLinkService_iOS: PlaidLinkServiceProtocol {
    static var isAvailable: Bool { true }
    
    func createLinkToken(config: PlaidLinkConfig) async throws -> String {
        // Call Plaid API to create link token
        // Returns link_token for Link SDK
    }
    
    func openLink(linkToken: String, onSuccess: @escaping (String) -> Void, onExit: @escaping (PlaidLinkExit?) -> Void) {
        // Use Plaid Link iOS SDK
        let linkConfiguration = LinkTokenConfiguration(token: linkToken) { result in
            switch result {
            case .success(let publicToken):
                onSuccess(publicToken)
            case .failure(let error):
                onExit(PlaidLinkExit(error: error))
            }
        }
        
        let linkHandler = Plaid.create(linkConfiguration)
        linkHandler.open(presentUsing: viewController)
    }
}
#endif
```

#### 2.3 macOS Implementation

```swift
#if os(macOS)
import SwiftUI
import WebKit

struct PlaidLinkService_macOS: PlaidLinkServiceProtocol {
    static var isAvailable: Bool { true }
    
    func createLinkToken(config: PlaidLinkConfig) async throws -> String {
        // Call Plaid API to create link token (same as iOS)
        // Returns link_token for web Link
    }
    
    func openLink(linkToken: String, onSuccess: @escaping (String) -> Void, onExit: @escaping (PlaidLinkExit?) -> Void) {
        // Use Plaid Link web flow in WebView
        // Load Plaid Link URL: https://cdn.plaid.com/link/v2/stable/link.html
        // Pass link_token as parameter
        // Listen for postMessage events from Plaid
        // Extract public_token from success event
    }
}
#endif
```

### 3. Platform-Specific Link View

#### 3.1 iOS: Native Link View

```swift
#if os(iOS)
import SwiftUI
import Plaid

struct PlaidLinkView_iOS: View {
    let linkToken: String
    let onSuccess: (String) -> Void
    let onExit: (PlaidLinkExit?) -> Void
    
    var body: some View {
        PlaidLinkViewRepresentable(
            linkToken: linkToken,
            onSuccess: onSuccess,
            onExit: onExit
        )
    }
}

struct PlaidLinkViewRepresentable: UIViewControllerRepresentable {
    let linkToken: String
    let onSuccess: (String) -> Void
    let onExit: (PlaidLinkExit?) -> Void
    
    func makeUIViewController(context: Context) -> UIViewController {
        let config = LinkTokenConfiguration(token: linkToken) { result in
            switch result {
            case .success(let publicToken):
                onSuccess(publicToken)
            case .failure(let error):
                onExit(PlaidLinkExit(error: error))
            }
        }
        let handler = Plaid.create(config)
        return handler.createViewController()
    }
    
    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
#endif
```

#### 3.2 macOS: WebView Link View

```swift
import SwiftUI
import WebKit

/// Unified Plaid Link view using WebView (works on both iOS and macOS)
struct PlaidLinkView: View {
    let linkToken: String
    let onSuccess: (String) -> Void
    let onExit: (PlaidLinkExit?) -> Void
    
    var body: some View {
        PlaidLinkWebView(
            linkToken: linkToken,
            onSuccess: onSuccess,
            onExit: onExit
        )
        #if os(iOS)
        .frame(minWidth: 375, minHeight: 600)
        #elseif os(macOS)
        .frame(minWidth: 600, minHeight: 500)
        #endif
    }
}

#if os(iOS)
struct PlaidLinkWebView: UIViewRepresentable {
    let linkToken: String
    let onSuccess: (String) -> Void
    let onExit: (PlaidLinkExit?) -> Void
    
    func makeUIView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        
        // Load Plaid Link
        let plaidLinkURL = "https://cdn.plaid.com/link/v2/stable/link.html?isWebview=true"
        let urlString = "\(plaidLinkURL)&token=\(linkToken)"
        
        if let url = URL(string: urlString) {
            webView.load(URLRequest(url: url))
        }
        
        return webView
    }
    
    func updateUIView(_ uiView: WKWebView, context: Context) {}
    
    func makeCoordinator() -> Coordinator {
        Coordinator(onSuccess: onSuccess, onExit: onExit)
    }
    
    class Coordinator: NSObject, WKNavigationDelegate {
        let onSuccess: (String) -> Void
        let onExit: (PlaidLinkExit?) -> Void
        
        init(onSuccess: @escaping (String) -> Void, onExit: @escaping (PlaidLinkExit?) -> Void) {
            self.onSuccess = onSuccess
            self.onExit = onExit
        }
        
        func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
            // Handle Plaid Link events via postMessage
            // Listen for 'PLAID_EVENT' messages
            // Extract public_token from success events
            decisionHandler(.allow)
        }
    }
}
#elseif os(macOS)
struct PlaidLinkWebView: NSViewRepresentable {
    let linkToken: String
    let onSuccess: (String) -> Void
    let onExit: (PlaidLinkExit?) -> Void
    
    func makeNSView(context: Context) -> WKWebView {
        let webView = WKWebView()
        webView.navigationDelegate = context.coordinator
        
        // Load Plaid Link
        let plaidLinkURL = "https://cdn.plaid.com/link/v2/stable/link.html?isWebview=true"
        let urlString = "\(plaidLinkURL)&token=\(linkToken)"
        
        if let url = URL(string: urlString) {
            webView.load(URLRequest(url: url))
        }
        
        return webView
    }
    
    func updateNSView(_ nsView: WKWebView, context: Context) {}
    
    func makeCoordinator() -> Coordinator {
        Coordinator(onSuccess: onSuccess, onExit: onExit)
    }
    
    class Coordinator: NSObject, WKNavigationDelegate {
        let onSuccess: (String) -> Void
        let onExit: (PlaidLinkExit?) -> Void
        
        init(onSuccess: @escaping (String) -> Void, onExit: @escaping (PlaidLinkExit?) -> Void) {
            self.onSuccess = onSuccess
            self.onExit = onExit
        }
        
        func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
            // Handle Plaid Link events via postMessage
            // Listen for 'PLAID_EVENT' messages
            // Extract public_token from success events
            decisionHandler(.allow)
        }
    }
}
#endif
```

### 3. Unified Link View (WebView for Both Platforms)

```swift
// See implementation above - unified WebView for both platforms
```

---

## Platform-Specific Considerations

### iOS & macOS (Both Use WebView)
- ✅ Consistent experience across platforms
- ✅ Single implementation to maintain
- ✅ WebView-based Plaid Link
- ✅ Same functionality on both platforms
- ⚠️ Requires WebView setup (WKWebView)
- ⚠️ Need to handle postMessage events
- ✅ No SDK dependency needed

---

## Shared Components (Platform-Agnostic)

All these components work identically on both platforms:

### ✅ PlaidService
- Core API calls (HTTP requests)
- Token exchange
- Fetch accounts
- Fetch transactions
- Platform-agnostic

### ✅ PlaidTokenManager
- Keychain storage (works on both iOS and macOS)
- Token management
- Platform-agnostic

### ✅ PlaidTransactionSyncService
- Transaction sync logic
- Duplicate detection
- Domain model mapping
- Platform-agnostic

### ✅ PlaidRepository
- Data persistence
- Sync date tracking
- Platform-agnostic

### ✅ PlaidModels
- Data models
- API response parsing
- Platform-agnostic

---

## Implementation Strategy

### Phase 1: Foundation (Platform-Agnostic)
- ✅ PlaidService (already done)
- ✅ PlaidTokenManager (already done)
- ✅ PlaidModels (already done)
- ✅ PlaidError enum (already done)

### Phase 2: Link Integration (WebView for Both)
1. Create `PlaidLinkServiceProtocol`
2. Implement `PlaidLinkService` (WebView-based, unified)
3. Create `PlaidLinkView` with WebView (iOS: UIViewRepresentable, macOS: NSViewRepresentable)
4. Implement postMessage event handling
5. Handle link_token creation and exchange

### Phase 3: Transaction Sync (Platform-Agnostic)
- PlaidTransactionSyncService (same as before)
- Works on both platforms

### Phase 4: Polish & Testing
- Test on both iOS and macOS
- Platform-specific UI refinements

---

## Key Design Decisions

### 1. Protocol-Based Platform Abstraction
- `PlaidLinkServiceProtocol` provides unified interface
- Platform-specific implementations behind protocol
- Easy to test and maintain

### 2. Shared Core Logic
- All API calls, token management, sync logic are platform-agnostic
- Only Link UI differs between platforms
- Maximum code reuse

### 3. SwiftUI Platform Conditionals
- Use `#if os(iOS)` and `#if os(macOS)` for platform-specific code
- Unified views that adapt to platform
- Clean separation of concerns

### 4. WebView for Both Platforms
- Uses Plaid's official web Link flow
- Same security and functionality
- Consistent experience across platforms
- Single implementation for both iOS and macOS

---

## macOS WebView Implementation Details

### Plaid Link Web Flow
1. Create link_token via API (same as iOS)
2. Load Plaid Link in WebView: `https://cdn.plaid.com/link/v2/stable/link.html`
3. Pass `link_token` as query parameter
4. Listen for postMessage events:
   - `PLAID_EVENT` with event type
   - `PUBLIC_TOKEN` in success event
5. Extract public_token and exchange for access_token

### Event Handling
```swift
// Listen for postMessage from Plaid
webView.evaluateJavaScript("""
    window.addEventListener('message', function(event) {
        if (event.data && event.data.plaidEvent) {
            // Handle Plaid events
            if (event.data.plaidEvent.eventName === 'EXIT') {
                // Handle exit
            } else if (event.data.plaidEvent.eventName === 'SUCCESS') {
                // Extract public_token
                var publicToken = event.data.plaidEvent.publicToken;
                // Send to Swift via message handler
            }
        }
    });
""")
```

---

## Testing Strategy

### iOS Testing
- Test with Plaid Link SDK
- Use sandbox credentials
- Verify native UI flow

### macOS Testing
- Test WebView implementation
- Verify postMessage handling
- Test in macOS app window

### Shared Testing
- Unit tests for platform-agnostic code
- Mock Link service for testing
- Integration tests for API calls

---

## Migration from iOS-Only

Since we already have Phase 1 implemented:

1. **Keep existing code** (it's already platform-agnostic)
2. **Add platform-specific Link service**:
   - Create `PlaidLinkServiceProtocol`
   - Implement iOS version (uses SDK)
   - Implement macOS version (uses WebView)
3. **Update views** to use unified `PlaidLinkView`
4. **Test on both platforms**

---

## Benefits of This Architecture

✅ **Platform Support**: Full iOS and macOS support
✅ **Code Reuse**: Maximum shared code (90%+)
✅ **Maintainability**: Clear separation of platform-specific code
✅ **Consistency**: Same functionality on both platforms
✅ **Flexibility**: Easy to add new platforms (visionOS, etc.)
✅ **User Experience**: Native feel on each platform

---

## Next Steps

1. ✅ Phase 1 complete (platform-agnostic foundation)
2. ⏳ Phase 2: Implement platform-specific Link services
3. ⏳ Phase 3: Transaction sync (shared)
4. ⏳ Phase 4: Testing and polish

---

## Conclusion

This revised architecture fully supports both iOS and macOS while maximizing code reuse. The key insight is that only the Link UI differs between platforms - all core functionality is platform-agnostic and can be shared.

