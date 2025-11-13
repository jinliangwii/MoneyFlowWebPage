# Account Feature Provider Architecture

## Overview

This document describes the decoupled architecture for account-specific functionality, allowing different account types (like CMB) to provide their own features without coupling them to general app logic.

## Problem Statement

Previously, CMB (China Merchants Bank) account functionality was tightly coupled with:
- **PropertiesView**: Hard-coded CMB detection and import button display
- **AppState**: CMB-specific import method (`importCMBZipPDF`)
- **ImportService**: CMB-specific import logic mixed with general import logic
- **Views**: Direct imports of CMB-specific views

This made it difficult to:
1. Add new account types with their own features
2. Maintain separation of concerns
3. Test account-specific functionality independently
4. Evolve the architecture without breaking existing code

## Solution: Account Feature Provider Pattern

### Architecture Components

#### 1. AccountFeatureProvider Protocol
```swift
protocol AccountFeatureProvider: Identifiable {
    var id: String { get }
    var supportedInstitution: String { get }
    
    func supports(account: Account) -> Bool
    func makeActionButtons(for account: Account) -> AnyView?
    func makeImportView(for account: Account) -> AnyView?
}
```

**Purpose**: Base protocol for all account-specific features. Allows providers to:
- Define which accounts they support
- Provide custom UI elements (buttons, views)
- Extend account functionality without modifying general app code

#### 2. AccountImportProvider Protocol
```swift
protocol AccountImportProvider: AccountFeatureProvider {
    func performImport(
        for account: Account,
        with parameters: AccountImportParameters,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult
}
```

**Purpose**: Extends `AccountFeatureProvider` for accounts that support import functionality. Provides:
- Type-safe import parameter structure (`AccountImportParameters`)
- Standardized import interface
- Progress reporting capability

#### 3. AccountFeatureRegistry
```swift
struct AccountFeatureRegistry {
    static let shared = AccountFeatureRegistry()
    
    func featureProvider(for account: Account) -> (any AccountFeatureProvider)?
    func importProvider(for account: Account) -> (any AccountImportProvider)?
}
```

**Purpose**: Central registry for finding feature providers. Provides:
- Decoupling between account models and their providers
- Easy lookup by account instance
- Extensibility for new account types

#### 4. CMBAccountFeatureProvider
Implementation of both protocols for CMB accounts:
- Provides CMB-specific import button and view
- Handles CMB ZIP/PDF import logic
- Encapsulates all CMB-specific business logic

### Key Design Principles

1. **Separation of Concerns**: Account-specific logic is isolated in feature providers
2. **Protocol-Based**: Uses Swift protocols for type safety and extensibility
3. **Registry Pattern**: Central registry allows dynamic lookup without coupling
4. **Open/Closed Principle**: Open for extension (new account types), closed for modification (general app logic)

## Usage Examples

### Adding Account-Specific UI

**Before** (coupled):
```swift
// In PropertiesView
if account.institution.lowercased().contains("招商") {
    Button("导入招商银行流水") { showCMBImport = true }
}
```

**After** (decoupled):
```swift
// In PropertiesView
if let featureProvider = AccountFeatureRegistry.shared.featureProvider(for: account),
   let actionButtons = featureProvider.makeActionButtons(for: account) {
    actionButtons
}
```

### Performing Account-Specific Import

**Before** (coupled):
```swift
// In AppState
func importCMBZipPDF(zipURL: URL, password: String, ...) {
    // CMB-specific logic
}
```

**After** (decoupled):
```swift
// In AppState
func importAccountTransactions(for account: Account, with parameters: AccountImportParameters, ...) {
    guard let importProvider = AccountFeatureRegistry.shared.importProvider(for: account) else {
        throw AppError.importError("该账户类型不支持导入功能")
    }
    let result = try await importProvider.performImport(for: account, with: parameters, ...)
    // Handle result generically
}
```

### Adding a New Account Type

To add a new account type (e.g., ICBC):

1. **Create Import Parameters**:
```swift
struct ICBCImportParameters: AccountImportParameters {
    let csvURL: URL
    let accountNumber: String
}
```

2. **Implement Feature Provider**:
```swift
struct ICBCAccountFeatureProvider: AccountFeatureProvider, AccountImportProvider {
    let id = "icbc-feature-provider"
    let supportedInstitution = "中国工商银行"
    
    func supports(account: Account) -> Bool {
        account.institution == supportedInstitution
    }
    
    func makeActionButtons(for account: Account) -> AnyView? {
        AnyView(ICBCImportButton(account: account))
    }
    
    func makeImportView(for account: Account) -> AnyView? {
        AnyView(ICBCImportView(account: account))
    }
    
    func performImport(for account: Account, with parameters: AccountImportParameters, ...) async throws -> ImportResult {
        // ICBC-specific import logic
    }
}
```

3. **Register Provider**:
```swift
// In AccountFeatureRegistry.registerProviders()
register(ICBCAccountFeatureProvider())
```

That's it! No changes needed to:
- `PropertiesView`
- `AppState`
- `ImportService`
- General app logic

## File Structure

```
MoneyFlow/
├── Features/
│   └── AccountProviders/
│       ├── AccountProvider.swift          # Account creation providers (existing)
│       ├── AccountFeatureProvider.swift   # NEW: Feature provider protocols & registry
│       └── CMBAccountFeatureProvider.swift # NEW: CMB feature implementation
├── AppState.swift                         # UPDATED: Uses feature providers
├── Views_PropertiesView.swift             # UPDATED: Queries feature providers
├── Views_AddAccountView.swift            # UPDATED: Uses feature providers
├── Services/
│   └── Services.swift                     # UPDATED: CMB logic removed
└── Data/
    └── Protocols.swift                    # UPDATED: CMB import removed from Importing
```

## Benefits

1. **Extensibility**: Easy to add new account types without modifying existing code
2. **Maintainability**: Account-specific logic is isolated and easier to test
3. **Decoupling**: General app logic is independent of account-specific implementations
4. **Type Safety**: Protocol-based design provides compile-time guarantees
5. **Testability**: Feature providers can be tested independently

## Migration Notes

### Removed Methods
- `AppState.importCMBZipPDF()` → `AppState.importAccountTransactions()`
- `ImportService.importCMBZipPDF()` → Moved to `CMBImportService`
- `Importing.importCMBZipPDF()` → Removed from protocol

### New Dependencies
- Views now depend on `AccountFeatureRegistry` instead of direct CMB checks
- AppState uses `AccountImportParameters` protocol instead of CMB-specific types

### Backward Compatibility
- Existing CMB accounts continue to work
- No data migration required
- Existing CMB import functionality preserved

## Future Enhancements

1. **Feature Discovery**: Auto-discover feature providers at runtime
2. **Plugin System**: Load account providers dynamically
3. **Feature Categories**: Support for features beyond import (e.g., export, sync)
4. **Provider Priority**: Allow multiple providers per account with priority ordering
5. **Feature Composition**: Combine features from multiple providers

