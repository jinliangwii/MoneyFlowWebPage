# Account Book (账本) - Modular Design Proposal

## Overview

This proposal designs the **Account Book (账本)** feature as a **completely independent module**, following the existing architecture patterns. The module is self-contained with clear separation of data, business logic, and UI layers.

## Design Principles

1. **Module Independence**: Ledger module is isolated from core app logic
2. **Layer Separation**: Clear boundaries between Domain, Data, Services, and UI
3. **Protocol-Based**: Dependencies flow through protocols, not concrete types
4. **Minimal Core Integration**: Core app only knows about ledger protocols, not implementations

---

## Module Structure

```
MoneyFlow/
├── Ledgers/                              # 🆕 Independent Ledger Module
│   ├── Domain/                           # Domain Models & Business Rules
│   │   ├── Models/
│   │   │   ├── Ledger.swift             # Ledger entity
│   │   │   ├── LedgerRules.swift        # Filtering rules
│   │   │   ├── LedgerStatistics.swift   # Statistics model
│   │   │   ├── LedgerBalanceStatistics.swift # Balance statistics model
│   │   │   ├── LedgerCategorySystem.swift # Category system for each ledger
│   │   │   └── LedgerType.swift         # Ledger type enum
│   │   └── Services/
│   │       ├── LedgerFilterService.swift # Rule-based filtering logic
│   │       ├── LedgerStatisticsService.swift # Statistics calculation
│   │       ├── LedgerMonthlyStatisticsService.swift # Monthly statistics calculation
│   │       ├── LedgerBalanceService.swift # Balance calculation and tracking
│   │       └── LedgerCategoryService.swift # Category mapping and classification
│   │
│   ├── Data/                             # Data Access Layer
│   │   ├── Protocols/
│   │   │   ├── LedgerRepositoryProtocol.swift
│   │   │   └── LedgerCategoryRepositoryProtocol.swift
│   │   └── Repositories/
│   │       ├── FileLedgerRepository.swift
│   │       ├── InMemoryLedgerRepository.swift
│   │       ├── FileLedgerCategoryRepository.swift
│   │       └── InMemoryLedgerCategoryRepository.swift
│   │
│   ├── Services/                         # Business Logic Layer
│   │   ├── LedgerService.swift          # Main service orchestrator
│   │   └── LedgerRuleEngine.swift       # Rule evaluation engine
│   │
│   └── UI/                               # UI Components
│       ├── AccountBookView.swift        # Main container
│       ├── Components/
│       │   ├── LedgerListPane.swift
│       │   ├── LedgerDetailPane.swift
│       │   ├── LedgerRow.swift
│       │   ├── LedgerHeaderCard.swift
│       │   ├── LedgerTransactionList.swift
│       │   ├── MonthCardSelector.swift      # 🆕 Month card selection (as shown in image)
│       │   ├── MonthCard.swift               # 🆕 Individual month card
│       │   └── AddLedgerView.swift
│       └── ViewModels/
│           └── AccountBookViewModel.swift
│
├── Core/                                 # Core App (minimal changes)
│   ├── Dependencies/
│   │   └── AppDependencies.swift        # Add LedgerRepositoryProtocol
│   └── State/
│       └── AppState.swift                # Add ledger management (via LedgerService)
│
└── UI/
    ├── Sidebar/
    │   └── Sidebar.swift                 # Add accountBook item
    └── ContentView.swift                 # Route to AccountBookView
```

---

## Layer 1: Domain Layer (`Ledgers/Domain/`)

### Purpose
Define ledger domain models and business rules. **No dependencies on other layers.**

### Files

#### `Ledgers/Domain/Models/Ledger.swift`

```swift
// Ledgers/Domain/Models/Ledger.swift
import Foundation

/// Ledger entity - represents a virtual view over transactions
public struct Ledger: Identifiable, Codable, Hashable {
    public var id: UUID
    public var name: String
    public var type: LedgerType
    public var isDefault: Bool
    public var rules: LedgerRules
    public var createdAt: Date
    public var updatedAt: Date
    
    public init(
        id: UUID = UUID(),
        name: String,
        type: LedgerType,
        isDefault: Bool = false,
        rules: LedgerRules,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.name = name
        self.type = type
        self.isDefault = isDefault
        
        // Enforce balance tracking for core ledger
        var adjustedRules = rules
        if type == .core {
            adjustedRules.trackBalances = true
            adjustedRules.balanceTrackingMode = .accountBased
        }
        self.rules = adjustedRules
        
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }
    
    /// Validate ledger configuration
    public func validate() throws {
        // Core ledger must track balances
        if type == .core && !rules.trackBalances {
            throw LedgerError.invalidConfiguration("核心账本必须启用余额统计")
        }
        
        // Single account mode requires exactly one account
        if rules.balanceTrackingMode == .singleAccount {
            guard let accountIds = rules.includeAccountIds,
                  accountIds.count == 1 else {
                throw LedgerError.invalidConfiguration("单账户模式需要绑定且仅绑定一个账户")
            }
        }
    }
}

public enum LedgerType: String, Codable, CaseIterable {
    case core
    case lifestyle
    case investment
    case travel
    case custom
    
    public var displayName: String {
        switch self {
        case .core: return "核心账本"
        case .lifestyle: return "生活账本"
        case .investment: return "投资账本"
        case .travel: return "旅游账本"
        case .custom: return "自定义"
        }
    }
    
    public var systemImage: String {
        switch self {
        case .core: return "book.fill"
        case .lifestyle: return "house.fill"
        case .investment: return "chart.line.uptrend.xyaxis"
        case .travel: return "airplane"
        case .custom: return "book.closed"
        }
    }
}
```

#### `Ledgers/Domain/Models/LedgerRules.swift`

```swift
// Ledgers/Domain/Models/LedgerRules.swift
import Foundation

/// Filtering rules for a ledger (like SQL WHERE clause)
public struct LedgerRules: Codable, Hashable {
    // Include/Exclude Logic
    public var includeAll: Bool
    public var includeAccountIds: Set<UUID>?
    public var excludeAccountIds: Set<UUID>?
    
    // Category/Tag Filtering
    /// Category IDs to include (uses ledger's custom category system)
    public var includeCategories: Set<UUID>?
    /// Category IDs to exclude (uses ledger's custom category system)
    public var excludeCategories: Set<UUID>?
    /// Tags to include (for flexible tagging)
    public var includeTags: Set<String>?
    /// Tags to exclude (for flexible tagging)
    public var excludeTags: Set<String>?
    
    // Date Range Filtering
    public var dateRange: DateRangeFilter?
    
    // Amount Filtering
    public var minAmount: Decimal?
    public var maxAmount: Decimal?
    public var excludeSmallAmounts: Bool
    public var smallAmountThreshold: Decimal?
    
    // Transaction Type Filtering
    public var includeIncome: Bool
    public var includeExpense: Bool
    public var includeTransfer: Bool
    
    // Balance Statistics Configuration
    /// Whether to track account balances for this ledger
    /// - Core ledger: Always true (required)
    /// - Scenario ledgers: Optional (default false)
    public var trackBalances: Bool
    
    /// Balance tracking mode
    public var balanceTrackingMode: BalanceTrackingMode
    
    public init(
        includeAll: Bool = false,
        includeAccountIds: Set<UUID>? = nil,
        excludeAccountIds: Set<UUID>? = nil,
        includeCategories: Set<UUID>? = nil,
        excludeCategories: Set<UUID>? = nil,
        includeTags: Set<String>? = nil,
        excludeTags: Set<String>? = nil,
        dateRange: DateRangeFilter? = nil,
        minAmount: Decimal? = nil,
        maxAmount: Decimal? = nil,
        excludeSmallAmounts: Bool = false,
        smallAmountThreshold: Decimal? = nil,
        includeIncome: Bool = true,
        includeExpense: Bool = true,
        includeTransfer: Bool = true,
        trackBalances: Bool = false,
        balanceTrackingMode: BalanceTrackingMode = .accountBased
    ) {
        self.includeAll = includeAll
        self.includeAccountIds = includeAccountIds
        self.excludeAccountIds = excludeAccountIds
        self.includeCategories = includeCategories
        self.excludeCategories = excludeCategories
        self.includeTags = includeTags
        self.excludeTags = excludeTags
        self.dateRange = dateRange
        self.minAmount = minAmount
        self.maxAmount = maxAmount
        self.excludeSmallAmounts = excludeSmallAmounts
        self.smallAmountThreshold = smallAmountThreshold
        self.includeIncome = includeIncome
        self.includeExpense = includeExpense
        self.includeTransfer = includeTransfer
        self.trackBalances = trackBalances
        self.balanceTrackingMode = balanceTrackingMode
    }
    
    /// Default rules for core ledger (include all transactions, track balances)
    public static var coreLedgerRules: LedgerRules {
        LedgerRules(
            includeAll: true,
            trackBalances: true,  // Core ledger MUST track balances
            balanceTrackingMode: .accountBased  // Track by account dimension
        )
    }
    
    /// Rules for lifestyle ledger (exclude investment and transfers, no balance tracking)
    public static var lifestyleLedgerRules: LedgerRules {
        LedgerRules(
            includeAll: true,
            excludeCategories: ["投资", "转账"],
            includeIncome: true,
            includeExpense: true,
            includeTransfer: false,
            trackBalances: false  // Scenario ledger: default no balance tracking
        )
    }
}

/// Balance tracking mode for ledgers
public enum BalanceTrackingMode: String, Codable {
    /// Track balances by account dimension (for core ledger)
    /// Each account has its own balance, total balance = sum of all accounts
    case accountBased
    
    /// Track balance for a single bound account (for scenario ledger with single account)
    /// Only applicable when ledger is bound to one account
    case singleAccount
    
    /// Track total balance as a fund amount (for special savings ledger)
    /// Balance represents accumulated amount, not tied to actual bank account
    case fundBased
}

public struct DateRangeFilter: Codable, Hashable {
    public var startDate: Date?
    public var endDate: Date?
    public var isRelative: Bool
    public var relativeDays: Int?
    
    public init(
        startDate: Date? = nil,
        endDate: Date? = nil,
        isRelative: Bool = false,
        relativeDays: Int? = nil
    ) {
        self.startDate = startDate
        self.endDate = endDate
        self.isRelative = isRelative
        self.relativeDays = relativeDays
    }
    
    public func effectiveRange() -> (start: Date?, end: Date?) {
        if isRelative, let days = relativeDays {
            let end = Date()
            let start = Calendar.current.date(byAdding: .day, value: -days, to: end)
            return (start, end)
        }
        return (startDate, endDate)
    }
}
```

#### `Ledgers/Domain/Models/LedgerStatistics.swift`

```swift
// Ledgers/Domain/Models/LedgerStatistics.swift
import Foundation

/// Statistics for a ledger
public struct LedgerStatistics: Hashable {
    public let totalTransactions: Int
    public let totalIncome: Decimal
    public let totalExpenses: Decimal
    public let net: Decimal
    
    public init(
        totalTransactions: Int,
        totalIncome: Decimal,
        totalExpenses: Decimal,
        net: Decimal
    ) {
        self.totalTransactions = totalTransactions
        self.totalIncome = totalIncome
        self.totalExpenses = totalExpenses
        self.net = net
    }
}
```

#### `Ledgers/Domain/Services/LedgerFilterService.swift`

```swift
// Ledgers/Domain/Services/LedgerFilterService.swift
import Foundation

/// Service for filtering transactions based on ledger rules
/// This is pure business logic, no dependencies on data layer
public struct LedgerFilterService {
    
    /// Filter transactions based on ledger rules
    /// - Parameters:
    ///   - transactions: All available transactions
    ///   - accounts: All available accounts (for account-based filtering)
    ///   - rules: Filtering rules to apply
    /// - Returns: Filtered transactions that match the rules
    public static func filterTransactions(
        _ transactions: [Transaction],
        accounts: [Account],
        rules: LedgerRules
    ) -> [Transaction] {
        var filtered = transactions
        
        // Rule 1: Include All (core ledger)
        if !rules.includeAll {
            // Rule 2: Account filtering
            if let includeIds = rules.includeAccountIds, !includeIds.isEmpty {
                filtered = filtered.filter { includeIds.contains($0.accountId) }
            }
            if let excludeIds = rules.excludeAccountIds, !excludeIds.isEmpty {
                filtered = filtered.filter { !excludeIds.contains($0.accountId) }
            }
        }
        
        // Rule 3: Date range filtering
        if let dateRange = rules.dateRange {
            let (start, end) = dateRange.effectiveRange()
            filtered = filtered.filter { tx in
                if let start = start, tx.date < start { return false }
                if let end = end, tx.date > end { return false }
                return true
            }
        }
        
        // Rule 4: Amount filtering
        if let minAmount = rules.minAmount {
            filtered = filtered.filter { abs($0.amount) >= minAmount }
        }
        if let maxAmount = rules.maxAmount {
            filtered = filtered.filter { abs($0.amount) <= maxAmount }
        }
        if rules.excludeSmallAmounts, let threshold = rules.smallAmountThreshold {
            filtered = filtered.filter { abs($0.amount) >= threshold }
        }
        
        // Rule 5: Transaction type filtering
        var typeFiltered: [Transaction] = []
        if rules.includeIncome {
            typeFiltered.append(contentsOf: filtered.filter { $0.amount > 0 })
        }
        if rules.includeExpense {
            typeFiltered.append(contentsOf: filtered.filter { $0.amount < 0 })
        }
        // Remove duplicates
        filtered = Array(Set(typeFiltered))
        
        // Rule 6: Category/Tag filtering
        // Note: This requires category mappings from LedgerCategoryRepository
        // Category filtering is handled by the caller who has access to category mappings
        // For now, we skip category filtering here - it's done in LedgerService
        
        // Sort by date (newest first)
        return Transaction.sortByDateDescending(filtered)
    }
}
```

#### `Ledgers/Domain/Services/LedgerStatisticsService.swift`

```swift
// Ledgers/Domain/Services/LedgerStatisticsService.swift
import Foundation

/// Service for calculating ledger statistics
public struct LedgerStatisticsService {
    
    /// Calculate statistics for filtered transactions
    /// - Parameter transactions: Filtered transactions for the ledger
    /// - Returns: Statistics including income, expenses, net, and count
    public static func calculateStatistics(
        for transactions: [Transaction]
    ) -> LedgerStatistics {
        let income = transactions
            .filter { $0.amount > 0 }
            .reduce(Decimal(0)) { $0 + $1.amount }
        
        let expenses = transactions
            .filter { $0.amount < 0 }
            .reduce(Decimal(0)) { $0 + abs($1.amount) }
        
        let net = income - expenses
        
        return LedgerStatistics(
            totalTransactions: transactions.count,
            totalIncome: income,
            totalExpenses: expenses,
            net: net
        )
    }
}
```

#### `Ledgers/Domain/Services/LedgerMonthlyStatisticsService.swift`

```swift
// Ledgers/Domain/Services/LedgerMonthlyStatisticsService.swift
import Foundation

/// Monthly statistics for a ledger
public struct LedgerMonthlyStatistics: Identifiable, Hashable {
    public let id: UUID
    public let year: Int
    public let month: Int
    public let income: Decimal
    public let expenses: Decimal
    public var net: Decimal { income - expenses }
    
    public init(
        id: UUID = UUID(),
        year: Int,
        month: Int,
        income: Decimal,
        expenses: Decimal
    ) {
        self.id = id
        self.year = year
        self.month = month
        self.income = income
        self.expenses = expenses
    }
}

/// Service for calculating monthly statistics
public struct LedgerMonthlyStatisticsService {
    
    /// Calculate monthly statistics for transactions
    /// - Parameter transactions: Filtered transactions for the ledger
    /// - Returns: Array of monthly statistics, sorted by year and month
    public static func calculateMonthlyStatistics(
        for transactions: [Transaction]
    ) -> [LedgerMonthlyStatistics] {
        let calendar = Calendar.current
        
        // Group transactions by year and month
        let grouped = Dictionary(grouping: transactions) { tx in
            (year: calendar.component(.year, from: tx.date),
             month: calendar.component(.month, from: tx.date))
        }
        
        // Calculate statistics for each month
        let monthlyStats = grouped.map { (key, txs) -> LedgerMonthlyStatistics in
            let income = txs
                .filter { $0.amount > 0 }
                .reduce(Decimal(0)) { $0 + $1.amount }
            
            let expenses = txs
                .filter { $0.amount < 0 }
                .reduce(Decimal(0)) { $0 + abs($1.amount) }
            
            return LedgerMonthlyStatistics(
                year: key.year,
                month: key.month,
                income: income,
                expenses: expenses
            )
        }
        
        // Sort by year and month (newest first)
        return monthlyStats.sorted { (a, b) in
            if a.year != b.year {
                return a.year > b.year
            }
            return a.month > b.month
        }
    }
    
    /// Get transactions for a specific month
    /// - Parameters:
    ///   - transactions: All transactions
    ///   - year: Year
    ///   - month: Month (1-12)
    /// - Returns: Filtered transactions for the specified month
    public static func transactionsForMonth(
        _ transactions: [Transaction],
        year: Int,
        month: Int
    ) -> [Transaction] {
        let calendar = Calendar.current
        return transactions.filter { tx in
            let txYear = calendar.component(.year, from: tx.date)
            let txMonth = calendar.component(.month, from: tx.date)
            return txYear == year && txMonth == month
        }
    }
}
```

---

## Layer 2: Data Layer (`Ledgers/Data/`)

### Purpose
Handle ledger persistence. **Only depends on Domain layer.**

### Files

#### `Ledgers/Data/Protocols/LedgerRepositoryProtocol.swift`

```swift
// Ledgers/Data/Protocols/LedgerRepositoryProtocol.swift
import Foundation

/// Protocol for ledger data access
/// Provides abstraction for ledger persistence and queries
public protocol LedgerRepositoryProtocol {
    // Basic CRUD
    func findAll() async throws -> [Ledger]
    func save(_ ledger: Ledger) async throws
    func save(_ ledgers: [Ledger]) async throws
    func delete(_ ledger: Ledger) async throws
    
    // Query methods
    func findById(_ id: UUID) async throws -> Ledger?
    func findDefault() async throws -> Ledger?
}
```

#### `Ledgers/Data/Repositories/FileLedgerRepository.swift`

```swift
// Ledgers/Data/Repositories/FileLedgerRepository.swift
import Foundation

/// File-based ledger repository implementation
/// Uses JSON file storage: "ledgers.json"
public actor FileLedgerRepository: LedgerRepositoryProtocol {
    private let filePersistence: FilePersistenceProtocol
    private let fileName = "ledgers.json"
    
    public init(filePersistence: FilePersistenceProtocol) {
        self.filePersistence = filePersistence
    }
    
    public func findAll() async throws -> [Ledger] {
        guard let data = try? await filePersistence.read(fileName) else {
            return []
        }
        return try JSONDecoder().decode([Ledger].self, from: data)
    }
    
    public func save(_ ledger: Ledger) async throws {
        var ledgers = try await findAll()
        if let index = ledgers.firstIndex(where: { $0.id == ledger.id }) {
            ledgers[index] = ledger
        } else {
            ledgers.append(ledger)
        }
        try await save(ledgers)
    }
    
    public func save(_ ledgers: [Ledger]) async throws {
        let data = try JSONEncoder().encode(ledgers)
        try await filePersistence.write(fileName, data: data)
    }
    
    public func delete(_ ledger: Ledger) async throws {
        var ledgers = try await findAll()
        ledgers.removeAll { $0.id == ledger.id }
        try await save(ledgers)
    }
    
    public func findById(_ id: UUID) async throws -> Ledger? {
        let ledgers = try await findAll()
        return ledgers.first { $0.id == id }
    }
    
    public func findDefault() async throws -> Ledger? {
        let ledgers = try await findAll()
        return ledgers.first { $0.isDefault }
    }
}
```

---

## Layer 3: Services Layer (`Ledgers/Services/`)

### Purpose
Orchestrate ledger business logic. **Depends on Domain and Data layers only.**

### Files

#### `Ledgers/Services/LedgerService.swift`

```swift
// Ledgers/Services/LedgerService.swift
import Foundation

/// Main service for ledger management
/// Orchestrates ledger operations and provides high-level API
@MainActor
public class LedgerService: ObservableObject {
    @Published public private(set) var ledgers: [Ledger] = []
    
    private let repository: LedgerRepositoryProtocol
    private let transactionsProvider: () -> [Transaction]
    private let accountsProvider: () -> [Account]
    
    public init(
        repository: LedgerRepositoryProtocol,
        transactionsProvider: @escaping () -> [Transaction],
        accountsProvider: @escaping () -> [Account]
    ) {
        self.repository = repository
        self.transactionsProvider = transactionsProvider
        self.accountsProvider = accountsProvider
    }
    
    // MARK: - Load & Save
    
    public func load() async throws {
        var loadedLedgers = try await repository.findAll()
        
        // Ensure default ledger exists
        if loadedLedgers.first(where: { $0.isDefault }) == nil {
            let defaultLedger = Ledger(
                name: "核心账本",
                type: .core,
                isDefault: true,
                rules: .coreLedgerRules
            )
            loadedLedgers.append(defaultLedger)
            try await repository.save(defaultLedger)
        }
        
        ledgers = loadedLedgers
    }
    
    // MARK: - CRUD Operations
    
    public func addLedger(_ ledger: Ledger) async throws {
        // Ensure only one default ledger
        if ledger.isDefault {
            // Remove default flag from other ledgers
            for i in ledgers.indices {
                if ledgers[i].isDefault {
                    var updated = ledgers[i]
                    updated.isDefault = false
                    ledgers[i] = updated
                    try await repository.save(updated)
                }
            }
        }
        
        ledgers.append(ledger)
        try await repository.save(ledger)
    }
    
    public func updateLedger(_ ledger: Ledger) async throws {
        guard let index = ledgers.firstIndex(where: { $0.id == ledger.id }) else {
            throw LedgerError.ledgerNotFound
        }
        
        // Handle default ledger constraint
        if ledger.isDefault {
            for i in ledgers.indices where i != index {
                if ledgers[i].isDefault {
                    var updated = ledgers[i]
                    updated.isDefault = false
                    ledgers[i] = updated
                    try await repository.save(updated)
                }
            }
        }
        
        var updated = ledger
        updated.updatedAt = Date()
        ledgers[index] = updated
        try await repository.save(updated)
    }
    
    public func deleteLedger(_ ledger: Ledger) async throws {
        // Prevent deleting default ledger
        guard !ledger.isDefault else {
            throw LedgerError.cannotDeleteDefault
        }
        
        ledgers.removeAll { $0.id == ledger.id }
        try await repository.delete(ledger)
    }
    
    // MARK: - Query Operations
    
    /// Get transactions filtered by ledger rules
    /// - Parameter categoryMappings: Optional category mappings for category-based filtering
    public func transactions(
        for ledger: Ledger,
        categoryMappings: [LedgerTransactionCategory]? = nil
    ) -> [Transaction] {
        let allTransactions = transactionsProvider()
        let allAccounts = accountsProvider()
        var filtered = LedgerFilterService.filterTransactions(
            allTransactions,
            accounts: allAccounts,
            rules: ledger.rules
        )
        
        // Apply category filtering if rules specify categories
        if let mappings = categoryMappings {
            // Include categories
            if let includeCategories = ledger.rules.includeCategories, !includeCategories.isEmpty {
                let includedTransactionIds = Set(
                    mappings.filter { includeCategories.contains($0.categoryId) }
                        .map { $0.transactionId }
                )
                filtered = filtered.filter { includedTransactionIds.contains($0.id) }
            }
            
            // Exclude categories
            if let excludeCategories = ledger.rules.excludeCategories, !excludeCategories.isEmpty {
                let excludedTransactionIds = Set(
                    mappings.filter { excludeCategories.contains($0.categoryId) }
                        .map { $0.transactionId }
                )
                filtered = filtered.filter { !excludedTransactionIds.contains($0.id) }
            }
            
            // Include tags
            if let includeTags = ledger.rules.includeTags, !includeTags.isEmpty {
                let includedTransactionIds = Set(
                    mappings.filter { !$0.tags.isDisjoint(with: includeTags) }
                        .map { $0.transactionId }
                )
                filtered = filtered.filter { includedTransactionIds.contains($0.id) }
            }
            
            // Exclude tags
            if let excludeTags = ledger.rules.excludeTags, !excludeTags.isEmpty {
                let excludedTransactionIds = Set(
                    mappings.filter { !$0.tags.isDisjoint(with: excludeTags) }
                        .map { $0.transactionId }
                )
                filtered = filtered.filter { !excludedTransactionIds.contains($0.id) }
            }
        }
        
        return filtered
    }
    
    /// Calculate statistics for a ledger
    public func statistics(for ledger: Ledger) -> LedgerStatistics {
        let filteredTransactions = transactions(for: ledger)
        return LedgerStatisticsService.calculateStatistics(for: filteredTransactions)
    }
    
    /// Get monthly statistics for a ledger
    public func monthlyStatistics(for ledger: Ledger) -> [LedgerMonthlyStatistics] {
        let filteredTransactions = transactions(for: ledger)
        return LedgerMonthlyStatisticsService.calculateMonthlyStatistics(for: filteredTransactions)
    }
    
    /// Get transactions for a specific month in a ledger
    public func transactions(for ledger: Ledger, year: Int, month: Int) -> [Transaction] {
        let filteredTransactions = transactions(for: ledger)
        return LedgerMonthlyStatisticsService.transactionsForMonth(
            filteredTransactions,
            year: year,
            month: month
        )
    }
    
    /// Calculate balance statistics for a ledger
    /// Returns nil if balance tracking is disabled for this ledger
    public func balanceStatistics(for ledger: Ledger) -> LedgerBalanceStatistics? {
        let filteredTransactions = transactions(for: ledger)
        let allAccounts = accountsProvider()
        return LedgerBalanceService.calculateBalances(
            transactions: filteredTransactions,
            accounts: allAccounts,
            rules: ledger.rules
        )
    }
    
    /// Calculate historical balance at a specific date
    public func historicalBalance(
        for ledger: Ledger,
        at date: Date
    ) -> HistoricalBalance? {
        let allTransactions = transactionsProvider()
        let allAccounts = accountsProvider()
        return LedgerBalanceService.calculateHistoricalBalance(
            transactions: allTransactions,
            accounts: allAccounts,
            rules: ledger.rules,
            targetDate: date
        )
    }
    
    /// Calculate historical account balances at a specific date (for core ledger only)
    public func historicalAccountBalances(
        for ledger: Ledger,
        at date: Date
    ) -> [UUID: HistoricalBalance]? {
        // Only applicable for account-based mode (core ledger)
        guard ledger.rules.trackBalances,
              ledger.rules.balanceTrackingMode == .accountBased else {
            return nil
        }
        
        let allTransactions = transactionsProvider()
        let allAccounts = accountsProvider()
        return LedgerBalanceService.calculateHistoricalAccountBalances(
            transactions: allTransactions,
            accounts: allAccounts,
            targetDate: date
        )
    }
}

// MARK: - Errors

public enum LedgerError: LocalizedError {
    case ledgerNotFound
    case cannotDeleteDefault
    case invalidConfiguration(String)
    
    public var errorDescription: String? {
        switch self {
        case .ledgerNotFound:
            return "账本未找到"
        case .cannotDeleteDefault:
            return "无法删除默认账本"
        case .invalidConfiguration(let message):
            return message
        }
    }
}
```

---

## Layer 4: UI Layer (`Ledgers/UI/`)

### Purpose
UI components for ledger management. **Depends on Services layer only.**

### Files

#### `Ledgers/UI/AccountBookView.swift`

```swift
// Ledgers/UI/AccountBookView.swift
import SwiftUI

/// Main container view for Account Book feature
public struct AccountBookView: View {
    @StateObject private var viewModel: AccountBookViewModel
    
    public init(viewModel: AccountBookViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }
    
    public var body: some View {
        HSplitView {
            // Left Pane: Ledger List
            LedgerListPane(
                ledgers: viewModel.ledgers,
                selectedLedger: $viewModel.selectedLedger,
                onAdd: { viewModel.showAddLedger = true },
                onDelete: { ledger in
                    Task {
                        await viewModel.deleteLedger(ledger)
                    }
                }
            )
            .frame(minWidth: 280, idealWidth: 300, maxWidth: 350)
            
            // Right Pane: Ledger Detail
            if let ledger = viewModel.selectedLedger {
                LedgerDetailPane(
                    ledger: ledger,
                    transactions: viewModel.transactions(for: ledger),
                    statistics: viewModel.statistics(for: ledger),
                    balanceStatistics: viewModel.balanceStatistics(for: ledger),
                    monthlyStatistics: viewModel.monthlyStatistics(for: ledger),
                    selectedYear: $viewModel.selectedYear,
                    selectedMonth: $viewModel.selectedMonth,
                    onSelectMonth: { month in
                        viewModel.selectMonth(month)
                    },
                    onSelectYear: { year in
                        viewModel.selectYear(year)
                    }
                )
            } else {
                EmptyLedgerState()
            }
        }
        .sheet(isPresented: $viewModel.showAddLedger) {
            AddLedgerView(
                onSave: { ledger in
                    Task {
                        await viewModel.addLedger(ledger)
                    }
                }
            )
        }
        .task {
            await viewModel.load()
        }
    }
}
```

#### `Ledgers/UI/ViewModels/AccountBookViewModel.swift`

```swift
// Ledgers/UI/ViewModels/AccountBookViewModel.swift
import SwiftUI

/// View model for Account Book view
@MainActor
public class AccountBookViewModel: ObservableObject {
    @Published var selectedLedger: Ledger?
    @Published var showAddLedger = false
    @Published var selectedYear: Int = Calendar.current.component(.year, from: Date())
    @Published var selectedMonth: Int? = nil  // nil means show all transactions
    
    private let ledgerService: LedgerService
    
    public var ledgers: [Ledger] {
        ledgerService.ledgers
    }
    
    public init(ledgerService: LedgerService) {
        self.ledgerService = ledgerService
    }
    
    public func load() async {
        do {
            try await ledgerService.load()
            // Select default ledger if none selected
            if selectedLedger == nil {
                selectedLedger = ledgers.first { $0.isDefault }
            }
        } catch {
            // Handle error
            print("Failed to load ledgers: \(error)")
        }
    }
    
    public func addLedger(_ ledger: Ledger) async {
        do {
            try await ledgerService.addLedger(ledger)
            selectedLedger = ledger
            showAddLedger = false
        } catch {
            // Handle error
            print("Failed to add ledger: \(error)")
        }
    }
    
    public func deleteLedger(_ ledger: Ledger) async {
        do {
            try await ledgerService.deleteLedger(ledger)
            if selectedLedger?.id == ledger.id {
                selectedLedger = ledgers.first { $0.isDefault }
                selectedMonth = nil  // Reset month selection
            }
        } catch {
            // Handle error
            print("Failed to delete ledger: \(error)")
        }
    }
    
    public func transactions(for ledger: Ledger) -> [Transaction] {
        if let month = selectedMonth {
            return ledgerService.transactions(for: ledger, year: selectedYear, month: month)
        } else {
            return ledgerService.transactions(for: ledger)
        }
    }
    
    public func statistics(for ledger: Ledger) -> LedgerStatistics {
        ledgerService.statistics(for: ledger)
    }
    
    public func monthlyStatistics(for ledger: Ledger) -> [LedgerMonthlyStatistics] {
        ledgerService.monthlyStatistics(for: ledger)
    }
    
    public func balanceStatistics(for ledger: Ledger) -> LedgerBalanceStatistics? {
        ledgerService.balanceStatistics(for: ledger)
    }
    
    public func selectMonth(_ month: Int?) {
        selectedMonth = month
    }
    
    public func selectYear(_ year: Int) {
        selectedYear = year
        selectedMonth = nil  // Reset month when year changes
    }
}
```

#### `Ledgers/UI/Components/LedgerListPane.swift`

```swift
// Ledgers/UI/Components/LedgerListPane.swift
import SwiftUI

struct LedgerListPane: View {
    let ledgers: [Ledger]
    @Binding var selectedLedger: Ledger?
    var onAdd: () -> Void
    var onDelete: (Ledger) -> Void
    
    var body: some View {
        ScrollView {
            VStack(spacing: 12) {
                // Header
                HStack {
                    Text("账本")
                        .font(.title2)
                        .bold()
                    Spacer()
                    Button(action: onAdd) {
                        Image(systemName: "plus.circle.fill")
                            .font(.title2)
                    }
                }
                .padding(.horizontal, 16)
                .padding(.top, 16)
                
                // Ledger list
                VStack(spacing: 8) {
                    ForEach(ledgers) { ledger in
                        LedgerRow(
                            ledger: ledger,
                            isSelected: selectedLedger?.id == ledger.id,
                            onSelect: { selectedLedger = ledger },
                            onDelete: { onDelete(ledger) }
                        )
                    }
                }
                .padding(.horizontal, 16)
            }
        }
    }
}
```

#### `Ledgers/UI/Components/LedgerDetailPane.swift`

```swift
// Ledgers/UI/Components/LedgerDetailPane.swift
import SwiftUI

/// Detail pane for selected ledger
/// Shows: header statistics, month card selector, and transaction list
struct LedgerDetailPane: View {
    let ledger: Ledger
    let transactions: [Transaction]
    let statistics: LedgerStatistics
    let balanceStatistics: LedgerBalanceStatistics?
    let monthlyStatistics: [LedgerMonthlyStatistics]
    @Binding var selectedYear: Int
    @Binding var selectedMonth: Int?
    var onSelectMonth: (Int?) -> Void
    var onSelectYear: (Int) -> Void
    
    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                // Header with statistics
                LedgerHeaderCard(
                    ledger: ledger,
                    statistics: statistics,
                    balanceStatistics: balanceStatistics
                )
                .padding(.horizontal, 16)
                .padding(.top, 16)
                
                // Month card selector
                MonthCardSelector(
                    year: selectedYear,
                    selectedMonth: $selectedMonth,
                    monthlyStatistics: monthlyStatistics,
                    onSelectMonth: onSelectMonth,
                    onSelectYear: onSelectYear
                )
                .padding(.horizontal, 16)
                
                // Transaction list (all transactions, no account grouping)
                LedgerTransactionList(
                    transactions: transactions
                )
                .padding(.horizontal, 16)
                .padding(.bottom, 16)
            }
        }
    }
}
```

#### `Ledgers/UI/Components/MonthCardSelector.swift`

```swift
// Ledgers/UI/Components/MonthCardSelector.swift
import SwiftUI

/// Month card selector component (as shown in the image)
/// Displays 12 month cards in a grid with year navigation
struct MonthCardSelector: View {
    let year: Int
    @Binding var selectedMonth: Int?
    let monthlyStatistics: [LedgerMonthlyStatistics]
    var onSelectMonth: (Int?) -> Void
    var onSelectYear: (Int) -> Void
    
    var body: some View {
        VStack(spacing: 16) {
            // Year header with navigation
            HStack {
                Button(action: { onSelectYear(year - 1) }) {
                    Image(systemName: "chevron.left.circle.fill")
                        .font(.title2)
                        .foregroundColor(.blue)
                }
                
                Spacer()
                
                Text("\(year)年")
                    .font(.title)
                    .bold()
                
                Spacer()
                
                Button(action: { onSelectYear(year + 1) }) {
                    Image(systemName: "chevron.right.circle.fill")
                        .font(.title2)
                        .foregroundColor(.blue)
                }
            }
            .padding(.horizontal, 8)
            
            // Month cards grid (2 rows x 6 columns)
            LazyVGrid(columns: Array(repeating: GridItem(.flexible(), spacing: 8), count: 6), spacing: 8) {
                ForEach(1...12, id: \.self) { month in
                    MonthCard(
                        month: month,
                        year: year,
                        isSelected: selectedMonth == month,
                        statistics: statisticsForMonth(year: year, month: month),
                        onSelect: {
                            if selectedMonth == month {
                                // Deselect if already selected
                                selectedMonth = nil
                                onSelectMonth(nil)
                            } else {
                                selectedMonth = month
                                onSelectMonth(month)
                            }
                        }
                    )
                }
            }
        }
        .padding(16)
        .background(RoundedRectangle(cornerRadius: 12).fill(Color.appSecondaryBackground))
    }
    
    private func statisticsForMonth(year: Int, month: Int) -> LedgerMonthlyStatistics? {
        monthlyStatistics.first { $0.year == year && $0.month == month }
    }
}
```

#### `Ledgers/UI/Components/MonthCard.swift`

```swift
// Ledgers/UI/Components/MonthCard.swift
import SwiftUI

/// Individual month card component
struct MonthCard: View {
    let month: Int
    let year: Int
    let isSelected: Bool
    let statistics: LedgerMonthlyStatistics?
    var onSelect: () -> Void
    
    private var monthName: String {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "zh_CN")
        formatter.dateFormat = "M月"
        let date = Calendar.current.date(from: DateComponents(year: year, month: month, day: 1))!
        return formatter.string(from: date)
    }
    
    var body: some View {
        Button(action: onSelect) {
            VStack(spacing: 8) {
                // Month label
                Text(monthName)
                    .font(.headline)
                    .foregroundColor(isSelected ? .white : .primary)
                
                // Statistics (if available)
                if let stats = statistics {
                    VStack(spacing: 4) {
                        // Expenses (red, negative)
                        Text(formatAmount(-stats.expenses))
                            .font(.caption)
                            .foregroundColor(isSelected ? .white.opacity(0.9) : .red)
                        
                        // Income (green, positive)
                        Text(formatAmount(stats.income))
                            .font(.caption)
                            .foregroundColor(isSelected ? .white.opacity(0.9) : .green)
                    }
                } else {
                    // No data indicator
                    Text("—")
                        .font(.caption)
                        .foregroundColor(isSelected ? .white.opacity(0.7) : .secondary)
                }
            }
            .frame(maxWidth: .infinity)
            .padding(.vertical, 12)
            .padding(.horizontal, 8)
            .background(
                RoundedRectangle(cornerRadius: 8)
                    .fill(isSelected ? Color.orange : Color.appTertiaryBackground)
            )
            .overlay(
                RoundedRectangle(cornerRadius: 8)
                    .stroke(isSelected ? Color.orange : Color.appSeparator, lineWidth: isSelected ? 2 : 1)
            )
        }
        .buttonStyle(.plain)
    }
    
    private func formatAmount(_ amount: Decimal) -> String {
        let formatter = NumberFormatter()
        formatter.numberStyle = .decimal
        formatter.maximumFractionDigits = 2
        formatter.minimumFractionDigits = 0
        
        let absAmount = abs(amount)
        let value = NSDecimalNumber(decimal: absAmount)
        
        // Format with "w" (万) suffix for amounts >= 10,000
        if absAmount >= 10000 {
            let wan = absAmount / 10000
            if let formatted = formatter.string(from: NSDecimalNumber(decimal: wan)) {
                return "\(amount < 0 ? "-" : "+")\(formatted)w"
            }
        }
        
        // Regular formatting for smaller amounts
        if let formatted = formatter.string(from: value) {
            return "\(amount < 0 ? "-" : "+")\(formatted)"
        }
        
        return "\(amount)"
    }
}
```

#### `Ledgers/Domain/Models/LedgerCategorySystem.swift`

```swift
// Ledgers/Domain/Models/LedgerCategorySystem.swift
import Foundation

/// Category definition for a ledger
public struct LedgerCategory: Identifiable, Codable, Hashable {
    public let id: UUID
    public let ledgerId: UUID
    public let name: String
    public let color: String?  // Hex color code
    public let icon: String?  // SF Symbol name
    public let parentId: UUID?  // For hierarchical categories
    public let createdAt: Date
    public let updatedAt: Date
    
    public init(
        id: UUID = UUID(),
        ledgerId: UUID,
        name: String,
        color: String? = nil,
        icon: String? = nil,
        parentId: UUID? = nil,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.ledgerId = ledgerId
        self.name = name
        self.color = color
        self.icon = icon
        self.parentId = parentId
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }
}

/// Transaction-to-category mapping for a ledger
/// Stores which category each transaction belongs to in this ledger
public struct LedgerTransactionCategory: Identifiable, Codable, Hashable {
    public let id: UUID
    public let ledgerId: UUID
    public let transactionId: UUID
    public let categoryId: UUID
    public let tags: Set<String>  // Flexible tags for additional classification
    public let assignedAt: Date
    public let assignedBy: String?  // "auto" or "manual"
    
    public init(
        id: UUID = UUID(),
        ledgerId: UUID,
        transactionId: UUID,
        categoryId: UUID,
        tags: Set<String> = [],
        assignedAt: Date = Date(),
        assignedBy: String? = nil
    ) {
        self.id = id
        self.ledgerId = ledgerId
        self.transactionId = transactionId
        self.categoryId = categoryId
        self.tags = tags
        self.assignedAt = assignedAt
        self.assignedBy = assignedBy
    }
}

/// Category rule for automatic classification
public struct LedgerCategoryRule: Identifiable, Codable, Hashable {
    public let id: UUID
    public let ledgerId: UUID
    public let categoryId: UUID
    public let priority: Int  // Higher priority rules are evaluated first
    public let conditions: CategoryRuleConditions
    public let createdAt: Date
    public let updatedAt: Date
    
    public init(
        id: UUID = UUID(),
        ledgerId: UUID,
        categoryId: UUID,
        priority: Int = 0,
        conditions: CategoryRuleConditions,
        createdAt: Date = Date(),
        updatedAt: Date = Date()
    ) {
        self.id = id
        self.ledgerId = ledgerId
        self.categoryId = categoryId
        self.priority = priority
        self.conditions = conditions
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }
}

/// Conditions for automatic category assignment
public struct CategoryRuleConditions: Codable, Hashable {
    /// Match merchant name (contains, equals, starts with, etc.)
    public var merchantPattern: String?
    public var merchantMatchType: PatternMatchType
    
    /// Match notes/description
    public var notesPattern: String?
    public var notesMatchType: PatternMatchType
    
    /// Match account
    public var accountIds: Set<UUID>?
    
    /// Match amount range
    public var minAmount: Decimal?
    public var maxAmount: Decimal?
    
    /// Match transaction type
    public var isIncome: Bool?
    public var isExpense: Bool?
    
    public init(
        merchantPattern: String? = nil,
        merchantMatchType: PatternMatchType = .contains,
        notesPattern: String? = nil,
        notesMatchType: PatternMatchType = .contains,
        accountIds: Set<UUID>? = nil,
        minAmount: Decimal? = nil,
        maxAmount: Decimal? = nil,
        isIncome: Bool? = nil,
        isExpense: Bool? = nil
    ) {
        self.merchantPattern = merchantPattern
        self.merchantMatchType = merchantMatchType
        self.notesPattern = notesPattern
        self.notesMatchType = notesMatchType
        self.accountIds = accountIds
        self.minAmount = minAmount
        self.maxAmount = maxAmount
        self.isIncome = isIncome
        self.isExpense = isExpense
    }
}

public enum PatternMatchType: String, Codable {
    case contains
    case equals
    case startsWith
    case endsWith
    case regex
}
```

#### `Ledgers/Domain/Services/LedgerCategoryService.swift`

```swift
// Ledgers/Domain/Services/LedgerCategoryService.swift
import Foundation

/// Service for category management and classification
public struct LedgerCategoryService {
    
    /// Classify a transaction using ledger's category rules
    /// - Parameters:
    ///   - transaction: Transaction to classify
    ///   - ledger: Ledger to classify for
    ///   - categories: All categories for this ledger
    ///   - rules: All category rules for this ledger (sorted by priority)
    /// - Returns: Category ID if matched, nil otherwise
    public static func classifyTransaction(
        _ transaction: Transaction,
        for ledger: Ledger,
        categories: [LedgerCategory],
        rules: [LedgerCategoryRule]
    ) -> UUID? {
        // Sort rules by priority (higher first)
        let sortedRules = rules.sorted { $0.priority > $1.priority }
        
        // Try each rule in priority order
        for rule in sortedRules {
            if matchesConditions(transaction: transaction, conditions: rule.conditions) {
                return rule.categoryId
            }
        }
        
        // No rule matched
        return nil
    }
    
    /// Check if transaction matches rule conditions
    private static func matchesConditions(
        transaction: Transaction,
        conditions: CategoryRuleConditions
    ) -> Bool {
        // Merchant pattern matching
        if let pattern = conditions.merchantPattern {
            if !matchesPattern(
                text: transaction.merchant,
                pattern: pattern,
                matchType: conditions.merchantMatchType
            ) {
                return false
            }
        }
        
        // Notes pattern matching
        if let pattern = conditions.notesPattern, let notes = transaction.notes {
            if !matchesPattern(
                text: notes,
                pattern: pattern,
                matchType: conditions.notesMatchType
            ) {
                return false
            }
        }
        
        // Account matching
        if let accountIds = conditions.accountIds {
            if !accountIds.contains(transaction.accountId) {
                return false
            }
        }
        
        // Amount range matching
        if let minAmount = conditions.minAmount {
            if abs(transaction.amount) < minAmount {
                return false
            }
        }
        if let maxAmount = conditions.maxAmount {
            if abs(transaction.amount) > maxAmount {
                return false
            }
        }
        
        // Transaction type matching
        if let isIncome = conditions.isIncome {
            if (transaction.amount > 0) != isIncome {
                return false
            }
        }
        if let isExpense = conditions.isExpense {
            if (transaction.amount < 0) != isExpense {
                return false
            }
        }
        
        return true
    }
    
    /// Match text against pattern
    private static func matchesPattern(
        text: String,
        pattern: String,
        matchType: PatternMatchType
    ) -> Bool {
        switch matchType {
        case .contains:
            return text.localizedCaseInsensitiveContains(pattern)
        case .equals:
            return text.localizedCaseInsensitiveCompare(pattern) == .orderedSame
        case .startsWith:
            return text.localizedCaseInsensitiveHasPrefix(pattern)
        case .endsWith:
            return text.localizedCaseInsensitiveHasSuffix(pattern)
        case .regex:
            // TODO: Implement regex matching
            return text.range(of: pattern, options: .regularExpression) != nil
        }
    }
    
    /// Get category for a transaction in a ledger
    /// - Parameters:
    ///   - transactionId: Transaction ID
    ///   - ledgerId: Ledger ID
    ///   - mappings: All transaction-category mappings
    /// - Returns: Category ID if exists, nil otherwise
    public static func getCategory(
        for transactionId: UUID,
        in ledgerId: UUID,
        mappings: [LedgerTransactionCategory]
    ) -> UUID? {
        return mappings.first { 
            $0.transactionId == transactionId && $0.ledgerId == ledgerId 
        }?.categoryId
    }
    
    /// Get tags for a transaction in a ledger
    public static func getTags(
        for transactionId: UUID,
        in ledgerId: UUID,
        mappings: [LedgerTransactionCategory]
    ) -> Set<String> {
        return mappings.first { 
            $0.transactionId == transactionId && $0.ledgerId == ledgerId 
        }?.tags ?? []
    }
}
```

#### `Ledgers/Data/Protocols/LedgerCategoryRepositoryProtocol.swift`

```swift
// Ledgers/Data/Protocols/LedgerCategoryRepositoryProtocol.swift
import Foundation

/// Protocol for ledger category data access
public protocol LedgerCategoryRepositoryProtocol {
    // Category CRUD
    func findAllCategories(for ledgerId: UUID) async throws -> [LedgerCategory]
    func saveCategory(_ category: LedgerCategory) async throws
    func deleteCategory(_ category: LedgerCategory) async throws
    
    // Transaction-Category Mapping CRUD
    func findMapping(for transactionId: UUID, in ledgerId: UUID) async throws -> LedgerTransactionCategory?
    func findAllMappings(for ledgerId: UUID) async throws -> [LedgerTransactionCategory]
    func saveMapping(_ mapping: LedgerTransactionCategory) async throws
    func deleteMapping(_ mapping: LedgerTransactionCategory) async throws
    func deleteMappings(for transactionId: UUID) async throws  // Delete all mappings for a transaction
    
    // Category Rules CRUD
    func findAllRules(for ledgerId: UUID) async throws -> [LedgerCategoryRule]
    func saveRule(_ rule: LedgerCategoryRule) async throws
    func deleteRule(_ rule: LedgerCategoryRule) async throws
}
```

#### `Ledgers/Data/Repositories/FileLedgerCategoryRepository.swift`

```swift
// Ledgers/Data/Repositories/FileLedgerCategoryRepository.swift
import Foundation

/// File-based ledger category repository implementation
/// Uses JSON file storage: "ledger_categories_{ledgerId}.json", "ledger_mappings_{ledgerId}.json", "ledger_rules_{ledgerId}.json"
public actor FileLedgerCategoryRepository: LedgerCategoryRepositoryProtocol {
    private let filePersistence: FilePersistenceProtocol
    
    public init(filePersistence: FilePersistenceProtocol) {
        self.filePersistence = filePersistence
    }
    
    // MARK: - Category CRUD
    
    public func findAllCategories(for ledgerId: UUID) async throws -> [LedgerCategory] {
        let fileName = "ledger_categories_\(ledgerId.uuidString).json"
        guard let data = try? await filePersistence.read(fileName) else {
            return []
        }
        return try JSONDecoder().decode([LedgerCategory].self, from: data)
    }
    
    public func saveCategory(_ category: LedgerCategory) async throws {
        var categories = try await findAllCategories(for: category.ledgerId)
        if let index = categories.firstIndex(where: { $0.id == category.id }) {
            categories[index] = category
        } else {
            categories.append(category)
        }
        try await saveCategories(categories, for: category.ledgerId)
    }
    
    public func deleteCategory(_ category: LedgerCategory) async throws {
        var categories = try await findAllCategories(for: category.ledgerId)
        categories.removeAll { $0.id == category.id }
        try await saveCategories(categories, for: category.ledgerId)
    }
    
    private func saveCategories(_ categories: [LedgerCategory], for ledgerId: UUID) async throws {
        let fileName = "ledger_categories_\(ledgerId.uuidString).json"
        let data = try JSONEncoder().encode(categories)
        try await filePersistence.write(fileName, data: data)
    }
    
    // MARK: - Mapping CRUD
    
    public func findMapping(for transactionId: UUID, in ledgerId: UUID) async throws -> LedgerTransactionCategory? {
        let mappings = try await findAllMappings(for: ledgerId)
        return mappings.first { $0.transactionId == transactionId }
    }
    
    public func findAllMappings(for ledgerId: UUID) async throws -> [LedgerTransactionCategory] {
        let fileName = "ledger_mappings_\(ledgerId.uuidString).json"
        guard let data = try? await filePersistence.read(fileName) else {
            return []
        }
        return try JSONDecoder().decode([LedgerTransactionCategory].self, from: data)
    }
    
    public func saveMapping(_ mapping: LedgerTransactionCategory) async throws {
        var mappings = try await findAllMappings(for: mapping.ledgerId)
        if let index = mappings.firstIndex(where: { $0.id == mapping.id }) {
            mappings[index] = mapping
        } else {
            // Remove existing mapping for this transaction if exists
            mappings.removeAll { $0.transactionId == mapping.transactionId }
            mappings.append(mapping)
        }
        try await saveMappings(mappings, for: mapping.ledgerId)
    }
    
    public func deleteMapping(_ mapping: LedgerTransactionCategory) async throws {
        var mappings = try await findAllMappings(for: mapping.ledgerId)
        mappings.removeAll { $0.id == mapping.id }
        try await saveMappings(mappings, for: mapping.ledgerId)
    }
    
    public func deleteMappings(for transactionId: UUID) async throws {
        // Find all ledgers that have mappings for this transaction
        // This requires scanning all ledger mapping files
        // For now, this is a placeholder - implementation depends on how we store mappings
        // TODO: Implement efficient deletion across all ledgers
    }
    
    private func saveMappings(_ mappings: [LedgerTransactionCategory], for ledgerId: UUID) async throws {
        let fileName = "ledger_mappings_\(ledgerId.uuidString).json"
        let data = try JSONEncoder().encode(mappings)
        try await filePersistence.write(fileName, data: data)
    }
    
    // MARK: - Rules CRUD
    
    public func findAllRules(for ledgerId: UUID) async throws -> [LedgerCategoryRule] {
        let fileName = "ledger_rules_\(ledgerId.uuidString).json"
        guard let data = try? await filePersistence.read(fileName) else {
            return []
        }
        return try JSONDecoder().decode([LedgerCategoryRule].self, from: data)
    }
    
    public func saveRule(_ rule: LedgerCategoryRule) async throws {
        var rules = try await findAllRules(for: rule.ledgerId)
        if let index = rules.firstIndex(where: { $0.id == rule.id }) {
            rules[index] = rule
        } else {
            rules.append(rule)
        }
        try await saveRules(rules, for: rule.ledgerId)
    }
    
    public func deleteRule(_ rule: LedgerCategoryRule) async throws {
        var rules = try await findAllRules(for: rule.ledgerId)
        rules.removeAll { $0.id == rule.id }
        try await saveRules(rules, for: rule.ledgerId)
    }
    
    private func saveRules(_ rules: [LedgerCategoryRule], for ledgerId: UUID) async throws {
        let fileName = "ledger_rules_\(ledgerId.uuidString).json"
        let data = try JSONEncoder().encode(rules)
        try await filePersistence.write(fileName, data: data)
    }
}
```

#### `Ledgers/UI/Components/LedgerTransactionList.swift`

```swift
// Ledgers/UI/Components/LedgerTransactionList.swift
import SwiftUI

/// Transaction list for ledger (no account grouping, just list all transactions)
struct LedgerTransactionList: View {
    let transactions: [Transaction]
    
    var body: some View {
        if transactions.isEmpty {
            EmptyTransactionState()
        } else {
            VStack(alignment: .leading, spacing: 16) {
                // Group by date
                let grouped = Dictionary(grouping: transactions) { tx in
                    Calendar.current.startOfDay(for: tx.date)
                }
                let sortedDays = Array(grouped.keys).sorted(by: >) // newest first
                
                ForEach(sortedDays, id: \.self) { day in
                    if let dayTransactions = grouped[day] {
                        TransactionDayGroup(
                            date: day,
                            transactions: dayTransactions.sorted { tx1, tx2 in
                                if tx1.date != tx2.date {
                                    return tx1.date > tx2.date
                                }
                                return tx1.sequenceNumber > tx2.sequenceNumber
                            },
                            currencyCode: "CNY", // TODO: Support multi-currency
                            institution: nil // No institution grouping in ledger view
                        )
                    }
                }
            }
        }
    }
}

struct EmptyTransactionState: View {
    var body: some View {
        VStack(spacing: 16) {
            Image(systemName: "list.bullet.rectangle")
                .font(.system(size: 48))
                .foregroundColor(.secondary)
            Text("暂无交易")
                .font(.headline)
                .foregroundColor(.secondary)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .padding(40)
    }
}
```

#### `Ledgers/UI/Components/LedgerHeaderCard.swift`

```swift
// Ledgers/UI/Components/LedgerHeaderCard.swift
import SwiftUI

/// Header card showing ledger statistics
struct LedgerHeaderCard: View {
    let ledger: Ledger
    let statistics: LedgerStatistics
    let balanceStatistics: LedgerBalanceStatistics?
    
    var body: some View {
        VStack(spacing: 16) {
            HStack {
                Image(systemName: ledger.type.systemImage)
                    .font(.title)
                Text(ledger.name)
                    .font(.title2)
                    .bold()
                Spacer()
            }
            
            // Income/Expense/Net row
            HStack(spacing: 24) {
                StatCard(
                    title: "收入",
                    value: formatCurrency(statistics.totalIncome),
                    color: .green
                )
                StatCard(
                    title: "支出",
                    value: formatCurrency(statistics.totalExpenses),
                    color: .red
                )
                StatCard(
                    title: "净额",
                    value: formatCurrency(statistics.net),
                    color: statistics.net >= 0 ? .green : .red
                )
            }
            
            // Balance row (only shown if balance tracking is enabled)
            if let balanceStats = balanceStatistics {
                Divider()
                
                VStack(spacing: 12) {
                    // Warning message (if applicable)
                    if let warning = balanceStats.warningMessage {
                        HStack {
                            Image(systemName: "exclamationmark.triangle.fill")
                                .foregroundColor(.orange)
                            Text(warning)
                                .font(.caption)
                                .foregroundColor(.secondary)
                        }
                    }
                    
                    // Balance display
                    if balanceStats.trackingMode == .accountBased {
                        // Show total balance and account breakdown
                        VStack(spacing: 8) {
                            HStack {
                                Text("总余额")
                                    .font(.headline)
                                Spacer()
                                Text(formatCurrency(balanceStats.totalBalance))
                                    .font(.title3)
                                    .bold()
                                    .foregroundColor(balanceStats.totalBalance >= 0 ? .green : .red)
                            }
                            
                            // Account balances (collapsible)
                            AccountBalancesView(accountBalances: balanceStats.accountBalances)
                        }
                    } else {
                        // Single account or fund-based: show single balance
                        HStack {
                            Text(balanceStats.trackingMode == .singleAccount ? "账户余额" : "专项资金余额")
                                .font(.headline)
                            Spacer()
                            Text(formatCurrency(balanceStats.totalBalance))
                                .font(.title3)
                                .bold()
                                .foregroundColor(balanceStats.totalBalance >= 0 ? .green : .red)
                        }
                    }
                }
            }
        }
        .padding(16)
        .background(RoundedRectangle(cornerRadius: 12).fill(Color.appSecondaryBackground))
    }
    
    private func formatCurrency(_ value: Decimal) -> String {
        let formatter = Formatters.currency
        formatter.currencyCode = "CNY" // TODO: Support multi-currency
        return formatter.string(from: NSDecimalNumber(decimal: value)) ?? "\(value)"
    }
}

/// View for displaying account balances (for account-based mode)
struct AccountBalancesView: View {
    let accountBalances: [UUID: Decimal]
    @State private var isExpanded = false
    
    var body: some View {
        if accountBalances.isEmpty {
            EmptyView()
        } else {
            VStack(alignment: .leading, spacing: 8) {
                Button(action: { isExpanded.toggle() }) {
                    HStack {
                        Text("账户明细")
                            .font(.caption)
                            .foregroundColor(.secondary)
                        Spacer()
                        Image(systemName: isExpanded ? "chevron.up" : "chevron.down")
                            .font(.caption)
                            .foregroundColor(.secondary)
                    }
                }
                .buttonStyle(.plain)
                
                if isExpanded {
                    ForEach(Array(accountBalances.keys), id: \.self) { accountId in
                        // TODO: Look up account name from account repository
                        HStack {
                            Text("账户 \(accountId.uuidString.prefix(8))")
                                .font(.caption)
                            Spacer()
                            Text(formatCurrency(accountBalances[accountId] ?? 0))
                                .font(.caption)
                                .foregroundColor((accountBalances[accountId] ?? 0) >= 0 ? .green : .red)
                        }
                    }
                }
            }
        }
    }
    
    private func formatCurrency(_ value: Decimal) -> String {
        let formatter = Formatters.currency
        formatter.currencyCode = "CNY"
        return formatter.string(from: NSDecimalNumber(decimal: value)) ?? "\(value)"
    }
}

struct StatCard: View {
    let title: String
    let value: String
    let color: Color
    
    var body: some View {
        VStack(spacing: 4) {
            Text(title)
                .font(.caption)
                .foregroundColor(.secondary)
            Text(value)
                .font(.headline)
                .foregroundColor(color)
        }
        .frame(maxWidth: .infinity)
    }
}
```

---

## Integration with Core App

### Minimal Changes to Core

#### `Core/Dependencies/AppDependencies.swift`

```swift
// Add to AppDependencies
public struct AppDependencies {
    // ... existing dependencies ...
    
    // Ledger module dependency
    let ledgerRepository: LedgerRepositoryProtocol
    
    static func production() -> AppDependencies {
        // ... existing code ...
        
        return AppDependencies(
            // ... existing parameters ...
            ledgerRepository: FileLedgerRepository(filePersistence: filePersistence)
        )
    }
}
```

#### `Core/State/AppState.swift`

```swift
// Add to AppState (minimal integration)
@MainActor
final class AppState: ObservableObject {
    // ... existing properties ...
    
    // Ledger service (optional - only if needed for cross-feature integration)
    private(set) var ledgerService: LedgerService?
    
    init(dependencies: AppDependencies = .production(), ...) {
        // ... existing init ...
        
        // Initialize ledger service
        ledgerService = LedgerService(
            repository: dependencies.ledgerRepository,
            transactionsProvider: { [weak self] in self?.transactions ?? [] },
            accountsProvider: { [weak self] in self?.accounts ?? [] }
        )
    }
}
```

#### `UI/Sidebar/Sidebar.swift`

```swift
// Add accountBook case
enum SidebarItem: String, CaseIterable, Identifiable {
    case overview
    case properties
    case accountBook  // NEW
    case summary
    case settings
    // ...
}
```

#### `UI/ContentView.swift`

```swift
// Route to AccountBookView
case .accountBook:
    if let ledgerService = appState.ledgerService {
        AccountBookView(
            viewModel: AccountBookViewModel(ledgerService: ledgerService)
        )
    }
```

---

## Key Benefits

### 1. **Complete Module Independence**
- Ledger module is self-contained
- Can be developed, tested, and maintained independently
- No tight coupling to core app logic

### 2. **Clear Layer Separation**
- **Domain**: Pure business logic, no dependencies
- **Data**: Only depends on Domain
- **Services**: Depends on Domain + Data
- **UI**: Depends on Services only

### 3. **Protocol-Based Integration**
- Core app only knows about `LedgerRepositoryProtocol`
- Implementation details hidden in Ledger module
- Easy to swap implementations (e.g., for testing)

### 4. **Minimal Core Changes**
- Core app only needs to:
  - Add `LedgerRepositoryProtocol` to dependencies
  - Create `LedgerService` instance
  - Route UI to `AccountBookView`

### 5. **Testability**
- Each layer can be tested independently
- Mock repositories for service tests
- Mock services for UI tests

---

## Implementation Phases

### Phase 1: Domain Layer (Week 1)
- ✅ Create domain models (`Ledger`, `LedgerRules`, etc.)
- ✅ Implement filtering service
- ✅ Implement statistics service
- ✅ Unit tests for domain logic

### Phase 2: Data Layer (Week 1-2)
- ✅ Create `LedgerRepositoryProtocol`
- ✅ Implement `FileLedgerRepository`
- ✅ Integration tests for repository

### Phase 3: Services Layer (Week 2)
- ✅ Create `LedgerService`
- ✅ Integrate with domain services
- ✅ Unit tests for service layer

### Phase 4: UI Layer (Week 2-3)
- ✅ Create main `AccountBookView`
- ✅ Implement list and detail panes
- ✅ Add ledger management UI
- ✅ UI tests

### Phase 5: Core Integration (Week 3)
- ✅ Add to `AppDependencies`
- ✅ Integrate with `AppState`
- ✅ Add sidebar item
- ✅ Route in `ContentView`

---

## Future Extensibility

The modular design allows easy extension:

1. **New Rule Types**: Add to `LedgerRules` without affecting other layers
2. **New Statistics**: Add to `LedgerStatisticsService` without UI changes
3. **Alternative Storage**: Swap `FileLedgerRepository` for database implementation
4. **Advanced UI**: Add charts, exports, etc. without touching business logic

---

## Key Design Updates

### Month Card Selector Feature

Based on the user's requirements, the following updates have been made:

1. **Removed Account List**: Ledger detail view no longer shows account list. All transactions are displayed directly.

2. **Added Month Card Selector**: 
   - 12 month cards displayed in a 2x6 grid
   - Year navigation with left/right arrows
   - Each card shows month name, expenses (red), and income (green)
   - Selected month highlighted in orange
   - Clicking selected month again deselects it (shows all transactions)

3. **Transaction Filtering**:
   - When no month is selected: shows all transactions for the ledger
   - When a month is selected: shows only transactions for that month
   - Transactions are grouped by date (not by account)

4. **Monthly Statistics**:
   - New `LedgerMonthlyStatisticsService` calculates monthly income/expenses
   - Statistics displayed on each month card
   - Cards without data show "—" indicator

### UI Flow

```
LedgerDetailPane
├── LedgerHeaderCard (overall statistics)
├── MonthCardSelector
│   ├── Year navigation (← 2025年 →)
│   └── 12 MonthCards (grid layout)
│       ├── 1月 (expenses/income)
│       ├── 2月 (expenses/income)
│       ├── ...
│       └── 12月 (expenses/income)
└── LedgerTransactionList
    └── All transactions (grouped by date, no account grouping)
```

## Balance Statistics Design Summary

### Core Principles

Based on the ledger's positioning and use cases, balance statistics follow these principles:

#### 1. **Core Ledger (核心账本): Balance Tracking REQUIRED**

**Why:**
- **Reconciliation Need**: Users need to verify ledger balances against actual bank account balances
- **Asset Overview**: Core ledger manages all assets/liabilities, balance is direct reflection of financial status
- **Logical Closure**: Core ledger includes transfers, income, expenses - only with balance tracking can it show "transfers don't change total balance, income/expense change balance"

**Implementation:**
- `trackBalances: true` (forced, cannot be disabled)
- `balanceTrackingMode: .accountBased` (track by account dimension)
- Calculate real-time balance: `balance = initialBalance + sum(all transactions)`
- Support historical balance lookup (e.g., "Balance on 2024-05-01")

#### 2. **Scenario Ledgers (场景化账本): Balance Tracking OPTIONAL**

**Why:**
- **Focus on Classification**: Scenario ledgers focus on "categorizing income/expense by scenario", not "tracking actual account funds"
- **Avoid Confusion**: For multi-account scenario ledgers, showing "ledger balance" would confuse users (ledger balance ≠ any bank account balance)
- **Flexible Design**: Allow users to enable balance tracking when needed

**Implementation:**
- `trackBalances: false` (default, can be enabled)
- Three modes available when enabled:
  - `.singleAccount`: For scenario ledger bound to one account (e.g., investment ledger for stock account)
  - `.fundBased`: For special savings ledger (e.g., "house down payment fund")
  - `.accountBased`: Rarely used for scenario ledgers

**Special Cases:**
- **Single Account Binding**: Investment ledger bound to stock account → enable balance tracking
- **Special Savings**: "House down payment ledger" → enable fund-based balance tracking
- **Warning Messages**: When balance tracking is enabled, show warning: "账本余额≠银行账户余额" or "对应账户：XX银行卡"

### Balance Calculation Logic

#### Account-Based Mode (Core Ledger)

```swift
// For each account:
balance = account.initialBalance + sum(transactions for this account)

// Total balance:
totalBalance = sum(all account balances)

// Transfer handling:
// Transfer from A to B: A.balance -= amount, B.balance += amount
// Total balance unchanged (internal transfer)
```

#### Single Account Mode (Scenario Ledger)

```swift
// For bound account:
balance = account.calculateBalance(from: filteredTransactions)

// Uses account type-specific calculation logic (via AccountTraits)
```

#### Fund-Based Mode (Special Savings)

```swift
// Accumulated amount:
balance = sum(all transactions in ledger)

// Represents "saved amount" for specific purpose, not tied to bank account
```

### Historical Balance Lookup

**Core Ledger:**
- Query balance at any date: `historicalBalance(for: ledger, at: date)`
- Query account balances at date: `historicalAccountBalances(for: ledger, at: date)`
- Useful for reconciliation: "What was my balance on 2024-05-01?"

**Scenario Ledgers:**
- Historical balance only available if `trackBalances: true`
- Behavior depends on `balanceTrackingMode`

### UI Display Rules

1. **Core Ledger**: Always show balance section
   - Total balance (sum of all accounts)
   - Expandable account breakdown
   - No warning message

2. **Scenario Ledger (Balance Disabled)**: Hide balance section completely

3. **Scenario Ledger (Balance Enabled)**:
   - Show balance with appropriate label ("账户余额" or "专项资金余额")
   - Show warning message if applicable
   - Single balance display (no account breakdown)

### Validation Rules

- **Core Ledger**: Cannot disable `trackBalances`, cannot change `balanceTrackingMode` from `.accountBased`
- **Scenario Ledger**: Can enable/disable `trackBalances`, can choose any `balanceTrackingMode`
- **Single Account Mode**: Requires `includeAccountIds.count == 1`

## Questions for Discussion

1. Should `LedgerService` be part of `AppState` or completely separate?
2. How should ledger updates trigger UI refreshes? (Combine publishers?)
3. Should statistics be cached or computed on-demand?
4. Do we need ledger templates for quick creation?
5. Should month cards show statistics for all years or only selected year?
6. Should we support balance reconciliation UI (compare ledger balance vs bank statement)?
7. How to handle currency conversion for multi-currency balances?

## Category System Design Summary

### Core Principles

Each ledger has its own independent category system:

1. **Per-Ledger Categories**: Each ledger defines its own categories (e.g., "餐饮", "交通", "购物")
2. **Transaction-Category Mapping**: Each transaction can be assigned to a category in each ledger (stored in `LedgerTransactionCategory`)
3. **Automatic Classification**: Category rules can automatically classify transactions based on merchant, notes, amount, etc.
4. **Manual Override**: Users can manually assign categories to transactions
5. **Tags Support**: Additional flexible tagging for transactions (e.g., "旅游", "报销")

### Data Storage

**Storage Structure:**
- Categories: `ledger_categories_{ledgerId}.json`
- Transaction-Category Mappings: `ledger_mappings_{ledgerId}.json`
- Category Rules: `ledger_rules_{ledgerId}.json`

**Key Points:**
- Data is stored in ledger-based repository (separate files per ledger)
- Transaction-category mappings are linked by `transactionId` and `ledgerId`
- One transaction can have different categories in different ledgers
- Category rules are evaluated in priority order (higher priority first)

### Category System Components

#### 1. **LedgerCategory** (Category Definition)
- `id`: Unique category ID (UUID)
- `ledgerId`: Which ledger this category belongs to
- `name`: Category name (e.g., "餐饮", "交通")
- `color`: Optional hex color code for UI display
- `icon`: Optional SF Symbol name
- `parentId`: Optional parent category ID (for hierarchical categories)

#### 2. **LedgerTransactionCategory** (Transaction-Category Mapping)
- `transactionId`: Transaction ID
- `ledgerId`: Ledger ID
- `categoryId`: Assigned category ID
- `tags`: Set of tags (e.g., ["旅游", "报销"])
- `assignedBy`: "auto" or "manual"

#### 3. **LedgerCategoryRule** (Automatic Classification Rules)
- `categoryId`: Target category
- `priority`: Rule priority (higher = evaluated first)
- `conditions`: Matching conditions (merchant pattern, notes pattern, amount range, etc.)

### Classification Flow

1. **Automatic Classification**:
   - When a transaction is added to a ledger, evaluate category rules
   - Match transaction against rules (sorted by priority)
   - First matching rule assigns the category
   - Save mapping with `assignedBy: "auto"`

2. **Manual Classification**:
   - User manually assigns category to a transaction
   - Save mapping with `assignedBy: "manual"`
   - Manual assignments override automatic ones

3. **Rule Evaluation**:
   - Rules are evaluated in priority order (highest first)
   - First matching rule wins
   - Conditions can match: merchant name, notes, account, amount range, transaction type

### Filtering by Category

**LedgerRules** supports category-based filtering:
- `includeCategories: Set<UUID>?` - Only show transactions in these categories
- `excludeCategories: Set<UUID>?` - Hide transactions in these categories
- `includeTags: Set<String>?` - Only show transactions with these tags
- `excludeTags: Set<String>?` - Hide transactions with these tags

**Example:**
- Lifestyle ledger: `excludeCategories: ["投资", "转账"]` - Hide investment and transfer categories
- Travel ledger: `includeTags: ["旅游"]` - Only show transactions tagged with "旅游"

### Use Cases

1. **Core Ledger**: 
   - Comprehensive categories (all transaction types)
   - Automatic classification rules for common merchants
   - Manual categorization for edge cases

2. **Lifestyle Ledger**:
   - Focused categories (餐饮, 交通, 购物, 娱乐)
   - Exclude investment and transfer categories
   - Tags for specific events (e.g., "双十一", "春节")

3. **Travel Ledger**:
   - Travel-specific categories (机票, 酒店, 餐饮, 景点)
   - Tag transactions with "旅游" tag
   - Filter by date range and "旅游" tag

4. **Investment Ledger**:
   - Investment categories (股票, 基金, 债券)
   - Track investment-related transactions only
   - Exclude daily expenses

### Integration with LedgerService

`LedgerService.transactions(for:categoryMappings:)` accepts optional category mappings:
- If provided, applies category-based filtering
- If not provided, skips category filtering (for performance)
- Category mappings are loaded from `LedgerCategoryRepository` when needed

### Future Enhancements

1. **Hierarchical Categories**: Support parent-child category relationships
2. **Category Templates**: Pre-defined category sets for common ledger types
3. **Smart Suggestions**: ML-based category suggestions based on transaction history
4. **Bulk Operations**: Assign categories to multiple transactions at once
5. **Category Statistics**: Show spending by category in ledger statistics

