# Plaid Integration Architecture Proposal

## Overview

This document outlines the architecture redesign for integrating Plaid to support importing bank transactions from all U.S. banks. Plaid provides a unified API for connecting to 11,000+ financial institutions, which will significantly expand MoneyFlow's capabilities beyond the current CMB (China Merchants Bank) import feature.

## Key Differences: Plaid vs CMB

| Aspect | CMB (Current) | Plaid (New) |
|--------|---------------|-------------|
| **Data Source** | PDF files (manual upload) | API (real-time) |
| **Authentication** | ZIP password | OAuth-style Link flow |
| **Sync Model** | Manual, one-time | Automatic, periodic |
| **Token Management** | Not needed | Access tokens required |
| **Account Discovery** | Manual entry | Automatic via Link |
| **Multi-Account** | Single account per file | Multiple accounts per institution |

## Architecture Design

### 1. Feature Structure

```
MoneyFlow/
├── Features/
│   ├── Plaid/                          # NEW: Plaid feature module
│   │   ├── Provider/
│   │   │   ├── PlaidAccountProvider.swift       # Account creation via Link
│   │   │   └── PlaidAccountFeatureProvider.swift # Feature provider implementation
│   │   ├── Services/
│   │   │   ├── PlaidService.swift              # Core API service
│   │   │   ├── PlaidLinkService.swift          # Link integration
│   │   │   ├── PlaidTransactionSyncService.swift # Sync logic
│   │   │   └── PlaidTokenManager.swift          # Secure token storage
│   │   ├── Data/
│   │   │   ├── PlaidRepository.swift           # Plaid-specific data storage
│   │   │   └── PlaidModels.swift                # Plaid API models
│   │   ├── Domain/
│   │   │   └── PlaidImportParameters.swift      # Import parameters
│   │   └── Import/
│   │       ├── PlaidLinkView.swift              # Link UI integration
│   │       └── PlaidAccountConfigurationView.swift # Account setup
│   └── CMB/                             # Existing (unchanged)
│       └── ...
```

### 2. Core Components

#### 2.1 PlaidService (Core API Layer)

**Responsibilities:**
- Handle all Plaid API calls
- Manage API authentication
- Transform Plaid API responses to domain models
- Handle rate limiting and retries

**Key Methods:**
```swift
protocol PlaidServiceProtocol {
    func exchangePublicToken(_ token: String) async throws -> PlaidAccessToken
    func fetchAccounts(accessToken: String) async throws -> [PlaidAccount]
    func fetchTransactions(
        accessToken: String,
        startDate: Date,
        endDate: Date
    ) async throws -> [PlaidTransaction]
    func refreshAccessToken(_ token: String) async throws -> PlaidAccessToken
}
```

#### 2.2 PlaidLinkService (Link Integration)

**Responsibilities:**
- Integrate Plaid Link SDK
- Handle Link flow (open, success, error)
- Manage Link configuration

**Key Methods:**
```swift
protocol PlaidLinkServiceProtocol {
    func openLink(
        for accountType: AccountType,
        onSuccess: @escaping (String) -> Void,  // public_token
        onExit: @escaping (PlaidLinkExit?) -> Void
    )
    func createLinkToken(config: PlaidLinkConfig) async throws -> String
}
```

#### 2.3 PlaidTokenManager (Security Layer)

**Responsibilities:**
- Securely store access tokens in Keychain
- Manage token refresh
- Handle token expiration
- Support multiple tokens (one per Plaid item)

**Key Methods:**
```swift
protocol PlaidTokenManagerProtocol {
    func saveAccessToken(_ token: String, for itemId: String) throws
    func getAccessToken(for itemId: String) throws -> String?
    func refreshToken(for itemId: String) async throws -> String
    func deleteToken(for itemId: String) throws
}
```

#### 2.4 PlaidTransactionSyncService (Sync Logic)

**Responsibilities:**
- Coordinate transaction fetching
- Handle incremental sync (only new transactions)
- Detect duplicates
- Map Plaid transactions to domain models
- Handle account balance updates

**Key Methods:**
```swift
protocol PlaidTransactionSyncServiceProtocol {
    func syncTransactions(
        for account: Account,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult
    func getLastSyncDate(for accountId: UUID) -> Date?
    func setLastSyncDate(_ date: Date, for accountId: UUID)
}
```

#### 2.5 PlaidAccountFeatureProvider (Feature Provider)

**Responsibilities:**
- Implement `AccountFeatureProvider` protocol
- Provide Plaid-specific UI (Link button, sync button)
- Handle import via Plaid service
- Support auto-sync configuration

**Implementation:**
```swift
struct PlaidAccountFeatureProvider: AccountFeatureProvider, AccountImportProvider {
    let id = "plaid-feature-provider"
    let supportedInstitution = "Plaid"  // Or detect by account metadata
    
    func supports(account: Account) -> Bool {
        // Check if account was created via Plaid (e.g., metadata flag)
        account.metadata?["source"] == "plaid"
    }
    
    func makeActionButtons(for account: Account) -> AnyView? {
        AnyView(
            VStack {
                PlaidSyncButton(account: account)
                PlaidReconnectButton(account: account)  // If token expired
            }
        )
    }
    
    func performImport(...) async throws -> ImportResult {
        // Use PlaidTransactionSyncService
    }
}
```

### 3. Data Models

#### 3.1 Plaid-Specific Models

```swift
// PlaidModels.swift

struct PlaidAccessToken: Codable {
    let accessToken: String
    let itemId: String
    let institutionId: String?
    let institutionName: String?
}

struct PlaidAccount: Codable {
    let accountId: String  // Plaid account_id
    let name: String
    let type: String  // "depository", "credit", etc.
    let subtype: String?  // "checking", "savings", etc.
    let balances: PlaidBalances
}

struct PlaidTransaction: Codable {
    let transactionId: String  // Plaid transaction_id
    let accountId: String
    let amount: Decimal
    let date: Date
    let name: String
    let merchantName: String?
    let category: [String]?
    let pending: Bool
}
```

#### 3.2 Account Metadata Extension

Extend `Account` model to store Plaid metadata:

```swift
extension Account {
    // Store Plaid item ID for token lookup
    var plaidItemId: String? {
        get { metadata?["plaidItemId"] }
        set {
            var meta = metadata ?? [:]
            meta["plaidItemId"] = newValue
            metadata = meta
        }
    }
    
    var plaidAccountId: String? {
        get { metadata?["plaidAccountId"] }
        set {
            var meta = metadata ?? [:]
            meta["plaidAccountId"] = newValue
            metadata = meta
        }
    }
    
    var source: String? {
        get { metadata?["source"] }
        set {
            var meta = metadata ?? [:]
            meta["source"] = newValue
            metadata = meta
        }
    }
}
```

**Note:** This requires adding a `metadata: [String: String]?` field to `Account` model.

### 4. Repository Pattern

#### 4.1 PlaidRepository

Store Plaid-specific data separately from core transactions:

```swift
protocol PlaidRepositoryProtocol {
    func saveAccessToken(_ token: PlaidAccessToken, for accountId: UUID) async throws
    func getAccessToken(for accountId: UUID) async throws -> PlaidAccessToken?
    func saveLastSyncDate(_ date: Date, for accountId: UUID) async throws
    func getLastSyncDate(for accountId: UUID) async throws -> Date?
    func savePlaidTransactionMapping(
        plaidTransactionId: String,
        localTransactionId: UUID
    ) async throws
    func getLocalTransactionId(for plaidTransactionId: String) async throws -> UUID?
}
```

### 5. Integration Points

#### 5.1 Account Creation Flow

**Current Flow (CMB):**
```
User selects account type → Configuration form → Upload ZIP → Import
```

**New Flow (Plaid):**
```
User selects account type → Choose provider (DIY or Plaid) → 
If Plaid: Open Link → User connects bank → 
Create Account(s) from Plaid response → Initial sync
```

#### 5.2 Auto-Sync Mechanism

**Options:**
1. **Background Sync** (iOS Background Tasks)
   - Periodic sync when app is backgrounded
   - Limited execution time
   
2. **Foreground Sync** (App Launch/Active)
   - Sync when app becomes active
   - User-initiated sync button
   
3. **Server Webhooks** (Future)
   - Plaid sends webhooks to server
   - Server pushes notifications to app
   - Requires backend infrastructure

**Initial Implementation:** Focus on foreground sync with manual trigger button.

### 6. Security Considerations

#### 6.1 Token Storage

- **Keychain Storage**: Store access tokens in iOS Keychain
- **Item-Specific Tokens**: One token per Plaid item (can have multiple accounts)
- **Token Refresh**: Automatically refresh expired tokens
- **Token Encryption**: Use Keychain's built-in encryption

#### 6.2 API Keys

- **Client ID & Secret**: Store in secure configuration
- **Environment**: Support sandbox and production
- **Key Rotation**: Plan for key rotation without app update

### 7. Error Handling

#### 7.1 Plaid-Specific Errors

```swift
enum PlaidError: LocalizedError {
    case linkCancelled
    case linkFailed(String)
    case tokenExchangeFailed(String)
    case tokenRefreshFailed(String)
    case apiError(String, code: String?)
    case accountNotFound
    case transactionFetchFailed(String)
    case rateLimitExceeded
    case itemLoginRequired  // User needs to reconnect
}
```

#### 7.2 Error Recovery

- **Token Expired**: Prompt user to reconnect via Link
- **API Rate Limit**: Implement exponential backoff
- **Network Error**: Retry with backoff
- **Item Login Required**: Show reconnect button in UI

### 8. User Experience

#### 8.1 Link Integration UI

- **Embedded Link**: Use Plaid Link SDK for seamless flow
- **Loading States**: Show progress during token exchange
- **Error Messages**: Clear, actionable error messages
- **Success Confirmation**: Show connected accounts clearly

#### 8.2 Account Management

- **Account Identification**: Show institution logo/name
- **Sync Status**: Display last sync time
- **Sync Button**: Manual trigger for sync
- **Reconnect Button**: If connection expired
- **Disconnect Option**: Remove Plaid connection

#### 8.3 Transaction Sync

- **Progress Indicators**: Show sync progress
- **Duplicate Handling**: Silent (no UI interruption)
- **New Transactions**: Highlight or badge count
- **Sync History**: Track sync attempts and results

## Migration Strategy

### Phase 1: Foundation (Week 1-2)
1. Add Plaid SDK dependency
2. Create Plaid feature folder structure
3. Implement PlaidService (basic API calls)
4. Implement PlaidTokenManager (Keychain storage)
5. Extend Account model with metadata

### Phase 2: Link Integration (Week 3-4)
1. Integrate Plaid Link SDK
2. Create PlaidLinkView
3. Implement PlaidAccountProvider
4. Update AddAccountView to support Plaid
5. Test Link flow end-to-end

### Phase 3: Transaction Sync (Week 5-6)
1. Implement PlaidTransactionSyncService
2. Create PlaidAccountFeatureProvider
3. Implement duplicate detection
4. Add sync button to account detail view
5. Test sync with real Plaid accounts

### Phase 4: Polish & Testing (Week 7-8)
1. Error handling improvements
2. UI/UX refinements
3. Comprehensive testing
4. Documentation
5. Beta testing

## Technical Dependencies

### Required
- **Plaid iOS SDK**: `https://github.com/plaid/plaid-link-ios`
- **Keychain Services**: Native iOS Keychain API
- **Swift Concurrency**: async/await for API calls

### Optional (Future)
- **Background Tasks**: For auto-sync
- **Combine**: For reactive token refresh
- **Server Backend**: For webhooks (if needed)

## Configuration

### Plaid Environment Setup

```swift
struct PlaidConfig {
    static let clientId: String = {
        // Load from Info.plist or environment
        Bundle.main.object(forInfoDictionaryKey: "PlaidClientId") as? String ?? ""
    }()
    
    static let secret: String = {
        // Load from Keychain or secure storage
        // Never hardcode in source
    }()
    
    static let environment: PlaidEnvironment = {
        #if DEBUG
        return .sandbox
        #else
        return .production
        #endif
    }()
}
```

## Testing Strategy

### Unit Tests
- PlaidService API calls (with mocked responses)
- PlaidTokenManager Keychain operations
- Transaction mapping logic
- Duplicate detection

### Integration Tests
- Link flow end-to-end
- Token exchange and storage
- Transaction sync with test accounts
- Error scenarios

### UI Tests
- Link UI interaction
- Account creation flow
- Sync button and progress
- Error message display

## Future Enhancements

1. **Auto-Sync**: Background sync when app launches
2. **Webhooks**: Real-time transaction updates (requires server)
3. **Multi-Institution**: Support multiple Plaid connections
4. **Category Enhancement**: Use Plaid's category suggestions
5. **Merchant Recognition**: Better merchant name extraction
6. **Balance Sync**: Real-time balance updates
7. **Account Aggregation**: View all accounts in one place

## Resolved Decisions

### Multi-Account Handling
**Decision**: Each Plaid account becomes an individual `Account` record:
- Accounts appear in the existing account list (no special UI grouping)
- Accounts respect existing account type groups (as they do now)
- When syncing, sync all accounts from the same Plaid item together (efficient - one API call)
- Consistent with CMB pattern (each account is individual)

**Rationale**: Matches existing Account model and UI patterns, maintains consistency with CMB, and enables efficient syncing.

For detailed discussion of all architectural decisions, see `PlaidOpenQuestionsDiscussion.md`.

## Open Questions

1. **Account Metadata**: Should we extend `Account` model or create separate `PlaidAccountMetadata`? *(See discussion doc)*
2. **Sync Frequency**: How often should we sync? User-configurable? *(See discussion doc)*
3. **Token Refresh**: Automatic or manual? When to prompt user? *(See discussion doc)*
4. ~~**Multi-Account**: One Plaid item = multiple accounts. How to handle?~~ ✅ **RESOLVED**: See above
5. **Cost**: Plaid pricing model (per transaction or per account)? *(See discussion doc)*

## Conclusion

This architecture leverages the existing AccountFeatureProvider pattern, ensuring Plaid integration is as decoupled as CMB. The modular design allows for:
- Easy testing
- Clear separation of concerns
- Future extensibility
- Maintainable codebase

The phased approach minimizes risk and allows for iterative development and testing.

