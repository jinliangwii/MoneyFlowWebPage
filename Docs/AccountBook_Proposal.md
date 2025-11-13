# Account Book (账本) Feature Proposal

## Overview

This proposal outlines the implementation of the **Account Book (账本)** feature, which acts as a **database view** over the transaction data. Each ledger provides a filtered, categorized, and aggregated view of transactions based on customizable rules, following the Table-View pattern from database design.

## Architecture Alignment

### Current Architecture
- **Raw Data Layer (Table)**: Raw transactions from bank imports (preserved in `RawTransaction` models)
- **Account Layer (Standardized Table)**: Normalized `Transaction` and `Account` models
- **View Layer (NEW)**: Ledgers that provide virtual views over transactions

### Design Principles
1. **Views are Virtual**: Ledgers don't store transaction data; they filter and present data from the base `Transaction` table
2. **Rule-Based Filtering**: Each ledger defines rules (like SQL WHERE clauses) to determine which transactions belong to it
3. **Multi-Ledger Support**: A single transaction can belong to multiple ledgers (many-to-many relationship, logical only)
4. **Default Core Ledger**: One system ledger that includes all transactions, serving as a template for custom ledgers

---

## Data Model

### 1. Ledger Model

```swift
// Core/Domain/Models.swift (add to existing file)

public struct Ledger: Identifiable, Codable, Hashable {
    public var id: UUID
    public var name: String
    public var type: LedgerType
    public var isDefault: Bool  // Only one default ledger (core ledger)
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
        self.rules = rules
        self.createdAt = createdAt
        self.updatedAt = updatedAt
    }
}

public enum LedgerType: String, Codable, CaseIterable {
    case core        // 核心账本 (system default)
    case lifestyle   // 生活账本
    case investment  // 投资账本
    case travel      // 旅游账本
    case custom      // 自定义账本
    
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

### 2. Ledger Rules (Filtering Logic)

```swift
// Core/Domain/Models.swift

public struct LedgerRules: Codable, Hashable {
    // Include/Exclude Logic
    public var includeAll: Bool  // If true, include all transactions (core ledger)
    public var includeAccountIds: Set<UUID>?  // Include only specific accounts
    public var excludeAccountIds: Set<UUID>?  // Exclude specific accounts
    
    // Category/Tag Filtering (for future categorization system)
    public var includeCategories: Set<String>?
    public var excludeCategories: Set<String>?
    public var includeTags: Set<String>?
    public var excludeTags: Set<String>?
    
    // Date Range Filtering
    public var dateRange: DateRangeFilter?
    
    // Amount Filtering
    public var minAmount: Decimal?
    public var maxAmount: Decimal?
    public var excludeSmallAmounts: Bool  // Exclude transactions below threshold
    public var smallAmountThreshold: Decimal?
    
    // Transaction Type Filtering
    public var includeIncome: Bool
    public var includeExpense: Bool
    public var includeTransfer: Bool  // Future: detect transfers between accounts
    
    public init(
        includeAll: Bool = false,
        includeAccountIds: Set<UUID>? = nil,
        excludeAccountIds: Set<UUID>? = nil,
        includeCategories: Set<String>? = nil,
        excludeCategories: Set<String>? = nil,
        includeTags: Set<String>? = nil,
        excludeTags: Set<String>? = nil,
        dateRange: DateRangeFilter? = nil,
        minAmount: Decimal? = nil,
        maxAmount: Decimal? = nil,
        excludeSmallAmounts: Bool = false,
        smallAmountThreshold: Decimal? = nil,
        includeIncome: Bool = true,
        includeExpense: Bool = true,
        includeTransfer: Bool = true
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
    }
    
    /// Default rules for core ledger (include all transactions)
    public static var coreLedgerRules: LedgerRules {
        LedgerRules(includeAll: true)
    }
    
    /// Rules for lifestyle ledger (exclude investment and transfers)
    public static var lifestyleLedgerRules: LedgerRules {
        LedgerRules(
            includeAll: true,
            excludeCategories: ["投资", "转账"],
            includeIncome: true,
            includeExpense: true,
            includeTransfer: false
        )
    }
}

public struct DateRangeFilter: Codable, Hashable {
    public var startDate: Date?
    public var endDate: Date?
    public var isRelative: Bool  // If true, use relative dates (e.g., "last 30 days")
    public var relativeDays: Int?  // Number of days from today
    
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
    
    /// Get effective date range
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

### 3. Ledger Transaction Filtering Logic

```swift
// Core/Domain/Models.swift (extension)

extension Ledger {
    /// Filter transactions based on ledger rules
    /// - Parameter transactions: All available transactions
    /// - Returns: Filtered transactions that belong to this ledger
    public func filterTransactions(_ transactions: [Transaction], accounts: [Account]) -> [Transaction] {
        var filtered = transactions
        
        // Rule 1: Include All (core ledger)
        if rules.includeAll {
            // Continue with other filters
        } else {
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
        
        // Rule 5: Transaction type filtering (income/expense)
        // Note: amount > 0 is income, amount < 0 is expense
        var typeFiltered: [Transaction] = []
        if rules.includeIncome {
            typeFiltered.append(contentsOf: filtered.filter { $0.amount > 0 })
        }
        if rules.includeExpense {
            typeFiltered.append(contentsOf: filtered.filter { $0.amount < 0 })
        }
        // Remove duplicates (transactions that match both income and expense criteria)
        filtered = Array(Set(typeFiltered))
        
        // Rule 6: Category/Tag filtering (future implementation)
        // This will be implemented when categorization system is added
        
        // Sort by date (newest first)
        return Transaction.sortByDateDescending(filtered)
    }
    
    /// Calculate statistics for this ledger
    public func calculateStatistics(_ transactions: [Transaction], accounts: [Account]) -> LedgerStatistics {
        let filtered = filterTransactions(transactions, accounts: accounts)
        let income = filtered.filter { $0.amount > 0 }.reduce(Decimal(0)) { $0 + $1.amount }
        let expenses = filtered.filter { $0.amount < 0 }.reduce(Decimal(0)) { $0 + abs($1.amount) }
        let net = income - expenses
        
        return LedgerStatistics(
            totalTransactions: filtered.count,
            totalIncome: income,
            totalExpenses: expenses,
            net: net
        )
    }
}

public struct LedgerStatistics: Hashable {
    public let totalTransactions: Int
    public let totalIncome: Decimal
    public let totalExpenses: Decimal
    public let net: Decimal
}
```

---

## Repository Layer

### Ledger Repository

```swift
// Core/Data/Protocols.swift (add to existing file)

public protocol LedgerRepositoryProtocol {
    func findAll() async throws -> [Ledger]
    func find(id: UUID) async throws -> Ledger?
    func findDefault() async throws -> Ledger?
    func save(_ ledger: Ledger) async throws
    func save(_ ledgers: [Ledger]) async throws
    func delete(_ ledger: Ledger) async throws
}

// DataSources/Implementations/LedgerRepository.swift (new file)
// Similar to existing TransactionRepository and AccountRepository
// Uses JSON file storage: "ledgers.json"
```

---

## State Management

### AppState Extensions

```swift
// Core/State/AppState.swift (add to existing class)

@MainActor
final class AppState: ObservableObject {
    // ... existing properties ...
    
    @Published private(set) var ledgers: [Ledger] = []
    private let ledgerRepository: LedgerRepositoryProtocol
    
    // ... existing init ...
    // Add ledgerRepository to dependencies
    
    func load() async {
        // ... existing load logic ...
        
        // Load ledgers
        do {
            ledgers = try await ledgerRepository.findAll()
            
            // Ensure default ledger exists
            if ledgers.first(where: { $0.isDefault }) == nil {
                let defaultLedger = Ledger(
                    name: "核心账本",
                    type: .core,
                    isDefault: true,
                    rules: .coreLedgerRules
                )
                ledgers.append(defaultLedger)
                try await ledgerRepository.save(defaultLedger)
            }
        } catch {
            // Handle error
        }
    }
    
    // Ledger-specific methods
    func transactions(for ledger: Ledger) -> [Transaction] {
        ledger.filterTransactions(transactions, accounts: accounts)
    }
    
    func statistics(for ledger: Ledger) -> LedgerStatistics {
        ledger.calculateStatistics(transactions, accounts: accounts)
    }
    
    func addLedger(_ ledger: Ledger) async {
        // Ensure only one default ledger
        if ledger.isDefault {
            // Remove default flag from other ledgers
            for i in ledgers.indices {
                ledgers[i].isDefault = false
            }
        }
        ledgers.append(ledger)
        try? await ledgerRepository.save(ledger)
    }
    
    func updateLedger(_ ledger: Ledger) async {
        if let index = ledgers.firstIndex(where: { $0.id == ledger.id }) {
            // Handle default ledger constraint
            if ledger.isDefault {
                for i in ledgers.indices where i != index {
                    ledgers[i].isDefault = false
                }
            }
            ledgers[index] = ledger
            try? await ledgerRepository.save(ledger)
        }
    }
    
    func deleteLedger(_ ledger: Ledger) async {
        // Prevent deleting default ledger
        guard !ledger.isDefault else { return }
        ledgers.removeAll { $0.id == ledger.id }
        try? await ledgerRepository.delete(ledger)
    }
}
```

---

## UI Layer

### 1. Sidebar Integration

```swift
// UI/Sidebar/Sidebar.swift (modify existing)

enum SidebarItem: String, CaseIterable, Identifiable {
    case overview
    case properties
    case accountBook  // NEW: 账本
    case summary
    case settings
    #if DEBUG
    case debug
    #endif
    
    var id: String { rawValue }
    var title: String {
        switch self {
        case .overview: return "概览"
        case .properties: return "资产"
        case .accountBook: return "账本"  // NEW
        case .summary: return "统计"
        case .settings: return "设置"
        #if DEBUG
        case .debug: return "Debug"
        #endif
        }
    }
    var systemImage: String {
        switch self {
        case .overview: return "rectangle.grid.2x2.fill"
        case .properties: return "building.columns.fill"
        case .accountBook: return "book.fill"  // NEW
        case .summary: return "chart.pie.fill"
        case .settings: return "gearshape.fill"
        #if DEBUG
        case .debug: return "ant.circle.fill"
        #endif
        }
    }
    
    // Update allCases to include .accountBook
}
```

### 2. Account Book View Structure

```
UI/
├── AccountBook/
│   ├── AccountBookView.swift          # Main container view
│   ├── AccountBookViewModel.swift     # View model (optional, for complex logic)
│   ├── Components/
│   │   ├── LedgerListPane.swift       # Left pane: list of ledgers
│   │   ├── LedgerDetailPane.swift     # Right pane: selected ledger transactions
│   │   ├── LedgerRow.swift            # Individual ledger row in list
│   │   ├── LedgerHeaderCard.swift     # Statistics card for selected ledger
│   │   ├── LedgerTransactionList.swift # Transaction list for ledger
│   │   ├── LedgerRulesEditor.swift    # Edit ledger rules (future)
│   │   └── AddLedgerView.swift        # Create new ledger dialog
```

### 3. Account Book View Implementation

```swift
// UI/AccountBook/AccountBookView.swift

struct AccountBookView: View {
    @EnvironmentObject private var app: AppState
    @State private var selectedLedger: Ledger?
    @State private var showAddLedger = false
    
    var body: some View {
        HSplitView {
            // Left Pane: Ledger List
            LedgerListPane(
                selectedLedger: $selectedLedger,
                onAdd: { showAddLedger = true }
            )
            .frame(minWidth: 280, idealWidth: 300, maxWidth: 350)
            
            // Right Pane: Ledger Detail
            if let ledger = selectedLedger {
                LedgerDetailPane(ledger: ledger)
            } else {
                EmptyLedgerState()
            }
        }
        .sheet(isPresented: $showAddLedger) {
            AddLedgerView()
        }
    }
}

// UI/AccountBook/Components/LedgerListPane.swift

struct LedgerListPane: View {
    @EnvironmentObject private var app: AppState
    @Binding var selectedLedger: Ledger?
    var onAdd: () -> Void
    
    var body: some View {
        ScrollView {
            VStack(spacing: 12) {
                // Header with add button
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
                    ForEach(app.ledgers) { ledger in
                        LedgerRow(
                            ledger: ledger,
                            isSelected: selectedLedger?.id == ledger.id,
                            onSelect: { selectedLedger = ledger }
                        )
                    }
                }
                .padding(.horizontal, 16)
            }
        }
    }
}

// UI/AccountBook/Components/LedgerRow.swift

struct LedgerRow: View {
    let ledger: Ledger
    let isSelected: Bool
    let onSelect: () -> Void
    @EnvironmentObject private var app: AppState
    
    var body: some View {
        Button(action: onSelect) {
            HStack(spacing: 12) {
                Image(systemName: ledger.type.systemImage)
                    .foregroundColor(ledger.isDefault ? .blue : .secondary)
                    .frame(width: 24)
                
                VStack(alignment: .leading, spacing: 4) {
                    Text(ledger.name)
                        .font(.headline)
                    if let stats = statistics {
                        Text("\(stats.totalTransactions) 笔交易")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                
                Spacer()
                
                if ledger.isDefault {
                    Text("默认")
                        .font(.caption)
                        .foregroundStyle(.blue)
                        .padding(.horizontal, 8)
                        .padding(.vertical, 4)
                        .background(Capsule().fill(Color.blue.opacity(0.1)))
                }
            }
            .padding(.horizontal, 12)
            .padding(.vertical, 12)
            .background(
                RoundedRectangle(cornerRadius: 10)
                    .fill(isSelected ? Color.appTertiaryBackground : .clear)
            )
        }
        .buttonStyle(.plain)
    }
    
    private var statistics: LedgerStatistics? {
        app.statistics(for: ledger)
    }
}

// UI/AccountBook/Components/LedgerDetailPane.swift

struct LedgerDetailPane: View {
    let ledger: Ledger
    @EnvironmentObject private var app: AppState
    
    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                // Header with statistics
                LedgerHeaderCard(ledger: ledger)
                    .padding(.horizontal, 16)
                    .padding(.top, 16)
                
                // Transaction list (similar to PropertiesView)
                LedgerTransactionList(ledger: ledger)
                    .padding(.horizontal, 16)
            }
        }
    }
}

// UI/AccountBook/Components/LedgerHeaderCard.swift

struct LedgerHeaderCard: View {
    let ledger: Ledger
    @EnvironmentObject private var app: AppState
    
    var body: some View {
        let stats = app.statistics(for: ledger)
        
        VStack(spacing: 16) {
            HStack {
                Image(systemName: ledger.type.systemImage)
                    .font(.title)
                Text(ledger.name)
                    .font(.title2)
                    .bold()
                Spacer()
            }
            
            HStack(spacing: 24) {
                StatCard(
                    title: "收入",
                    value: currency(stats.totalIncome),
                    color: .green
                )
                StatCard(
                    title: "支出",
                    value: currency(stats.totalExpenses),
                    color: .red
                )
                StatCard(
                    title: "净额",
                    value: currency(stats.net),
                    color: stats.net >= 0 ? .green : .red
                )
            }
        }
        .padding(16)
        .background(RoundedRectangle(cornerRadius: 12).fill(Color.appSecondaryBackground))
    }
    
    private func currency(_ value: Decimal) -> String {
        // Use existing Formatters.currency
        let formatter = Formatters.currency
        return formatter.string(from: NSDecimalNumber(decimal: value)) ?? "\(value)"
    }
}

// UI/AccountBook/Components/LedgerTransactionList.swift

struct LedgerTransactionList: View {
    let ledger: Ledger
    @EnvironmentObject private var app: AppState
    
    var body: some View {
        let transactions = app.transactions(for: ledger)
        
        // Group by month (similar to AccountMonthlyList)
        let grouped = Dictionary(grouping: transactions) { tx in
            Calendar.current.dateInterval(of: .month, for: tx.date)?.start ?? tx.date
        }
        let sortedMonths = Array(grouped.keys).sorted(by: >)
        
        VStack(alignment: .leading, spacing: 16) {
            ForEach(sortedMonths, id: \.self) { monthStart in
                MonthlyTransactionSection(
                    month: monthStart,
                    transactions: grouped[monthStart] ?? [],
                    ledger: ledger
                )
            }
        }
    }
}
```

### 4. ContentView Integration

```swift
// UI/ContentView.swift (modify existing)

struct ContentView: View {
    // ... existing code ...
    
    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            SidebarView(selection: $selection)
                .navigationSplitViewColumnWidth(min: 220, ideal: 240, max: 300)
        } detail: {
            Group {
                switch selection ?? .properties {
                case .overview:
                    DashboardView()
                case .properties:
                    PropertiesView()
                case .accountBook:  // NEW
                    AccountBookView()
                case .summary:
                    SummaryView()
                case .settings:
                    SettingsView()
                #if DEBUG
                case .debug:
                    DebugPaneView()
                #endif
                }
            }
        }
        // ... existing alert code ...
    }
}
```

---

## Implementation Phases

### Phase 1: Core Infrastructure (MVP)
1. ✅ Add `Ledger` and `LedgerRules` models
2. ✅ Implement `LedgerRepository`
3. ✅ Add ledger management to `AppState`
4. ✅ Create default core ledger on first launch
5. ✅ Add sidebar item for Account Book

### Phase 2: Basic UI
1. ✅ Create `AccountBookView` with two-pane layout
2. ✅ Implement `LedgerListPane` (left)
3. ✅ Implement `LedgerDetailPane` (right)
4. ✅ Show transaction list filtered by ledger rules
5. ✅ Display basic statistics (income, expense, net)

### Phase 3: Ledger Management
1. ✅ Add "Create Ledger" functionality
2. ✅ Implement basic rule editor (account selection, date range)
3. ✅ Add ledger deletion (prevent deleting default)
4. ✅ Add ledger editing

### Phase 4: Advanced Features
1. ⏳ Category/Tag filtering (when categorization system is ready)
2. ⏳ Transfer detection and handling
3. ⏳ Advanced rule editor UI
4. ⏳ Ledger templates (lifestyle, investment, travel)
5. ⏳ Export ledger data
6. ⏳ Ledger-specific charts and analytics

---

## Key Design Decisions

### 1. Virtual Views (No Physical Storage)
- Transactions are NOT duplicated for each ledger
- Ledgers only store rules, not transaction IDs
- Filtering happens at query time (like SQL views)

### 2. Default Core Ledger
- Always exists and cannot be deleted
- Serves as template for creating new ledgers
- Includes all transactions by default

### 3. Rule-Based Filtering
- Rules are stored as structured data (not SQL queries)
- Easy to serialize/deserialize
- Can be extended with new rule types in the future

### 4. Performance Considerations
- Filtering is done in-memory (fast for typical transaction volumes)
- Statistics can be cached per ledger
- Consider lazy loading for very large transaction sets

### 5. Future Extensibility
- Rules structure supports future features (categories, tags, transfer detection)
- Ledger types can be extended
- UI components are modular and reusable

---

## Next Steps

1. **Review this proposal** and provide feedback
2. **Start with Phase 1**: Implement data models and repository
3. **Test with sample data**: Ensure filtering logic works correctly
4. **Iterate on UI**: Build basic UI and refine based on usage
5. **Add advanced features**: Gradually implement Phase 3 and 4 features

---

## Questions for Discussion

1. Should ledgers support custom categorization rules, or use a global categorization system?
2. How should transfers between accounts be handled in ledgers? (Show both sides? Show net effect?)
3. Should ledgers support time-based rules (e.g., "only transactions from last 30 days")?
4. Do we need ledger-level currency conversion, or use base currency?
5. Should statistics be pre-calculated and cached, or computed on-demand?

