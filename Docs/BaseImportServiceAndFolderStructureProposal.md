# Base Import Service & Folder Structure Proposal

## Problem Statement

Currently, each account import service (Alipay, WeChat, CMB, Plaid) reimplements similar logic, leading to:

1. **Code Duplication**: ~60-70% of import service code is duplicated across services
2. **Inconsistent Implementations**: Some use O(1) lookups, some use O(n) searches
3. **Maintenance Burden**: Fixing bugs or adding features requires changes in multiple places
4. **Higher Error Risk**: Adding new account types increases chance of bugs
5. **Poor Organization**: `Features/` folder is getting crowded with mixed concerns

### Current Issues

**Code Duplication**:
- Raw Transaction Duplicate Detection: Loading existing raw transactions, building fingerprint sets, checking duplicates
- Transaction Duplicate Detection: Checking against existing Transaction objects
- Sequence Number Management: Building maps of existing sequence numbers, assigning new sequence numbers
- Progress Reporting: Reporting progress during duplicate detection
- Data Loading: Loading existing raw transactions and Transaction objects
- Data Saving: Saving raw transactions and import batches

**Folder Structure Issues**:
- `Features/` folder contains mixed concerns:
  - Account-specific implementations (Alipay, CMB, WeChat, Plaid)
  - UI features (Dashboard, Properties, Settings, Sidebar, Summary)
  - Account providers (AccountProviders/)
  - Debug features
- Hard to find related files across different account types
- No clear separation between business logic and UI

## Part 1: Common Patterns Analysis

### Identified Common Patterns Across Import Services

#### 1. **Account Metadata Verification & Management**
**Pattern**: All services verify account metadata (email, nickname, account number) and save it if it doesn't exist.

**Current Implementation**:
- Alipay: Verifies email from CSV
- WeChat: Verifies nickname from Excel
- CMB: Verifies account number from PDF
- Plaid: Manages Plaid-specific metadata

**Common Logic**:
```swift
// Pattern repeated in all services
if let metadataValue = parsedInfo.metadataValue {
    let storedMetadata = try await repository.loadAccountMetadata(accountId: accountId)
    
    if let storedValue = storedMetadata?.metadataValue {
        if storedValue != metadataValue {
            throw AppError.importError("验证失败...")
        }
    } else {
        // Save metadata from this import
        let metadata = AccountMetadata(accountId: accountId, value: metadataValue)
        try await repository.saveAccountMetadata(metadata)
    }
}
```

#### 2. **Transaction Filtering**
**Pattern**: All services filter transactions based on various criteria.

**Current Implementation**:
- Alipay/WeChat: Filter by status ("还款失败"/"支付失败") and amount == 0
- CMB: Filter by currency
- All: Filter invalid transactions (missing date, amount, etc.)

**Common Logic**:
```swift
// Pattern repeated with slight variations
let rawTransactions = allRawTransactions.filter { rawTx in
    // Status-based filtering (account-specific)
    if shouldExcludeByStatus(rawTx) { return false }
    
    // Amount-based filtering
    if let amount = rawTx.parsedAmount, amount == 0 { return false }
    
    // Currency-based filtering (CMB-specific)
    if shouldFilterByCurrency(rawTx) { return false }
    
    return true
}
```

#### 3. **Raw Transaction Duplicate Detection**
**Pattern**: All services check for duplicates in existing raw transactions using fingerprints.

**Current Implementation**:
- Alipay: Uses optimized Set-based lookup (O(1))
- WeChat: Uses linear search (O(n))
- CMB: Uses linear search (O(n))

**Common Logic**:
```swift
// Pattern - should be O(1) for all
let fingerprint = generateTransactionFingerprint(rawTx)
let isDuplicate = existingRawTransactions.contains { existingRaw in
    generateTransactionFingerprint(existingRaw) == fingerprint
}
```

#### 4. **Transaction Duplicate Detection**
**Pattern**: All services check if raw transactions are already converted to Transaction objects.

**Current Implementation**:
- Alipay: Uses optimized Dictionary lookup (O(1))
- WeChat: Uses linear search (O(n))
- CMB: Uses linear search (O(n))

**Common Logic**:
```swift
// Pattern - should be O(1) for all
let isDuplicateInTransactions = existingTransactions.contains { existingTx in
    guard existingTx.accountId == accountId,
          existingTx.date == rawTx.parsedDate,
          existingTx.amount == expectedAmount else {
        return false
    }
    // Account-specific additional checks (balance, notes, etc.)
    return additionalDuplicateCheck(existingTx, rawTx)
}
```

#### 5. **Sequence Number Management**
**Pattern**: All services manage sequence numbers to preserve transaction order.

**Current Implementation**:
- All accounts (Alipay, WeChat, CMB): Time-based (seconds precision)
- Compare time precise to seconds first
- If time matches exactly (to seconds), use sequence number to order
- Lower sequence number = happens later (more recent)
- Higher sequence number = happens earlier

**Common Logic**:
```swift
// Pattern: All accounts use time-based sequence (seconds precision)
// Build map of existing max sequence numbers per second
var existingMaxSequence: [Date: Int] = [:]
for existingTx in accountTransactions {
    // Round to seconds precision
    let calendar = Calendar.current
    let components = calendar.dateComponents([.year, .month, .day, .hour, .minute, .second], from: existingTx.date)
    let dateKey = calendar.date(from: components) ?? existingTx.date
    let currentMax = existingMaxSequence[dateKey] ?? -1
    if existingTx.sequenceNumber > currentMax {
        existingMaxSequence[dateKey] = existingTx.sequenceNumber
    }
}

// Track sequence numbers for new transactions
// Lower sequence number = happens later (more recent)
// Higher sequence number = happens earlier
var sequenceCounters: [Date: Int] = [:]
for (dateKey, maxSeq) in existingMaxSequence {
    sequenceCounters[dateKey] = maxSeq + 1
}
```

#### 6. **Progress Reporting**
**Pattern**: All services report progress at similar stages.

**Common Stages**:
- Extracting ZIP/parsing file
- Verifying account metadata
- Parsing transactions
- Detecting duplicates
- Saving data

#### 7. **Import Batch Management**
**Pattern**: All services create and save import batches with similar structure.

**Common Logic**:
```swift
// Pattern repeated in all services
let importBatch = ImportBatch(
    sourceFileName: sourceFileName,
    accountId: accountId,
    totalRawRecords: count,
    successfulImports: newTransactions.count,
    duplicateCount: duplicateCount,
    // Account-specific fields (date ranges, etc.)
)
var allBatches = try await repository.loadImportBatches()
allBatches.append(importBatch)
try await repository.saveImportBatches(allBatches)
```

#### 8. **Data Saving Pattern**
**Pattern**: All services save data in the same order.

**Common Logic**:
```swift
// Pattern repeated in all services
// 1. Merge existing and new raw transactions
var allRawTransactions = existingRawTransactions
allRawTransactions.append(contentsOf: rawTransactionsToSave)
try await repository.saveRawTransactions(allRawTransactions)

// 2. Save import batch
// (see Import Batch Management above)

// 3. Calculate latest transaction date
let latestTransactionDate = newTransactions.map { $0.date }.max()
```

#### 9. **Transaction Conversion**
**Pattern**: All services convert raw transactions to Transaction objects with similar logic.

**Common Logic**:
- Extract date, amount, merchant, category
- Handle amount sign (income vs expense)
- Assign sequence number
- Set balance (0 for payment gateways, parsed for CMB)

#### 10. **ZIP File Extraction with Cleanup**
**Pattern**: Services that use ZIP files follow the same extraction and cleanup pattern.

**Common Logic**:
```swift
// Pattern in Alipay and CMB
let zipProcessor = ZipFileProcessingService()
let extractedURL = try await zipProcessor.extractZipFile(...)
defer {
    Task {
        try? await zipProcessor.cleanupTempFiles(at: extractedURL)
    }
}
```

#### 11. **Balance Calculation**
**Pattern**: Some services calculate final balance from transactions.

**Common Logic**:
- CMB: Uses balance from last transaction
- Alipay/WeChat: Balance is always 0 (payment gateways)
- Plaid: Balance comes from API

#### 12. **Error Handling for Empty Results**
**Pattern**: All services check for empty results and throw appropriate errors.

**Common Logic**:
```swift
guard !rawTransactions.isEmpty else {
    throw AppError.importError("导入失败：未找到有效的交易记录。")
}
```

#### 13. **Latest Transaction Date Calculation**
**Pattern**: All services calculate latest transaction date for balance update logic.

**Common Logic**:
```swift
let latestTransactionDate = newTransactions
    .map { $0.date }
    .max()
```

## Part 2: Proposed Folder Structure

### Current Structure Issues

**Problems**:
1. `Features/` folder is getting crowded with mixed concerns
2. Account providers are in Features/ but should be at root level
3. UI features are mixed with business logic
4. Hard to find related files across different account types

### Proposed New Structure

```
MoneyFlow/
├── App/                          # App entry point (unchanged)
│   ├── ContentView.swift
│   └── MoneyFlowApp.swift
│
├── Core/                          # Core business logic (unchanged)
│   ├── Data/
│   ├── DesignSystem/
│   ├── Domain/
│   ├── Services/
│   └── State/
│
├── Accounts/                      # Account-specific implementations
│   ├── Alipay/
│   │   ├── Data/
│   │   │   └── AlipayRepository.swift
│   │   ├── Domain/
│   │   │   ├── ImportAlipayTransactionBatch.swift
│   │   │   └── RawAlipayTransaction.swift
│   │   ├── Services/
│   │   │   ├── AlipayCSVParser.swift
│   │   │   └── AlipayImportService.swift
│   │   └── UI/                    # Account-specific UI
│   │       └── AlipayProcessingRulesView.swift
│   │
│   ├── CMB/
│   │   ├── Data/
│   │   ├── Domain/
│   │   ├── Services/
│   │   └── UI/
│   │       ├── CMBAccountImportConfigurationView.swift
│   │       └── CMBImportView.swift
│   │
│   ├── WeChat/
│   │   ├── Data/
│   │   ├── Domain/
│   │   ├── Services/
│   │   └── UI/
│   │
│   ├── Plaid/
│   │   ├── Data/
│   │   ├── Domain/
│   │   ├── Services/
│   │   └── UI/
│   │       ├── PlaidAccountConfigurationView.swift
│   │       ├── PlaidLinkView.swift
│   │       └── PlaidReconnectView.swift
│   │
│   └── Shared/                     # Shared account infrastructure
│       ├── BaseImportServiceHelper.swift
│       ├── ImportServiceProtocols.swift
│       ├── DuplicateDetectionStrategies.swift
│       ├── SequenceNumberStrategies.swift
│       ├── AccountMetadataManager.swift
│       ├── TransactionFilter.swift
│       └── ImportBatchManager.swift
│
├── Providers/                     # Account providers (moved from Features/)
│   ├── AccountProvider.swift
│   ├── AccountFeatureProvider.swift
│   ├── ProviderRegistry.swift
│   │
│   └── Implementations/           # Provider implementations
│       ├── Alipay/
│       │   ├── AlipayPaymentGateAccountProvider.swift
│       │   └── AlipayPaymentGateAccountFeatureProvider.swift
│       ├── CMB/
│       │   ├── CMBAccountProvider.swift
│       │   └── CMBAccountFeatureProvider.swift
│       ├── WeChat/
│       ├── Plaid/
│       └── DIY/
│           └── DIYConfigurationView.swift
│
├── UI/                            # Application UI (moved from Features/)
│   ├── Dashboard/
│   │   ├── DashboardView.swift
│   │   ├── DashboardViewModel.swift
│   │   └── Components/
│   │       ├── Charts.swift
│   │       ├── RightPanel.swift
│   │       └── SummaryCardsRow.swift
│   │
│   ├── Properties/
│   │   ├── PropertiesView.swift
│   │   ├── AddAccountView.swift
│   │   └── Components/
│   │       ├── AccountDetailPane.swift
│   │       ├── AccountHeaderCard.swift
│   │       ├── AccountMonthlyList.swift
│   │       ├── AccountRow.swift
│   │       ├── AccountsLeftPane.swift
│   │       ├── EmptyStates.swift
│   │       ├── HeaderSearchRow.swift
│   │       ├── MonthlySection.swift
│   │       ├── TransactionDayGroup.swift
│   │       ├── TransactionDetailView.swift
│   │       ├── TransactionListForMonth.swift
│   │       └── TransactionRow.swift
│   │
│   ├── Summary/
│   │   ├── SummaryView.swift
│   │   ├── SummaryViewModel.swift
│   │   └── Components/
│   │       └── SummaryComponents.swift
│   │
│   ├── Settings/
│   │   └── SettingsView.swift
│   │
│   ├── Sidebar/
│   │   └── Sidebar.swift
│   │
│   └── Debug/                     # Debug UI (moved from Features/)
│       ├── DebugPaneView.swift
│       ├── AlipayAutoImportView.swift
│       ├── CMBAutoImportView.swift
│       └── WeChatAutoImportView.swift
│
└── Assets.xcassets/               # Assets (unchanged)
```

### Benefits of New Structure

1. **Clear Separation of Concerns**:
   - `Accounts/` - Business logic for account types
   - `Providers/` - Account creation and feature providers
   - `UI/` - Application UI components
   - `Core/` - Core business logic

2. **Easier Navigation**:
   - All account-related code in `Accounts/`
   - All UI in `UI/`
   - All providers in `Providers/`

3. **Better Scalability**:
   - Easy to add new account types (just add to `Accounts/`)
   - Easy to add new UI features (just add to `UI/`)
   - Shared infrastructure in `Accounts/Shared/`

4. **Consistent Structure**:
   - Each account type follows same structure (Data, Domain, Services, UI)
   - Shared code in `Accounts/Shared/`

## Part 3: Base Import Service Design

### Architecture Overview

```
BaseImportServiceHelper (actor class)
├── Common Infrastructure
│   ├── Data Loading (existing raw transactions, Transaction objects)
│   ├── Duplicate Detection Framework (O(1) lookups)
│   ├── Sequence Number Management
│   ├── Account Metadata Management
│   ├── Transaction Filtering
│   ├── Import Batch Management
│   └── Data Saving Helpers
│
└── Account-Specific Hooks (via Strategy Pattern)
    ├── DuplicateDetectionStrategy (fingerprint generation, duplicate checking)
    ├── SequenceNumberStrategy (time-based, seconds precision)
    ├── TransactionFilter (status, amount, currency filters)
    └── Transaction Conversion Logic (account-specific)
```

### Design Approach: Protocol + Helper Class

**Why Protocol-Based Helper?**
- Flexible - services can override any method
- Type-safe
- Easy to test
- Follows Swift protocol-oriented design
- No inheritance limitations

### Core Protocols

```swift
/// Protocol for account-specific raw transaction types
protocol RawTransactionProtocol {
    /// Get parsed date (if available)
    var parsedDate: Date? { get }
    
    /// Get parsed amount (if available)
    var parsedAmount: Decimal? { get }
    
    /// Get parsed balance (if available, for accounts that track balance)
    var parsedBalance: Decimal? { get }
    
    /// Check if transaction is income
    var isIncome: Bool { get }
    
    /// Get order ID or transaction ID for duplicate detection
    var transactionOrderId: String { get }
}

/// Protocol for account-specific duplicate detection strategies
protocol DuplicateDetectionStrategy {
    associatedtype RawTransactionType: RawTransactionProtocol
    
    /// Generate fingerprint for raw transaction (for raw-to-raw duplicate check)
    func generateFingerprint(for rawTx: RawTransactionType) -> String
    
    /// Check if raw transaction is duplicate of existing Transaction
    func isDuplicate(
        rawTx: RawTransactionType,
        existingTx: Transaction,
        accountId: UUID
    ) -> Bool
}

/// Protocol for sequence number assignment strategies
protocol SequenceNumberStrategy {
    /// Get date key for grouping transactions (time-based, seconds precision)
    func getDateKey(for date: Date) -> Date
    
    /// Get next sequence number for a date
    func getNextSequenceNumber(
        for date: Date,
        counters: inout [Date: Int]
    ) -> Int
}

/// Protocol for transaction filtering
protocol TransactionFilter {
    associatedtype RawTransactionType: RawTransactionProtocol
    
    /// Check if transaction should be excluded
    func shouldExclude(_ transaction: RawTransactionType) -> Bool
}
```

### BaseImportServiceHelper Implementation

```swift
/// Main helper class providing common import service infrastructure
actor BaseImportServiceHelper {
    private let filePersistence: FilePersistenceProtocol
    
    init(filePersistence: FilePersistenceProtocol? = nil) {
        self.filePersistence = filePersistence ?? FilePersistence()
    }
    
    // MARK: - Data Loading
    
    struct ExistingData {
        let allTransactions: [Transaction]
        let accountTransactions: [Transaction]
    }
    
    /// Load existing data for duplicate detection
    func loadExistingData(accountId: UUID) async throws -> ExistingData {
        let allTransactions = try await filePersistence.loadTransactions()
        let accountTransactions = allTransactions.filter { $0.accountId == accountId }
        return ExistingData(
            allTransactions: allTransactions,
            accountTransactions: accountTransactions
        )
    }
    
    // MARK: - Account Metadata Management
    
    /// Verify and save account metadata
    func verifyAndSaveMetadata<MetadataType>(
        accountId: UUID,
        newValue: String?,
        storedMetadata: MetadataType?,
        metadataKey: String,
        errorMessage: (String, String) -> String,
        createMetadata: (UUID, String) -> MetadataType,
        repository: AccountMetadataRepository<MetadataType>
    ) async throws {
        guard let newValue = newValue else { return }
        
        if let storedValue = storedMetadata?.metadataValue {
            if storedValue != newValue {
                throw AppError.importError(errorMessage(newValue, storedValue))
            }
        } else {
            // Save metadata from this import
            let metadata = createMetadata(accountId, newValue)
            try await repository.saveAccountMetadata(metadata)
        }
    }
    
    // MARK: - Transaction Filtering
    
    /// Filter transactions using provided filters
    func filterTransactions<RawTx: RawTransactionProtocol>(
        _ transactions: [RawTx],
        filters: [any TransactionFilter<RawTx>]
    ) -> [RawTx] {
        return transactions.filter { rawTx in
            for filter in filters {
                if filter.shouldExclude(rawTx) {
                    return false
                }
            }
            return true
        }
    }
    
    // MARK: - Duplicate Detection
    
    struct DuplicateDetectionStructures {
        let rawFingerprints: Set<String>
        let transactionLookup: [String: [String]]
    }
    
    /// Build optimized lookup structures for duplicate detection (O(1) lookups)
    func buildDuplicateDetectionStructures<RawTx: RawTransactionProtocol>(
        rawTransactions: [RawTx],
        existingRawTransactions: [RawTx],
        existingTransactions: [Transaction],
        accountId: UUID,
        strategy: some DuplicateDetectionStrategy<RawTx>
    ) -> DuplicateDetectionStructures {
        // Build Set of fingerprints from existing raw transactions (O(1) lookup)
        let rawFingerprints = Set(existingRawTransactions.map { strategy.generateFingerprint(for: $0) })
        
        // Build Dictionary for quick lookup of existing transactions
        var transactionLookup: [String: [String]] = [:]
        for existingTx in existingTransactions {
            guard existingTx.accountId == accountId else { continue }
            let key = "\(existingTx.accountId.uuidString)|\(existingTx.date.timeIntervalSince1970)|\(existingTx.amount)"
            if let notes = existingTx.notes {
                var notesList = transactionLookup[key] ?? []
                notesList.append(notes)
                transactionLookup[key] = notesList
            }
        }
        
        return DuplicateDetectionStructures(
            rawFingerprints: rawFingerprints,
            transactionLookup: transactionLookup
        )
    }
    
    /// Check if raw transaction is duplicate (O(1) lookup)
    func isDuplicate<RawTx: RawTransactionProtocol>(
        rawTx: RawTx,
        structures: DuplicateDetectionStructures,
        existingTransactions: [Transaction],
        accountId: UUID,
        strategy: some DuplicateDetectionStrategy<RawTx>
    ) -> Bool {
        // Check against raw transactions (O(1))
        let fingerprint = strategy.generateFingerprint(for: rawTx)
        if structures.rawFingerprints.contains(fingerprint) {
            return true
        }
        
        // Check against existing transactions (O(1) for lookup, O(n) for notes check)
        for existingTx in existingTransactions {
            if strategy.isDuplicate(rawTx: rawTx, existingTx: existingTx, accountId: accountId) {
                return true
            }
        }
        
        return false
    }
    
    // MARK: - Sequence Number Management
    
    /// Build sequence number counters from existing transactions
    func buildSequenceCounters(
        from transactions: [Transaction],
        strategy: some SequenceNumberStrategy
    ) -> [Date: Int] {
        var counters: [Date: Int] = [:]
        for tx in transactions {
            let dateKey = strategy.getDateKey(for: tx.date)
            let currentMax = counters[dateKey] ?? -1
            if tx.sequenceNumber > currentMax {
                counters[dateKey] = tx.sequenceNumber
            }
        }
        return counters
    }
    
    /// Get next sequence number for a date
    func getNextSequenceNumber(
        for date: Date,
        counters: inout [Date: Int],
        strategy: some SequenceNumberStrategy
    ) -> Int {
        return strategy.getNextSequenceNumber(for: date, counters: &counters)
    }
    
    // MARK: - Import Batch Management
    
    /// Create and save import batch
    func createAndSaveImportBatch<BatchType>(
        sourceFileName: String,
        accountId: UUID,
        totalRawRecords: Int,
        successfulImports: Int,
        duplicateCount: Int,
        createBatch: (UUID, String, Int, Int, Int) -> BatchType,
        repository: ImportBatchRepository<BatchType>
    ) async throws {
        let importBatch = createBatch(
            accountId,
            sourceFileName,
            totalRawRecords,
            successfulImports,
            duplicateCount
        )
        var allBatches = try await repository.loadImportBatches()
        allBatches.append(importBatch)
        try await repository.saveImportBatches(allBatches)
    }
    
    // MARK: - Data Saving
    
    /// Save import data (raw transactions and import batch)
    func saveImportData<RawTx, BatchType>(
        newRawTransactions: [RawTx],
        existingRawTransactions: [RawTx],
        importBatch: BatchType,
        rawRepository: RawTransactionRepository<RawTx>,
        batchRepository: ImportBatchRepository<BatchType>
    ) async throws {
        // Merge and save raw transactions
        var allRawTransactions = existingRawTransactions
        allRawTransactions.append(contentsOf: newRawTransactions)
        try await rawRepository.saveRawTransactions(allRawTransactions)
        
        // Save import batch
        var allBatches = try await batchRepository.loadImportBatches()
        allBatches.append(importBatch)
        try await batchRepository.saveImportBatches(allBatches)
    }
    
    // MARK: - Result Building
    
    /// Build import result
    func buildImportResult(
        transactions: [Transaction],
        newAccountBalance: Decimal?,
        latestTransactionDate: Date?
    ) -> ImportResult {
        return ImportResult(
            transactions: transactions,
            importedCount: transactions.count,
            skippedRows: [],
            newAccountBalance: newAccountBalance,
            latestTransactionDate: latestTransactionDate
        )
    }
}
```

### Account-Specific Strategy Implementations

```swift
// Accounts/Shared/DuplicateDetectionStrategies.swift

// AlipayDuplicateDetectionStrategy
struct AlipayDuplicateDetectionStrategy: DuplicateDetectionStrategy {
    typealias RawTransactionType = RawAlipayTransaction
    
    func generateFingerprint(for rawTx: RawAlipayTransaction) -> String {
        let dateStr = rawTx.parsedDate.map { DateFormatter.iso8601.string(from: $0) } ?? rawTx.rawTransactionTime
        let amountStr = rawTx.parsedAmount?.description ?? rawTx.rawAmount
        return "\(dateStr)|\(amountStr)|\(rawTx.rawTransactionOrderId)|\(rawTx.accountId.uuidString)"
    }
    
    func isDuplicate(
        rawTx: RawAlipayTransaction,
        existingTx: Transaction,
        accountId: UUID
    ) -> Bool {
        guard existingTx.accountId == accountId,
              existingTx.date == rawTx.parsedDate else {
            return false
        }
        
        let expectedAmount: Decimal? = {
            guard let amount = rawTx.parsedAmount else { return nil }
            return rawTx.isIncome ? amount : -amount
        }()
        
        guard let expectedAmount = expectedAmount,
              existingTx.amount == expectedAmount,
              let notes = existingTx.notes else {
            return false
        }
        
        return notes.contains(rawTx.rawTransactionOrderId)
    }
}

// CMBDuplicateDetectionStrategy
struct CMBDuplicateDetectionStrategy: DuplicateDetectionStrategy {
    typealias RawTransactionType = RawCMBTransaction
    
    func generateFingerprint(for rawTx: RawCMBTransaction) -> String {
        let dateStr = rawTx.parsedDate.map { DateFormatter.iso8601.string(from: $0) } ?? ""
        let amountStr = rawTx.parsedAmount?.description ?? ""
        let balanceStr = rawTx.parsedBalance?.description ?? ""
        return "\(dateStr)|\(amountStr)|\(balanceStr)|\(rawTx.accountId.uuidString)"
    }
    
    func isDuplicate(
        rawTx: RawCMBTransaction,
        existingTx: Transaction,
        accountId: UUID
    ) -> Bool {
        guard existingTx.accountId == accountId,
              existingTx.date == rawTx.parsedDate,
              existingTx.amount == rawTx.parsedAmount else {
            return false
        }
        
        guard let newBalance = rawTx.parsedBalance else {
            return false
        }
        
        return existingTx.balance == newBalance
    }
}

// Accounts/Shared/SequenceNumberStrategies.swift

// TimeBasedSequenceStrategy (for all accounts: Alipay, WeChat, CMB)
// Compare time precise to seconds first
// If time matches exactly (to seconds), use sequence number to order
// Lower sequence number = happens later (more recent)
// Higher sequence number = happens earlier
struct TimeBasedSequenceStrategy: SequenceNumberStrategy {
    func getDateKey(for date: Date) -> Date {
        let calendar = Calendar.current
        let components = calendar.dateComponents([.year, .month, .day, .hour, .minute, .second], from: date)
        return calendar.date(from: components) ?? date
    }
    
    func getNextSequenceNumber(
        for date: Date,
        counters: inout [Date: Int]
    ) -> Int {
        let dateKey = getDateKey(for: date)
        let current = counters[dateKey] ?? -1
        let next = current + 1
        counters[dateKey] = next
        return next
    }
}

// Note: All accounts (Alipay, WeChat, CMB) use TimeBasedSequenceStrategy
// CMB does NOT use day-based sequence - it uses the same time-based logic
// as other accounts, precise to seconds

// Accounts/Shared/TransactionFilter.swift

struct StatusFilter<RawTx: RawTransactionProtocol>: TransactionFilter {
    let excludedStatuses: [String]
    
    func shouldExclude(_ transaction: RawTx) -> Bool {
        // Account-specific status check
        if let status = getStatus(transaction) {
            return excludedStatuses.contains(status)
        }
        return false
    }
    
    private func getStatus(_ transaction: RawTx) -> String? {
        // This would need to be implemented per account type
        // Could use a protocol extension or associated type
        return nil
    }
}

struct AmountFilter<RawTx: RawTransactionProtocol>: TransactionFilter {
    func shouldExclude(_ transaction: RawTx) -> Bool {
        if let amount = transaction.parsedAmount, amount == 0 {
            return true
        }
        return false
    }
}
```

### Usage Example with New Structure

```swift
// Accounts/Alipay/Services/AlipayImportService.swift
struct AlipayImportService {
    private let alipayRepository: AlipayRepositoryProtocol
    private let helper = BaseImportServiceHelper()
    
    func importAlipayZipCSV(
        zipURL: URL,
        password: String,
        accountId: UUID,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult {
        // 1. Extract ZIP (account-specific)
        let zipProcessor = ZipFileProcessingService()
        let extractedCSVURL = try await zipProcessor.extractZipFile(
            zipURL: zipURL,
            password: password,
            fileExtension: "csv"
        )
        defer {
            Task {
                try? await zipProcessor.cleanupTempFiles(at: extractedCSVURL)
            }
        }
        
        // 2. Parse CSV (account-specific)
        onProgress(ImportProgress(stage: .parsingPDF, current: 0, total: 1, message: "正在解析CSV文件..."))
        let csvParser = AlipayCSVParser(csvURL: extractedCSVURL)
        let csvInfo = try await csvParser.getInfo()
        
        // 3. Verify metadata (using helper)
        try await helper.verifyAndSaveMetadata(
            accountId: accountId,
            newValue: csvInfo.email,
            storedMetadata: try await alipayRepository.loadAccountMetadata(accountId: accountId),
            metadataKey: "email",
            errorMessage: { new, stored in "账户验证失败：文件中的邮箱(\(new))与已保存的邮箱(\(stored))不匹配" },
            createMetadata: { id, email in AlipayAccountMetadata(accountId: id, email: email) },
            repository: alipayRepository
        )
        
        // 4. Parse transactions (account-specific)
        onProgress(ImportProgress(stage: .extractingText, current: 0, total: 1, message: "正在解析交易数据..."))
        let allRawTransactions = try await csvParser.parseTransactions(
            accountId: accountId,
            importBatchId: UUID(),
            sourceFileName: zipURL.lastPathComponent
        )
        
        // 5. Filter transactions (using helper)
        let rawTransactions = helper.filterTransactions(
            allRawTransactions,
            filters: [
                StatusFilter(excludedStatuses: ["还款失败"]),
                AmountFilter()
            ]
        )
        
        guard !rawTransactions.isEmpty else {
            throw AppError.importError("导入失败：未找到有效的交易记录。")
        }
        
        // 6. Load existing data (using helper)
        let existingData = try await helper.loadExistingData(accountId: accountId)
        let existingRawTransactions = try await alipayRepository.loadRawTransactions()
        
        // 7. Build duplicate detection structures (using helper)
        let strategy = AlipayDuplicateDetectionStrategy()
        let duplicateStructures = helper.buildDuplicateDetectionStructures(
            rawTransactions: rawTransactions,
            existingRawTransactions: existingRawTransactions,
            existingTransactions: existingData.allTransactions,
            accountId: accountId,
            strategy: strategy
        )
        
        // 8. Process transactions (using helper for sequence numbers)
        let seqStrategy = TimeBasedSequenceStrategy()
        var sequenceCounters = helper.buildSequenceCounters(
            from: existingData.accountTransactions,
            strategy: seqStrategy
        )
        
        var newTransactions: [Transaction] = []
        var duplicateCount = 0
        var rawTransactionsToSave: [RawAlipayTransaction] = []
        
        for (index, rawTx) in rawTransactions.enumerated() {
            onProgress(ImportProgress(
                stage: .identifyingDuplicates,
                current: index + 1,
                total: rawTransactions.count,
                message: "正在检测重复交易 (\(index + 1)/\(rawTransactions.count))..."
            ))
            
            if helper.isDuplicate(
                rawTx: rawTx,
                structures: duplicateStructures,
                existingTransactions: existingData.allTransactions,
                accountId: accountId,
                strategy: strategy
            ) {
                duplicateCount += 1
                continue
            }
            
            guard let date = rawTx.parsedDate else { continue }
            let sequenceNumber = helper.getNextSequenceNumber(
                for: date,
                counters: &sequenceCounters,
                strategy: seqStrategy
            )
            
            guard let transaction = convertRawToTransaction(
                rawTx,
                sequenceNumber: sequenceNumber,
                balance: 0
            ) else { continue }
            
            newTransactions.append(transaction)
            rawTransactionsToSave.append(rawTx)
        }
        
        // 9. Save data (using helper)
        onProgress(ImportProgress(stage: .importingData, current: 0, total: 1, message: "正在保存数据..."))
        
        let importBatch = ImportAlipayTransactionBatch(
            sourceFileName: zipURL.lastPathComponent,
            accountId: accountId,
            totalRawRecords: allRawTransactions.count,
            successfulImports: newTransactions.count,
            duplicateCount: duplicateCount,
            csvStartDate: csvInfo.startDate,
            csvEndDate: csvInfo.endDate
        )
        
        try await helper.saveImportData(
            newRawTransactions: rawTransactionsToSave,
            existingRawTransactions: existingRawTransactions,
            importBatch: importBatch,
            rawRepository: alipayRepository,
            batchRepository: alipayRepository
        )
        
        // 10. Build result (using helper)
        onProgress(ImportProgress(stage: .importingData, current: 1, total: 1, message: "导入完成"))
        
        return helper.buildImportResult(
            transactions: newTransactions,
            newAccountBalance: nil,
            latestTransactionDate: newTransactions.map { $0.date }.max()
        )
    }
    
    // Account-specific conversion logic
    private func convertRawToTransaction(
        _ raw: RawAlipayTransaction,
        sequenceNumber: Int,
        balance: Decimal
    ) -> Transaction? {
        // Account-specific conversion logic
        // ...
    }
}
```

**Note**: The above example shows Alipay, but CMB and WeChat would use the exact same pattern:
- All accounts use `TimeBasedSequenceStrategy` (seconds precision)
- CMB does NOT use day-based sequence - it uses the same time-based logic
- Sequence number ordering: Lower number = later (more recent), Higher number = earlier

## Migration Plan

### Phase 1: Create Base Infrastructure
1. Create `Accounts/Shared/` folder structure
2. Implement `BaseImportServiceHelper` with core functionality
3. Create strategy protocols and default implementations
4. Create helper protocols for metadata, filtering, batch management
5. Test with one account type (e.g., Alipay)

### Phase 2: Migrate Folder Structure
1. Create new folder structure (`Accounts/`, `Providers/`, `UI/`)
2. Move account implementations to `Accounts/`
3. Move providers to `Providers/`
4. Move UI to `UI/`
5. Update imports throughout codebase
6. Update Xcode project file

### Phase 3: Migrate Import Services
1. Migrate Alipay to use base helper
2. Verify Alipay import works correctly
3. Migrate WeChat to use base helper
4. Migrate CMB to use base helper
5. Remove duplicate code from migrated services

### Phase 4: Cleanup
1. Remove old `Features/` folder
2. Update documentation
3. Update tests
4. Verify all imports work correctly

## Benefits Summary

1. **Code Reuse**: ~60-70% of import service code can be shared
2. **Consistency**: All services use same O(1) duplicate detection
3. **Maintainability**: Fix bugs in one place
4. **Extensibility**: Easy to add new account types
5. **Testability**: Helper can be tested independently
6. **Organization**: Clear folder structure makes navigation easier
7. **Scalability**: Structure supports growth
8. **Performance**: Optimized O(1) lookups by default
9. **Type Safety**: Protocol-based design provides compile-time guarantees

## Alternative: Simpler Helper Functions (Stepping Stone)

If the full protocol-based approach is too complex initially, we could start with simpler helper functions:

```swift
/// Helper functions for common import operations
enum ImportServiceHelpers {
    /// Load existing data for duplicate detection
    static func loadExistingData(
        accountId: UUID,
        filePersistence: FilePersistenceProtocol
    ) async throws -> (all: [Transaction], account: [Transaction])
    
    /// Build Set of fingerprints from raw transactions (O(1) lookup)
    static func buildRawFingerprintSet<RawTx>(
        _ transactions: [RawTx],
        fingerprintGenerator: (RawTx) -> String
    ) -> Set<String>
    
    /// Build sequence number counters from existing transactions
    static func buildSequenceCounters(
        transactions: [Transaction],
        dateKeyGenerator: (Date) -> Date
    ) -> [Date: Int]
    
    /// Verify and save account metadata
    static func verifyAndSaveMetadata<MetadataType>(
        accountId: UUID,
        newValue: String?,
        storedMetadata: MetadataType?,
        errorMessage: (String, String) -> String,
        createMetadata: (UUID, String) -> MetadataType,
        repository: AccountMetadataRepository<MetadataType>
    ) async throws
}
```

This simpler approach could be a stepping stone to the full protocol-based solution.
