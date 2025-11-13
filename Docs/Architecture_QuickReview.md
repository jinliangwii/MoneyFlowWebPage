# Architecture Quick Review: Layer Independence Pattern

## Core Principle

**DataSource layer focuses on extracting original source data into structured data. Integration layer has institution knowledge via Type Registry (for mapping), but business logic remains account type-based.**

- **DataSource**: Extract & Parse - Parses original source files/APIs into structured data (raw account info, raw transaction data)
- **Integration**: Bridge & Orchestration - Has institution knowledge via Type Registry (DataSource → RawTransactionType mapping), converts to account type metadata, creates capabilities
- **Providers**: Account TYPE-specific (.creditCard, .loan, .deposit) - NOT bank-specific
- **UI**: Dynamic, renders based on account type, source file type, and metadata from DataSource

The data extracted from DataSource layer determines which features/capabilities are used in upper layers.

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    UI Layer                                 │
│  - Dynamic UI rendering                                     │
│  - Renders based on account type (.creditCard, .loan, etc.)│
│  - Renders based on source type (PDF, CSV, Excel, API, etc.)│
│  - Queries providers for account type-specific features    │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌─────────────────────────────────────────────────────────────┐
│                 Providers Layer                              │
│  - Account TYPE-based providers (.creditCard, .loan, etc.) │
│  - NO bank/institution-specific logic                      │
│  - Coordinates UI and Integration                           │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌─────────────────────────────────────────────────────────────┐
│              Integration Layer                               │
│  - Bridge: Converts institution-specific → account type     │
│  - Has institution knowledge via Type Registry              │
│    (Maps DataSource class → RawTransactionType)              │
│  - Knows: What capabilities are required                    │
│  - Knows: How to map/convert DataSource data               │
│  - Creates generic capabilities based on account type       │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │
┌─────────────────────────────────────────────────────────────┐
│              DataSource Layer                                │
│  ⚠️ FOCUS: Extract original source data into structured data│
│  - Parses original source files/APIs                       │
│  - Extracts raw account info (AccountMetadata)              │
│  - Extracts raw transaction data (RawTransactionProtocol)   │
│  - Bank/institution-specific parsers                        │
│    (CMBCreditCardPDFDataSource, WeChatExcelDataSource, etc.)│
│  - Provides structured data, NOT business logic            │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer Responsibilities

| Layer | Scope | Key Responsibilities | Should NOT Have |
|-------|-------|---------------------|-----------------|
| **DataSource** | Extract & Parse | Parse original source files/APIs, extract raw account info and raw transaction data into structured formats | Business logic, account type logic, conversion logic |
| **Integration** | Bridge & Orchestration | Has institution knowledge via Type Registry (DataSource → RawTransactionType mapping), converts institution→account type metadata, creates capabilities, maps raw→domain transactions | Business logic for account types, UI logic |
| **Providers** | Account TYPE-specific | Provide account type-specific UI, coordinate import flow | Bank/institution-specific logic, DataSource parsing knowledge |
| **UI** | Dynamic rendering | Render based on account type + source type | Bank/institution-specific logic, hard-coded account types |

---

## Data Flow Pattern

### 1. DataSource Extracts Raw Data
```swift
// DataSource provides raw AccountMetadata (generic structure)
let accountMetadata = AccountMetadata(
    accountTypeString: "creditCard",
    institution: "招商银行",
    accountNumber: "...",
    currencyCode: "CNY",
    balance: ...,
    metadata: ["statementPeriod": "...", "accountName": "..."]  // Generic dictionary
)
```

### 2. Integration Processes Data & Creates Capabilities
```swift
// Integration layer defines metadata protocols (e.g., CreditCardAccountMetadata)
// Integration processes raw DataSource data into account type-specific metadata
let accountType = AccountTypeMapper.map(accountMetadata.accountTypeString)
let accountTypeMetadata = Integration.processAccountMetadata(
    rawMetadata: accountMetadata  // From DataSource
)  // Returns: CreditCardAccountMetadata (defined in Integration layer)

// Integration creates capabilities based on account TYPE
let capabilities = CapabilityFactory.createCapabilities(
    forAccountType: accountType,
    using: accountTypeMetadata  // Processed metadata from Integration
)
```

### 2b. Integration Provides Processed Metadata to Provider
```swift
// Integration layer provides processed metadata
struct ProcessedAccountMetadata {
    let accountType: AccountType
    let accountTypeMetadata: AccountTypeMetadata  // CreditCardAccountMetadata, etc.
    // ... other processed fields
}

// Provider extracts/uses metadata from Integration (no direct DataSource access)
let processedMetadata = Integration.getProcessedMetadata(for: accountMetadata)
```

### 3. Provider Coordinates
```swift
// Provider uses Integration orchestrator
let orchestrator = IntegrationOrchestrator(...)
return try await orchestrator.executeImport(
    dataSource: dataSource,
    accountMetadata: accountMetadata,
    ...
)
```

### 4. UI Renders Dynamically
```swift
// UI renders based on account type + source type
switch (account.type, sourceType) {
case (.creditCard, .pdf):
    CreditCardPDFImportView(...)
case (.paymentGate, .excel):
    PaymentGateExcelImportView(...)
}
```

---

## Questions to Answer (Based on Real Code)

### ✅ 1. How Should DataSource Provide Capability Metadata? (RESOLVED)

**Resolution:**
- **Integration layer defines** account type-specific metadata protocols (e.g., `CreditCardAccountMetadata`)
- **DataSource provides raw data** in generic `AccountMetadata` structure (with generic `metadata` dictionary)
- **Integration processes** raw DataSource data into account type-specific metadata structures
- **Provider layer extracts/uses** processed metadata from Integration (no direct DataSource access)

**Example:**
```swift
// Integration layer defines interface
protocol CreditCardAccountMetadata {
    var accountNumber: String { get }
    var statementBalance: Decimal? { get }
    var statementPeriod: String? { get }
}

// DataSource provides raw metadata (generic structure)
AccountMetadata(
    accountTypeString: "creditCard",
    institution: "招商银行",
    metadata: ["accountNumber": "...", "statementBalance": "...", "statementPeriod": "..."]
)

// Integration processes raw data into account type-specific metadata
let accountTypeMetadata = Integration.processAccountMetadata(
    rawMetadata: accountMetadata
)  // Returns: CreditCardAccountMetadata (defined in Integration layer)
```

---

### ✅ 2. How Should Integration Layer Select Capabilities? (RESOLVED)

**Resolution:**
Integration layer should know:
1. **What capabilities are required** - Based on account type metadata from DataSource
2. **How to map/convert DataSource data** - Raw transactions → Domain transactions
3. **These are the ONLY two DataSource-specific things** Integration layer should know
4. **Carefully pick/develop capabilities** via DataSource-specific data in this layer, but remain generic (account type-based, not bank-specific)

**Key Points:**
- Integration layer **creates** capabilities (doesn't receive them)
- Capabilities are created based on **account type** (not bank/institution)
- Integration uses **DataSource-specific data** (from `AccountMetadata.accountTypeMetadata`) to configure capabilities
- Integration knows **mapping rules** for converting raw transactions to domain transactions
- All other logic should be generic and account type-based

---

## Repository Architecture

### Two-Tier Repository System

The architecture uses **two distinct repository layers** with different responsibilities:

#### 1. DataSource Layer Repositories (Institution-Specific)
**Purpose**: Store raw, institution-specific data

- **Examples**: `CMBCreditCardRepository`, `WeChatRepository`, `Alipay3rdPartyTxRepository`
- **Stores**:
  - Raw transactions (e.g., `RawCMBCreditCardTransaction`, `RawWeChatTransaction`)
  - Institution-specific account metadata (e.g., `CMBCreditCardAccountMetadata`)
  - Import batches
- **Why Institution-Specific?**
  - Raw transactions are in institution-specific format (CMB PDF structure, WeChat Excel format, etc.)
  - Account metadata may have institution-specific fields
  - These repositories are part of the DataSource layer (the ONLY layer with institution-specific logic)

#### 2. Domain Layer Repository (Account Type-Based)
**Purpose**: Store converted, domain transactions

- **Example**: `FileTransactionRepository` (implements `TransactionRepositoryProtocol`)
- **Stores**:
  - Domain `Transaction` models (account type-based, generic format)
- **Why Generic?**
  - Domain transactions are already converted to a generic format
  - All account types use the same `Transaction` model
  - Used by UI/Provider layer for presentation (account type-based queries)

#### Data Flow
```
DataSource Layer:
  Raw Transactions (Institution-Specific)
  ↓
  [CMBCreditCardRepository, WeChatRepository, etc.]
  
Integration Layer:
  Convert Raw → Domain
  ↓
  
Domain Layer:
  Domain Transactions (Account Type-Based)
  ↓
  [FileTransactionRepository]
  
Provider/UI Layer:
  Query by account type
  ↓
  [Presentation]
```

**Key Insight**: DataSource repositories correctly remain institution-specific because they store raw, institution-specific data. Domain repository is already generic and account type-based.

---

## Architecture Analysis: Pros, Cons & Feasibility

### ✅ Pros

1. **Clear Separation of Concerns**
   - Each layer has a well-defined responsibility
   - Bank-specific logic isolated to DataSource layer
   - Upper layers are generic and reusable

2. **Extensibility**
   - Adding new banks: Only need new DataSource implementation
   - Adding new account types: Only need new Provider + Integration capabilities
   - No changes needed to existing code

3. **Type Safety**
   - Account type-specific metadata protocols provide compile-time safety
   - No more generic dictionaries with runtime errors

4. **Testability**
   - Each layer can be tested independently
   - Easy to mock DataSource for testing Integration
   - Easy to mock Integration for testing Providers

5. **Maintainability**
   - Changes to one bank don't affect others
   - Changes to account type logic are isolated
   - Clear ownership of code

6. **Reusability**
   - Generic capabilities (BalanceCalculator, DuplicateDetector) reusable across accounts
   - Integration orchestrator works for all account types
   - UI components reusable for same account type + source type

---

### ⚠️ Cons & Challenges

1. **Complexity of Type System** ✅ RESOLVED
   - Associated types in `AccountCapabilities` (`RawTransactionType`)
   - ✅ **Type Registry solution** - No type erasure needed, type information preserved
   - Type casting simplified via Type Registry lookup

2. **Migration Effort**
   - Need to refactor all existing Providers (bank-specific → account type-specific)
   - Need to refactor all existing Capabilities (bank-specific → account type-specific)
   - Need to refactor all existing UI (bank-specific → account type + source type)
   - ✅ **Repositories already correct** - No repository refactoring needed
   - Need to create Type Registry and update Integration layer
   - Need to create metadata processing in Integration layer

3. **Repository Architecture** ✅ CLARIFIED
   - **DataSource Layer Repositories** (e.g., `CMBCreditCardRepository`, `WeChatRepository`):
     - Store **raw transactions** (institution-specific format: `RawCMBCreditCardTransaction`, `RawWeChatTransaction`)
     - Store institution-specific account metadata (e.g., `CMBCreditCardAccountMetadata`)
     - Store `ImportBatch` data
     - **SHOULD REMAIN institution-specific** because raw data is institution-specific
   - **Domain Layer Repository** (`FileTransactionRepository`):
     - Stores **domain transactions** (`Transaction` model, account type-based)
     - **Already generic and account type-based!** ✅
     - Used by UI/Provider layer for presentation
   - **Flow**: Raw transactions (institution-specific) → Convert → Domain transactions (account type-based) → Save to generic repository

4. **Raw Transaction Type Handling** (Real Case Scenario)

**Current Problem:**
- IntegrationOrchestrator receives capabilities with hard-coded `RawTransactionType`:
  ```swift
  // Provider creates capabilities with specific raw transaction type
  struct CMBCreditCardCapabilities: AccountCapabilities {
      typealias RawTransactionType = RawCMBCreditCardTransaction  // Hard-coded!
  }
  
  struct WeChatCapabilities: AccountCapabilities {
      typealias RawTransactionType = RawWeChatTransaction  // Hard-coded!
  }
  ```
- IntegrationOrchestrator uses `Capabilities.RawTransactionType` to type-cast:
  ```swift
  // Line 46-49 in IntegrationOrchestrator.swift
  let rawTransactions = try await convertToTyped(
      rawTransactionsProtocol,  // [any RawTransactionProtocol] from DataSource
      as: Capabilities.RawTransactionType.self  // Needs to know the type!
  )
  ```

**Real Cases:**
1. **CMB Credit Card PDF** (`CMBCreditCardPDFDataSource`)
   - Returns: `[RawCMBCreditCardTransaction]` (via `extractTransactions()`)
   - Integration needs: `RawTransactionType = RawCMBCreditCardTransaction`
   - Converter: `CMBCreditCardTransactionConverter`

2. **WeChat Excel** (`WeChatExcelDataSource`)
   - Returns: `[RawWeChatTransaction]` (via `extractTransactions()`)
   - Integration needs: `RawTransactionType = RawWeChatTransaction`
   - Converter: `WeChatTransactionConverter`

3. **Alipay CSV** (`AlipayCSVDataSource`)
   - Returns: `[RawAlipay3rdPartyTxTransaction]` (via `extractTransactions()`)
   - Integration needs: `RawTransactionType = RawAlipay3rdPartyTxTransaction`
   - Converter: `AlipayTransactionConverter`

**The Solution: Type Registry** ✅
- **Do NOT erase type** - Keep type information available
- **Type Registry** - Map DataSource class → RawTransactionType
- Integration can look up the correct `RawTransactionType` based on DataSource instance

**Implementation Approach:**
```swift
// Type registry mapping DataSource type to RawTransactionType
struct RawTransactionTypeRegistry {
    static let shared = RawTransactionTypeRegistry()
    
    private var typeMap: [String: any RawTransactionProtocol.Type] = [:]
    
    init() {
        // Register DataSource types
        register(CMBCreditCardPDFDataSource.self, rawType: RawCMBCreditCardTransaction.self)
        register(WeChatExcelDataSource.self, rawType: RawWeChatTransaction.self)
        register(AlipayCSVDataSource.self, rawType: RawAlipay3rdPartyTxTransaction.self)
        register(CMBPersonalCheckingPDFDataSource.self, rawType: RawCMBPersonalCheckingTransaction.self)
        // ... more registrations
    }
    
    func register<DS: DataSourceProtocol, RT: RawTransactionProtocol>(
        _ dataSourceType: DS.Type,
        rawType: RT.Type
    ) {
        let key = String(describing: dataSourceType)
        typeMap[key] = rawType
    }
    
    func getRawTransactionType(for dataSource: any DataSourceProtocol) -> (any RawTransactionProtocol.Type)? {
        let key = String(describing: type(of: dataSource))
        return typeMap[key]
    }
}

// Integration uses registry to determine RawTransactionType
class IntegrationOrchestrator {
    func executeImport(
        dataSource: any DataSourceProtocol,
        accountMetadata: AccountMetadata,
        ...
    ) async throws -> ImportResult {
        // Look up raw transaction type from registry
        guard let rawTransactionType = RawTransactionTypeRegistry.shared.getRawTransactionType(for: dataSource) else {
            throw AppError.importError("Unknown data source type")
        }
        
        // Now Integration can create capabilities with the correct RawTransactionType
        let capabilities = CapabilityFactory.createCapabilities(
            forAccountType: accountType,
            rawTransactionType: rawTransactionType,
            using: accountTypeMetadata
        )
        
        // Extract transactions (still type-erased, but we know the type from registry)
        let rawTransactionsProtocol = try await dataSource.extractTransactions(...)
        
        // Convert to typed using the known type
        let rawTransactions = try await convertToTyped(
            rawTransactionsProtocol,
            as: rawTransactionType
        )
        
        // Continue with typed transactions...
    }
}
```

**Benefits:**
- ✅ No type erasure - Type information preserved via registry
- ✅ Integration can create capabilities with correct `RawTransactionType`
- ✅ Centralized mapping - Easy to add new DataSources
- ✅ Type-safe - Compile-time checking of type relationships

5. **Converter Architecture** ✅ CLARIFIED
   - **Converters are DataSource-specific** (e.g., `CMBCreditCardTransactionConverter`, `WeChatTransactionConverter`)
   - **Why DataSource-specific?**
     - Conversion rules depend on DataSource format (CMB PDF structure vs WeChat Excel format vs Plaid API format)
     - Each DataSource has different raw transaction structures and field mappings
     - Example: WeChat's "零钱提现" handling vs CMB PDF's balance parsing
   - **No consolidation needed** - Converters correctly remain DataSource-specific
   - **Integration layer** uses these converters via capabilities (created based on account type + DataSource)

6. **UI Organization**
   - Currently: `UI/Import/AccountSpecific/CMBCreditCard/` (bank-specific)
   - Should be: `UI/Import/CreditCard/PDF/` or `UI/Import/CreditCard/API/` (account type + source type)
   - Need to reorganize UI structure

7. **Metadata Protocol Definition Location** ✅ CLARIFIED
   - **Metadata protocols defined in Integration layer** (e.g., `CreditCardAccountMetadata`)
   - **Integration layer processes DataSource data** into these metadata structures
   - **Provider layer extracts/uses** metadata from Integration (no direct DataSource access)
   - **No cross-layer communication** - Clear boundaries:
     - DataSource → Integration: Raw data (AccountMetadata with generic fields)
     - Integration → Provider: Processed metadata (account type-specific metadata protocols)
   - **Flow**:
     ```
     DataSource: Provides raw AccountMetadata (generic structure)
         ↓
     Integration: Processes raw data → Account type-specific metadata (CreditCardAccountMetadata, etc.)
         ↓
     Provider: Extracts/uses processed metadata from Integration
     ```

---

### ❓ Will It Work?

**YES, with careful implementation:**

**Strengths:**
- The pattern is sound and follows SOLID principles
- Clear boundaries between layers
- Type-safe metadata approach is better than generic dictionaries

**Critical Success Factors:**

1. **Raw Transaction Type Resolution**
   - **Solution**: DataSource should provide raw transaction type information
   - Could be in `AccountMetadata` or `SourceInfo`
   - Integration uses this to determine which converter to use

2. **Repository Architecture** ✅ Already Correct
   - **DataSource repositories**: Remain institution-specific (for raw transactions)
   - **Domain repository**: Already generic and account type-based (`FileTransactionRepository`)
   - **No consolidation needed** - Architecture is correct as-is

3. **Converter Architecture** ✅ Already Correct
   - **Converters are DataSource-specific** (correct as-is)
   - Each DataSource has its own converter (e.g., `CMBCreditCardTransactionConverter`, `WeChatTransactionConverter`)
   - Conversion rules are DataSource-specific (CMB PDF format vs WeChat Excel vs Plaid API)
   - Integration layer uses converters via capabilities (selected based on DataSource)

4. **Metadata Protocol Location** ✅ CLARIFIED
   - **Solution**: Define metadata protocols in Integration layer
   - **Integration processes DataSource data** into these metadata structures
   - **Provider layer extracts/uses** metadata from Integration (no direct DataSource access)
   - **No cross-layer communication** - Clear boundaries maintained

---

### 🔍 What's Not Covered Yet?

1. **Repository Selection** ✅ RESOLVED
   - **DataSource repositories**: Selected by DataSource (institution-specific, for raw transactions)
   - **Domain repository**: Generic `FileTransactionRepository` (already account type-based)
   - **No consolidation needed** - DataSource repositories correctly remain institution-specific
   - **Solution**: Integration accesses DataSource repositories via capabilities
     - Capabilities include repository access when needed (e.g., for metadata verification)
     - Integration doesn't directly access repositories - uses capabilities
     - Clear separation: DataSource manages its own repository, Integration uses capabilities

2. **Raw Transaction Type Determination** ✅ SOLUTION: Type Registry
   - **Solution**: Use type registry mapping DataSource class → RawTransactionType
   - **Do NOT erase type** - Keep type information available via registry
   - **Implementation**: `RawTransactionTypeRegistry` maps DataSource types to their RawTransactionType
   - **Real Cases**:
     - `CMBCreditCardPDFDataSource` → `RawCMBCreditCardTransaction`
     - `WeChatExcelDataSource` → `RawWeChatTransaction`
     - `AlipayCSVDataSource` → `RawAlipay3rdPartyTxTransaction`
   - **Benefits**:
     - ✅ No type erasure - Type information preserved
     - ✅ Integration can create capabilities with correct `RawTransactionType`
     - ✅ Centralized mapping - Easy to add new DataSources
     - ✅ Type-safe - Compile-time checking

3. **Converter Selection** ✅ RESOLVED
   - ✅ **Converters are DataSource-specific** (correct as-is)
   - **Solution**: Integration selects converter via Type Registry
     - Type Registry maps DataSource → RawTransactionType
     - Integration creates capabilities with correct converter based on RawTransactionType
     - Converter is selected when creating capabilities in `CapabilityFactory`
     - Flow: DataSource → Type Registry → RawTransactionType → CapabilityFactory → Converter

4. **UI Reorganization** ✅ SUGGESTION
   - **Organization**: By account type + source type (e.g., `UI/Import/CreditCard/PDF/`, `UI/Import/PaymentGate/Excel/`)
   - **Account type-specific UI**: Provider layer provides account type-specific components
     - Provider queries Integration for processed metadata (account type-specific)
     - UI renders based on account type (e.g., `.creditCard` → CreditCardImportView)
   - **Source type-specific UI**: UI adapts based on `DataSourceType` from DataSource
     - PDF sources → PDF-specific UI (file picker, password input)
     - API sources → API-specific UI (OAuth flow, token management)
   - **Dynamic rendering**: `switch (accountType, sourceType)` in Provider layer
   - **No bank-specific UI**: All UI is account type + source type based
   - **Example structure**:
     ```
     UI/Import/
       CreditCard/
         PDF/
           CreditCardPDFImportView.swift
         API/
           CreditCardAPIImportView.swift
       PaymentGate/
         Excel/
           PaymentGateExcelImportView.swift
     ```

5. **Metadata Protocol Location** ✅ CLARIFIED
   - **Solution**: Define metadata protocols in Integration layer
   - **Integration processes DataSource data** into these metadata structures
   - **Provider layer extracts/uses** metadata from Integration (no direct DataSource access)
   - **No cross-layer communication** - Clear boundaries:
     - DataSource → Integration: Raw data (AccountMetadata with generic fields)
     - Integration → Provider: Processed metadata (account type-specific metadata protocols)

6. **Error Handling** ✅ SUGGESTION
   - **Error flow**: DataSource → Integration → Provider → UI (no cross-layer error handling)
   - **DataSource errors**: Institution-specific errors (parsing failures, format issues)
     - DataSource throws `DataSourceError` with institution-specific context
   - **Integration errors**: Generic errors with account type context
     - Integration converts DataSource errors to generic `ImportError`
     - Integration adds account type context (e.g., "Credit card import failed: ...")
   - **Provider errors**: Account type-specific error handling
     - Provider catches Integration errors and provides account type-specific error messages
     - Provider may retry or suggest account type-specific solutions
   - **UI errors**: User-friendly messages based on account type + source type
     - UI receives errors from Provider (already account type-specific)
     - UI formats errors for display (e.g., "Credit card PDF import failed: invalid format")
   - **Error types**:
     ```swift
     // DataSource layer
     enum DataSourceError: Error {
         case invalidSource(String)  // Institution-specific
         case parsingFailed(String)   // Institution-specific
     }
     
     // Integration layer
     enum ImportError: Error {
         case dataSourceError(DataSourceError)  // Wraps DataSource error
         case capabilityError(String)            // Generic capability error
         case conversionError(String)            // Generic conversion error
     }
     
     // Provider layer
     // Uses ImportError, adds account type context
     ```

7. **Testing Strategy** ✅ SUGGESTION
   - **Test each layer independently** - No cross-layer dependencies in tests
   - **DataSource layer tests**:
     - Test parsing logic for each institution-specific format
     - Mock file system/API responses
     - Verify raw data extraction (AccountMetadata, RawTransactionProtocol)
   - **Integration layer tests**:
     - Mock DataSource (returns generic AccountMetadata, RawTransactionProtocol)
     - Test metadata processing (raw → account type-specific)
     - Test Type Registry lookup
     - Test CapabilityFactory creation (with mocked Type Registry)
     - Test capability orchestration (with mocked capabilities)
   - **Provider layer tests**:
     - Mock Integration layer (returns processed metadata, capabilities)
     - Test account type-specific logic (no bank-specific logic)
     - Test UI component selection based on account type + source type
   - **Integration tests**:
     - Test full flow: DataSource → Integration → Provider
     - Use real DataSource implementations with test data
     - Verify end-to-end import flow
   - **Test structure**:
     ```
     Tests/
       DataSourceTests/
         CMBCreditCardPDFDataSourceTests.swift
         WeChatExcelDataSourceTests.swift
       IntegrationTests/
         MetadataProcessingTests.swift
         TypeRegistryTests.swift
         CapabilityFactoryTests.swift
       ProviderTests/
         CreditCardProviderTests.swift
         PaymentGateProviderTests.swift
       IntegrationTests/
         FullImportFlowTests.swift
     ```

8. **Migration Path** ✅ SUGGESTION
   - **Incremental migration** - One account type at a time
   - **Phase 1: Create new infrastructure** (no breaking changes)
     1. Create Type Registry in Integration layer
     2. Define metadata protocols in Integration layer
     3. Create metadata processing in Integration layer
     4. Create CapabilityFactory in Integration layer
   - **Phase 2: Migrate one account type** (e.g., Credit Card)
     1. Update IntegrationOrchestrator to use Type Registry
     2. Create account type-specific Provider (CreditCardProvider)
     3. Migrate one bank's Provider (e.g., CMB → CreditCardProvider)
     4. Test with one account type
   - **Phase 3: Migrate remaining account types**
     1. Migrate other banks for same account type
     2. Migrate other account types (PaymentGate, Loan, etc.)
   - **Phase 4: Cleanup**
     1. Remove old bank-specific Providers
     2. Remove old bank-specific Capabilities
     3. Reorganize UI structure
   - **Backward compatibility**:
     - Keep old Provider implementations during migration
     - Use feature flags to switch between old/new implementations
     - Gradually migrate users to new architecture
   - **Testing strategy**:
     - Run both old and new implementations in parallel
     - Compare results to ensure correctness
     - Gradually switch traffic to new implementation

9. **Performance Considerations**
   - ✅ **Type erasure no longer an issue** - Using Type Registry preserves type information
   - **Capability caching**: Consider caching capabilities per (AccountType, DataSourceType) combination
   - **Repository lookups**: Already optimized via direct repository access in capabilities
   - **Type Registry**: O(1) lookup by DataSource type name (String-based key)

10. **Multi-Account DataSources** ✅ SUGGESTION
   - **DataSource responsibility**: Extract all accounts, return array of AccountMetadata
     - DataSource doesn't know which account to import
     - DataSource provides all available accounts from source
   - **Integration responsibility**: Process all accounts, provide account selection
     - Integration processes all AccountMetadata from DataSource
     - Integration provides processed account list to Provider
     - Integration doesn't select accounts - just processes them
   - **Provider responsibility**: Account selection UI (account type-specific)
     - Provider receives list of processed accounts from Integration
     - Provider shows account type-specific selection UI
     - Provider coordinates with Integration for selected account
   - **UI responsibility**: Render account selection based on account type
     - UI queries Provider for account selection component
     - UI renders based on account type (e.g., CreditCardAccountSelectionView)
     - No bank-specific UI - all selection UI is account type-based
   - **Flow**:
     ```
     DataSource.extractAccounts() → [AccountMetadata]  // All accounts
         ↓
     Integration.processAccounts([AccountMetadata]) → [ProcessedAccountMetadata]  // Process all
         ↓
     Provider.showAccountSelection([ProcessedAccountMetadata]) → Selected account
         ↓
     Integration.createCapabilities(for: selectedAccount) → Capabilities
         ↓
     IntegrationOrchestrator.executeImport(...) → Import result
     ```
   - **Example (Plaid)**:
     - PlaidDataSource extracts all Plaid accounts → [AccountMetadata]
     - Integration processes all → [ProcessedAccountMetadata] (with account types)
     - Provider shows account selection UI (grouped by account type)
     - User selects account → Integration creates capabilities → Import proceeds

---

## Next Steps

1. **Define account type metadata protocols in Integration layer** - Integration defines interfaces (e.g., `CreditCardAccountMetadata`)
2. **Create metadata processing in Integration** - Process raw DataSource `AccountMetadata` into account type-specific metadata
3. **Create Type Registry** - Map DataSource class → RawTransactionType
4. **Create CapabilityFactory in Integration** - Create capabilities based on account type + RawTransactionType from registry
5. **Refactor IntegrationOrchestrator** - Create capabilities instead of receiving them, use Type Registry
6. **Refactor Providers** - Remove bank-specific logic, use account type only, extract metadata from Integration
7. **Repository Architecture** ✅ Already correct:
   - DataSource repositories remain institution-specific (for raw transactions)
   - Domain repository is already generic (for domain transactions)
8. **Converter Architecture** ✅ Already correct:
   - Converters remain DataSource-specific (for DataSource-specific conversion rules)
   - Integration uses converters via capabilities (selected via Type Registry)
9. **Reorganize UI** - Bank-specific → Account type + Source type
10. **Test with one account type** - Validate the pattern works
11. **Migrate other account types** - Apply pattern to all accounts
