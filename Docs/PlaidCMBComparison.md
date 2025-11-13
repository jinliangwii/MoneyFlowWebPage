# Plaid vs CMB Implementation Comparison

This document compares the CMB (China Merchants Bank) and Plaid implementations to illustrate how the AccountFeatureProvider pattern is consistently applied across different account types.

## Pattern Consistency

Both CMB and Plaid follow the **same architectural pattern**, ensuring consistency and maintainability.

## Side-by-Side Comparison

| Component | CMB Implementation | Plaid Implementation |
|-----------|---------------------|----------------------|
| **Feature Provider** | `CMBAccountFeatureProvider` | `PlaidAccountFeatureProvider` |
| **Location** | `Features/CMB/Provider/` | `Features/Plaid/Provider/` |
| **Account Provider** | `CMBAccountProvider` | `PlaidAccountProvider` |
| **Import Service** | `CMBImportService` | `PlaidTransactionSyncService` |
| **Import Parameters** | `CMBImportParameters` | `PlaidImportParameters` |
| **Repository** | `CMBRepository` | `PlaidRepository` |
| **Import View** | `CMBImportView` | `PlaidLinkView` |

## Implementation Details

### 1. Feature Provider Pattern

#### CMB
```swift
struct CMBAccountFeatureProvider: AccountFeatureProvider, AccountImportProvider {
    let id = "cmb-feature-provider"
    let supportedInstitution = "招商银行"
    
    func supports(account: Account) -> Bool {
        account.institution == supportedInstitution
    }
    
    func performImport(...) async throws -> ImportResult {
        let service = CMBImportService()
        return try await service.importCMBZipPDF(...)
    }
}
```

#### Plaid
```swift
struct PlaidAccountFeatureProvider: AccountFeatureProvider, AccountImportProvider {
    let id = "plaid-feature-provider"
    let supportedInstitution = "Plaid"
    
    func supports(account: Account) -> Bool {
        account.metadata?["source"] == "plaid"
    }
    
    func performImport(...) async throws -> ImportResult {
        let service = PlaidTransactionSyncService()
        return try await service.syncTransactions(...)
    }
}
```

**Key Similarity**: Both implement the same protocols and follow the same structure.

### 2. Import Service Pattern

#### CMB Import Service
```swift
struct CMBImportService {
    func importCMBZipPDF(
        zipURL: URL,
        password: String,
        accountId: UUID,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult {
        // Step 1: Extract ZIP
        // Step 2: Parse PDF
        // Step 3: Validate balance
        // Step 4: Detect duplicates
        // Step 5: Convert to Transactions
        // Step 6: Return ImportResult
    }
}
```

#### Plaid Sync Service
```swift
struct PlaidTransactionSyncService {
    func syncTransactions(
        for account: Account,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult {
        // Step 1: Get access token
        // Step 2: Fetch transactions from API
        // Step 3: Detect duplicates
        // Step 4: Map to Transactions
        // Step 5: Return ImportResult
    }
}
```

**Key Similarity**: Both services follow the same flow:
1. Data source access (ZIP/PDF vs API)
2. Duplicate detection
3. Domain model conversion
4. Return standardized `ImportResult`

### 3. Import Parameters

#### CMB
```swift
struct CMBImportParameters: AccountImportParameters {
    let zipURL: URL
    let password: String
}
```

#### Plaid
```swift
struct PlaidImportParameters: AccountImportParameters {
    let accessToken: String?  // Optional for sync (can be retrieved from storage)
    let startDate: Date?
    let endDate: Date?
}
```

**Key Similarity**: Both conform to `AccountImportParameters` protocol, enabling generic handling in `AppState.importAccountTransactions()`.

### 4. Repository Pattern

#### CMB Repository
```swift
protocol CMBRepositoryProtocol {
    func loadRawTransactions() async throws -> [RawCMBTransaction]
    func saveRawTransactions(_ transactions: [RawCMBTransaction]) async throws
    func loadImportBatches() async throws -> [ImportCMBTransactionBatch]
    func saveImportBatches(_ batches: [ImportCMBTransactionBatch]) async throws
    func loadAccountMetadata(accountId: UUID) async throws -> CMBAccountMetadata?
    func saveAccountMetadata(_ metadata: CMBAccountMetadata) async throws
}
```

#### Plaid Repository
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
}
```

**Key Similarity**: Both use repository pattern for data persistence, keeping provider-specific data separate from core app data.

### 5. Account Provider Pattern

#### CMB Account Provider
```swift
struct CMBAccountProvider: AccountProvider {
    let id = "cmb-account-provider"
    let accountType: AccountType = .deposit
    
    func createConfigurationView(draft: AccountConfigurationDraft) -> AnyView {
        AnyView(CMBAccountImportConfigurationView(draft: draft))
    }
    
    func createAccount(from draft: AccountConfigurationDraft) throws -> Account {
        // Create account from CMB-specific configuration
    }
}
```

#### Plaid Account Provider
```swift
struct PlaidAccountProvider: AccountProvider {
    let id = "plaid-account-provider"
    let accountType: AccountType = .deposit  // Can support multiple types
    
    func createConfigurationView(draft: AccountConfigurationDraft) -> AnyView {
        AnyView(PlaidLinkView(draft: draft))
    }
    
    func createAccount(from draft: AccountConfigurationDraft) throws -> Account {
        // Create account from Plaid Link response
    }
}
```

**Key Similarity**: Both implement `AccountProvider` protocol for account creation flow.

## Registration

### Provider Registry
Both providers are registered in the same place:

```swift
// In ProviderRegistry.swift
private mutating func registerProviders() {
    // CMB
    register(CMBAccountProvider(), for: .deposit)
    
    // Plaid
    register(PlaidAccountProvider(), for: .deposit)
    register(PlaidAccountProvider(), for: .creditCard)
    // ... other account types
}
```

### Feature Registry
Both feature providers are registered in the same place:

```swift
// In AccountFeatureRegistry.swift
private mutating func registerProviders() {
    // CMB
    register(CMBAccountFeatureProvider())
    
    // Plaid
    register(PlaidAccountFeatureProvider())
}
```

## Integration Points

### AppState Integration

Both use the same generic method in `AppState`:

```swift
func importAccountTransactions(
    for account: Account,
    with parameters: AccountImportParameters,
    onProgress: @escaping (ImportProgress) -> Void
) async throws {
    // Generic implementation - works for both CMB and Plaid
    guard let importProvider = AccountFeatureRegistry.shared.importProvider(for: account) else {
        throw AppError.importError("该账户类型不支持导入功能")
    }
    
    let result = try await importProvider.performImport(
        for: account,
        with: parameters,
        onProgress: onProgress
    )
    
    // Generic result handling
    transactions.append(contentsOf: result.transactions)
    // ...
}
```

**Key Benefit**: No provider-specific code in `AppState` - works for both CMB and Plaid (and future providers).

### View Integration

Both use the same pattern in views:

```swift
// In PropertiesView or AccountDetailView
if let featureProvider = AccountFeatureRegistry.shared.featureProvider(for: account),
   let actionButtons = featureProvider.makeActionButtons(for: account) {
    actionButtons  // Shows CMB or Plaid buttons automatically
}
```

## Key Differences

### Data Source
- **CMB**: File-based (ZIP/PDF upload)
- **Plaid**: API-based (real-time fetch)

### Authentication
- **CMB**: Password-protected ZIP file
- **Plaid**: OAuth-style Link flow with access tokens

### Sync Model
- **CMB**: Manual, one-time import
- **Plaid**: Automatic, periodic sync capability

### Token Management
- **CMB**: Not needed
- **Plaid**: Requires secure token storage and refresh

### Account Discovery
- **CMB**: Manual entry
- **Plaid**: Automatic via Link

## Benefits of Consistent Pattern

✅ **Easy to Add New Providers**: Just follow the same pattern
✅ **Consistent User Experience**: Similar UI/UX across providers
✅ **Maintainable Codebase**: Same structure, easy to understand
✅ **Testable**: Same testing patterns apply
✅ **Scalable**: Can add providers without modifying core code

## Example: Adding a Third Provider

Following the same pattern, adding a new provider (e.g., Yodlee) would be:

```swift
// Features/Yodlee/Provider/YodleeAccountFeatureProvider.swift
struct YodleeAccountFeatureProvider: AccountFeatureProvider, AccountImportProvider {
    let id = "yodlee-feature-provider"
    let supportedInstitution = "Yodlee"
    
    // Same structure as CMB and Plaid
}

// Register in AccountFeatureRegistry
register(YodleeAccountFeatureProvider())
```

**That's it!** No changes needed to:
- `AppState`
- Core views
- Repository protocols
- Import flow

## Conclusion

The consistent pattern ensures that:
1. **CMB** and **Plaid** implementations are nearly identical in structure
2. Adding **future providers** follows the same pattern
3. **Core app logic** remains provider-agnostic
4. **Testing** follows the same patterns
5. **Maintenance** is easier with consistent code structure

This demonstrates the power of the AccountFeatureProvider pattern: it scales to support any number of account types while keeping the core app clean and maintainable.

