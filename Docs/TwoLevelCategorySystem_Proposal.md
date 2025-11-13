# Two-Level Category System Proposal

## Overview

This proposal introduces a **two-level category system** for transactions:
1. **First Level Category (Transaction Classification)**: High-level classification assigned during import/conversion
2. **Second Level Category (Detailed Categorization)**: Fine-grained categorization managed by Ledgers

---

## 1. First Level Category (Transaction Classification)

### Purpose
- **Automatic assignment** during transaction import/conversion
- **Account type-aware** classification rules
- **Foundation** for filtering, reporting, and second-level categorization

### Enum Definition

```swift
// Core/Domain/Models.swift

/// First-level transaction classification
/// Assigned automatically during import/conversion based on account type and transaction characteristics
public enum TransactionClassification: String, Codable, CaseIterable {
    case expense    // 支出 - Money going out or debt increasing
    case income     // 收入 - Money coming in or earning
    case neutral    // 中性 - Transfers, balance adjustments, or non-categorized movements
    
    public var displayName: String {
        switch self {
        case .expense: return "支出"
        case .income: return "收入"
        case .neutral: return "中性"
        }
    }
    
    public var systemImage: String {
        switch self {
        case .expense: return "arrow.down.circle.fill"
        case .income: return "arrow.up.circle.fill"
        case .neutral: return "arrow.left.arrow.right.circle.fill"
        }
    }
}
```

### Transaction Model Update

```swift
// Core/Domain/Models.swift

public struct Transaction: Identifiable, Codable, Hashable {
    public var id: UUID
    public var date: Date
    public var merchant: String
    public var amount: Decimal
    public var notes: String?
    public var accountId: UUID
    public var sequenceNumber: Int
    public var balance: Decimal
    
    // NEW: First-level classification
    public var classification: TransactionClassification
    
    public init(
        id: UUID = UUID(),
        date: Date,
        merchant: String,
        amount: Decimal,
        notes: String? = nil,
        accountId: UUID,
        sequenceNumber: Int,
        balance: Decimal,
        classification: TransactionClassification  // NEW parameter
    ) {
        self.id = id
        self.date = date
        self.merchant = merchant
        self.amount = amount
        self.notes = notes
        self.accountId = accountId
        self.sequenceNumber = sequenceNumber
        self.balance = balance
        self.classification = classification
    }
}
```

### Account Type Rules for First Level Category

#### Credit Card (`creditCard`)
- **Allowed**: `expense`, `neutral`
- **Not Allowed**: `income`
- **Rationale**: Credit card transactions are either spending (expense) or payments/refunds (neutral). Money going into a credit card is paying debt, not earning income.

#### Loan (`loan`)
- **Allowed**: `expense`, `neutral`
- **Not Allowed**: `income`
- **Rationale**: 
  - **In loan account perspective**: Paying debt (positive amount reducing debt) is `neutral` - it's just reducing debt, not an expense from the loan account's perspective
  - **In bank account perspective**: Paying debt (negative amount going out) is `expense` - money leaving the bank account
  - **Note**: Classification is based on the account where the transaction is recorded. The same payment appears as `neutral` in the loan account and `expense` in the bank account.

#### Deposit/Checking (`deposit`)
- **Allowed**: `expense`, `income`, `neutral`
- **Rationale**: Can receive income, make expenses, or handle transfers.

#### Savings (`savings`)
- **Allowed**: `expense`, `income`, `neutral`
- **Rationale**: Similar to deposit accounts.

#### Brokerage (`brokerage`)
- **Allowed**: `expense`, `income`, `neutral`
- **Rationale**: Stock purchases are `neutral` (investments, not consumption). Dividends are `income`, fees are `expense`, and transfers are `neutral`.

#### Wallet (`wallet`)
- **Allowed**: `expense`, `income`, `neutral`
- **Rationale**: Similar to deposit accounts, can receive money, spend money, or transfer.

#### Payment Gateway (`paymentGate`)
- **Allowed**: `neutral` only
- **Not Allowed**: `expense`, `income`
- **Rationale**: Payment gateways (e.g., Alipay, WeChat Pay) are intermediate transaction holders. All transactions will eventually reflect in real bank accounts or cash accounts. These are just intermediate records, so all transactions should be classified as `neutral`.

### Classification Logic

**Important Principle**: Classification is **account-perspective based**. The same payment transaction appears differently depending on which account it's recorded in:
- **Loan account**: Payment reducing debt (positive amount) = `neutral` (reducing debt, not expense from loan's perspective)
- **Bank account**: Payment going out (negative amount) = `expense` (money leaving the account)

The classification service determines the classification based on the account where the transaction is recorded.

#### Classification Service

```swift
// Core/Domain/Services/TransactionClassificationService.swift

public struct TransactionClassificationService {
    /// Determine first-level classification for a transaction
    /// - Parameters:
    ///   - amount: Transaction amount (positive = money in, negative = money out)
    ///   - accountType: Type of account this transaction belongs to
    ///   - rawTransaction: Optional raw transaction data for context
    /// - Returns: Appropriate classification
    public static func classify(
        amount: Decimal,
        accountType: AccountType,
        rawTransaction: (any RawTransactionProtocol)? = nil
    ) -> TransactionClassification {
        // Account type-specific rules
        switch accountType {
        case .creditCard, .loan:
            // Credit cards and loans: only expense or neutral
            return classifyForDebtAccount(amount: amount, rawTransaction: rawTransaction)
            
        case .paymentGate:
            // Payment gateways: always neutral (intermediate transactions)
            return .neutral
            
        case .deposit, .savings, .brokerage, .wallet:
            // Other accounts: can have all three types
            return classifyForAssetAccount(amount: amount, rawTransaction: rawTransaction)
        }
    }
    
    private static func classifyForDebtAccount(
        amount: Decimal,
        rawTransaction: (any RawTransactionProtocol)?
    ) -> TransactionClassification {
        // For credit cards and loans (debt accounts):
        // - Negative amount = expense (spending/increasing debt from account's perspective)
        // - Positive amount = neutral (payment/refund reducing debt - not an expense from debt account's perspective)
        // 
        // Note: The same payment appears as:
        //   - neutral in the loan account (reducing debt)
        //   - expense in the bank account (money going out)
        // Classification is always based on the account where the transaction is recorded.
        
        if amount < 0 {
            return .expense
        } else {
            // Check if it's a refund (could be marked as neutral)
            if let raw = rawTransaction {
                if raw.isNeutral {
                    return .neutral
                }
            }
            // Default: positive amount on debt account is neutral (payment reducing debt)
            return .neutral
        }
    }
    
    private static func classifyForAssetAccount(
        amount: Decimal,
        rawTransaction: (any RawTransactionProtocol)?
    ) -> TransactionClassification {
        // For deposit/savings/brokerage/wallet:
        // - Positive amount = income (money coming in)
        // - Negative amount = expense (money going out)
        // - Neutral flag = neutral (transfer)
        
        if let raw = rawTransaction, raw.isNeutral {
            return .neutral
        }
        
        if amount > 0 {
            return .income
        } else {
            return .expense
        }
    }
}
```

### Integration Layer Updates

#### Transaction Converter Protocol Update

```swift
// Integration/Capabilities/TransactionConverter.swift

public protocol TransactionConverter {
    associatedtype RawTransactionType: RawTransactionProtocol
    
    /// Convert a raw transaction to a domain transaction
    /// - Parameters:
    ///   - raw: The raw transaction to convert
    ///   - sequenceNumber: Sequence number for this transaction
    ///   - balance: Balance after this transaction
    ///   - accountType: Account type for classification
    /// - Returns: Domain transaction, or nil if conversion fails
    func convert(
        _ raw: RawTransactionType,
        sequenceNumber: Int,
        balance: Decimal,
        accountType: AccountType  // NEW parameter
    ) -> Transaction?
}
```

#### Converter Implementation Example

```swift
// Integration/CapabilityImplementations/Converters/CMBCreditCardTransactionConverter.swift

struct CMBCreditCardTransactionConverter: TransactionConverter {
    typealias RawTransactionType = RawCMBCreditCardTransaction
    
    func convert(
        _ raw: RawCMBCreditCardTransaction,
        sequenceNumber: Int,
        balance: Decimal,
        accountType: AccountType
    ) -> Transaction? {
        guard let date = raw.parsedDate,
              let amount = raw.parsedAmount else {
            return nil
        }
        
        let transactionAmount = calculateTransactionAmount(raw: raw, amount: amount)
        let classification = TransactionClassificationService.classify(
            amount: transactionAmount,
            accountType: accountType,
            rawTransaction: raw
        )
        
        let merchant = extractMerchantName(from: raw)
        let notes = buildNotes(from: raw)
        
        return Transaction(
            date: date,
            merchant: merchant,
            amount: transactionAmount,
            notes: notes,
            accountId: raw.accountId,
            sequenceNumber: sequenceNumber,
            balance: balance,
            classification: classification  // NEW
        )
    }
    
    // ... existing helper methods ...
}
```

---

## 2. Second Level Category (Detailed Categorization)

### Purpose
- **Fine-grained categorization** (e.g., "餐饮", "交通", "购物")
- **Ledger-specific** categorization
- **User-manageable** through category templates
- **Automatic classification** via rules

### Renaming Proposal

**Current Name**: `CategoryTemplate` → **Proposed Name**: `TransactionCategory` or `CategoryScheme`

**Rationale**: 
- "CategoryTemplate" sounds like a template for creating categories, not the categorization system itself
- "TransactionCategory" is clearer but might be confused with first-level classification
- "CategoryScheme" indicates a scheme/system for categorizing transactions

**Recommendation**: **`CategoryScheme`** (分类方案)
- Clear that it's a scheme/system for categorization
- Distinct from first-level classification
- Professional terminology

### Updated Model Names

```swift
// CategoryTemplates/Domain/Models/CategoryTemplate.swift → CategoryScheme.swift

// Rename:
CategoryTemplate → CategoryScheme
CategoryTemplateCategory → CategorySchemeCategory (or just CategorySchemeItem)
CategoryTemplateRule → CategorySchemeRule
CategoryTemplateService → CategorySchemeService
CategoryTemplateRepository → CategorySchemeRepository
```

### Structure (Unchanged, Just Renamed)

```swift
// CategoryTemplates/Domain/Models/CategoryScheme.swift

/// Category scheme - a reusable set of categories and rules for detailed transaction categorization
/// Schemes can be assigned to ledgers at creation time
public struct CategoryScheme: Identifiable, Codable, Hashable {
    public let id: UUID
    public var name: String
    public var description: String?
    public var isDefault: Bool
    public var isSystem: Bool
    public var createdAt: Date
    public var updatedAt: Date
    // ... existing implementation ...
}

/// Category definition within a scheme
public struct CategorySchemeCategory: Identifiable, Codable, Hashable {
    public let id: UUID
    public let schemeId: UUID  // Changed from templateId
    public let name: String
    public let color: String?
    public let icon: String?
    public let parentId: UUID?
    public let createdAt: Date
    public let updatedAt: Date
    // ... existing implementation ...
}

/// Category rule for automatic classification within a scheme
public struct CategorySchemeRule: Identifiable, Codable, Hashable {
    public let id: UUID
    public let schemeId: UUID  // Changed from templateId
    public let categoryId: UUID
    public let priority: Int
    public let conditions: CategoryRuleConditions
    public let createdAt: Date
    public let updatedAt: Date
    // ... existing implementation ...
}
```

### Ledger Integration

The Ledger layer will continue to:
- **Store** category scheme assignments
- **Assign** second-level categories to transactions
- **Use** category rules for automatic classification
- **Allow** manual override of category assignments

**No changes needed** to the existing ledger-category mapping system, just renaming.

---

## 3. Migration Strategy

### Phase 1: Add First-Level Classification
1. Add `TransactionClassification` enum
2. Add `classification` property to `Transaction` model (**required field**, no default)
3. Update `TransactionConverter` protocol
4. Update all converter implementations
5. Update `TransactionClassificationService` with account type rules
6. **No migration needed**: Classification is required for all new transactions during import

### Phase 2: Rename Category Template System
1. Rename all `CategoryTemplate*` types to `CategoryScheme*`
2. Update all references in codebase
3. Update repository file names
4. **Migration**: Rename JSON files if needed

### Phase 3: UI Updates
1. Update transaction rows to use classification for icons/colors
2. Add classification badges to transaction rows
3. Update transaction detail views to show both classification levels
4. Update category selection to filter by classification
5. Update filtering logic to use first-level classification
6. Update statistics calculations to use classification
7. Create new UI components (ClassificationBadge, enhanced DetailRow)

---

## 4. Account Type Classification Rules - Confirmed

### Final Rules:

1. **Loan Accounts (`loan`)**:
   - ✅ **Confirmed**: 
     - **In loan account**: Paying debt (positive amount) is `neutral` (reducing debt, not expense from loan account's perspective)
     - **In bank account**: Paying debt (negative amount) is `expense` (money going out)
     - Classification is account-perspective based

2. **Brokerage Accounts (`brokerage`)**:
   - ✅ **Confirmed**: Stock purchases are `neutral` (investments, not consumption)

3. **Payment Gateway (`paymentGate`)**:
   - ✅ **Confirmed**: ALL transactions are `neutral` (intermediate transactions that will reflect in real accounts)

4. **Wallet (`wallet`)**:
   - ✅ **Confirmed**: Transfers are `neutral` (as per existing logic)

### Final Account Type Rules:

| Account Type | Income Allowed? | Expense Allowed? | Neutral Allowed? | Notes |
|-------------|----------------|------------------|-----------------|-------|
| `creditCard` | ❌ No | ✅ Yes | ✅ Yes | Payments/refunds are neutral |
| `loan` | ❌ No | ✅ Yes | ✅ Yes | Payments reducing debt are neutral (in loan account), borrowing is expense |
| `deposit` | ✅ Yes | ✅ Yes | ✅ Yes | All types allowed |
| `savings` | ✅ Yes | ✅ Yes | ✅ Yes | All types allowed |
| `brokerage` | ✅ Yes | ✅ Yes | ✅ Yes | Stock purchases are neutral, dividends are income, fees are expense |
| `wallet` | ✅ Yes | ✅ Yes | ✅ Yes | All types allowed |
| `paymentGate` | ❌ No | ❌ No | ✅ Yes | **ALL transactions are neutral** (intermediate records) |

---

## 5. Implementation Impact

### Files to Modify

#### Core Domain
- `Core/Domain/Models.swift` - Add `TransactionClassification` enum and update `Transaction` model
- `Core/Domain/Services/TransactionClassificationService.swift` - NEW file

#### Integration Layer
- `Integration/Capabilities/TransactionConverter.swift` - Add `accountType` parameter
- All converter implementations:
  - `CMBCreditCardTransactionConverter.swift`
  - `CMBPersonalCheckingTransactionConverter.swift`
  - `CMBExpressLoanTransactionConverter.swift`
  - `AlipayTransactionConverter.swift`
  - `WeChatTransactionConverter.swift`
  - `PlaidTransactionConverter.swift` (if exists)

#### Category Template System (Rename)
- All files in `CategoryTemplates/` directory
- Rename `CategoryTemplate*` to `CategoryScheme*`

#### Integration Orchestrator
- `Integration/Core/IntegrationOrchestrator.swift` - Pass `accountType` to converters

#### UI Layer
- `UI/Properties/Components/TransactionRow.swift` - Use classification for icon/color
- `UI/Properties/Components/TransactionDetailView.swift` - Show classification
- `Ledgers/UI/Components/LedgerTransactionRow.swift` - Use classification, add badge
- `Ledgers/UI/Components/LedgerTransactionDetailView.swift` - Show both classification levels
- `CategoryTemplates/UI/Components/CategorySelectionView.swift` - Filter by classification
- `CategoryTemplates/UI/Components/CategoryManagementView.swift` - Update types/names
- `CategoryTemplates/UI/Components/CategoryTemplateManagementSplitView.swift` - Update types/names
- `Ledgers/Domain/Services/LedgerFilterService.swift` - Use classification for filtering
- `Ledgers/UI/Components/LedgerHeaderCard.swift` - Show neutral transactions in stats
- `UI/Shared/Components/ClassificationBadge.swift` - NEW: Badge component
- `UI/Shared/Components/DetailRow.swift` - NEW: Enhanced detail row with icon/color

#### Data Migration
- **No migration needed**: Classification is a **required field** for all transactions. User will delete and reimport all transactions, so all new transactions will have classification assigned during import.

---

## 6. Benefits

1. **Clear Separation**: First-level classification is automatic and account-aware; second-level is user-manageable
2. **Better Filtering**: Can filter by first-level classification (e.g., "show only expenses")
3. **Account Type Awareness**: Credit cards and loans correctly handle their unique characteristics
4. **Improved Reporting**: Statistics can be broken down by first-level classification
5. **Better Naming**: "CategoryScheme" is clearer than "CategoryTemplate"

---

## 7. UI Updates

### Overview
The UI needs to be updated to:
1. **Display first-level classification** in transaction rows and detail views
2. **Use classification** instead of amount sign for icons/colors
3. **Respect classification** in category selection (filter available categories)
4. **Show both levels** of categorization clearly
5. **Update filtering** to use classification

### 7.1 Transaction Row Updates

#### Properties View (`TransactionRow.swift`)
**Current**: Uses `tx.amount < 0` to determine icon/color  
**Update**: Use `tx.classification` instead

```swift
// UI/Properties/Components/TransactionRow.swift

struct TransactionRow: View {
    let tx: Transaction
    // ... existing properties ...
    
    var body: some View {
        Button(action: { showDetail = true }) {
            HStack(alignment: .top, spacing: 12) {
                // UPDATED: Use classification instead of amount sign
                ZStack {
                    Circle()
                        .fill(classificationBackgroundColor)
                        .frame(width: 40, height: 40)
                    Image(systemName: tx.classification.systemImage)
                        .font(.system(size: 18))
                        .foregroundStyle(classificationForegroundColor)
                }
                
                // ... rest of the row ...
            }
        }
    }
    
    // NEW: Classification-based colors
    private var classificationBackgroundColor: Color {
        switch tx.classification {
        case .expense: return Color.red.opacity(0.15)
        case .income: return Color.green.opacity(0.15)
        case .neutral: return Color.blue.opacity(0.15)
        }
    }
    
    private var classificationForegroundColor: Color {
        switch tx.classification {
        case .expense: return .red
        case .income: return .green
        case .neutral: return .blue
        }
    }
}
```

#### Ledger Transaction Row (`LedgerTransactionRow.swift`)
**Current**: Uses `tx.amount < 0` to determine icon/color  
**Update**: Use `tx.classification` instead, and show classification badge

```swift
// Ledgers/UI/Components/LedgerTransactionRow.swift

struct LedgerTransactionRow: View {
    let tx: Transaction
    let category: CategorySchemeCategory?  // Renamed from CategoryTemplateCategory
    // ... existing properties ...
    
    var body: some View {
        Button(action: onRowTap) {
            HStack(alignment: .center, spacing: 14) {
                // UPDATED: Use classification for icon
                ZStack {
                    Circle()
                        .fill(classificationBackgroundColor)
                        .frame(width: 48, height: 48)
                    Image(systemName: tx.classification.systemImage)
                        .font(.system(size: 20, weight: .medium))
                        .foregroundStyle(classificationForegroundColor)
                }
                
                VStack(alignment: .leading, spacing: 6) {
                    Text(tx.merchant)
                        .font(.system(size: 16, weight: .medium))
                    
                    HStack(spacing: 8) {
                        Label(timeString(tx.date), systemImage: "clock")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                        
                        // NEW: Show classification badge
                        ClassificationBadge(classification: tx.classification)
                        
                        if let notes = tx.notes, !notes.isEmpty {
                            Text("•")
                                .foregroundStyle(.secondary.opacity(0.5))
                            Text(notes)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                                .lineLimit(1)
                        }
                        
                        // Second-level category badge (if available)
                        if let category = category, !category.isTopLevel {
                            Text("•")
                                .foregroundStyle(.secondary.opacity(0.5))
                            CategoryBadge(category: category)
                        }
                    }
                }
                
                Spacer()
                
                // Amount (color based on classification)
                HStack(spacing: 8) {
                    VStack(alignment: .trailing, spacing: 2) {
                        Text(amountString(tx.amount))
                            .font(.system(size: 17, weight: .semibold))
                            .foregroundStyle(classificationForegroundColor)
                    }
                    
                    Image(systemName: "chevron.right")
                        .font(.system(size: 12, weight: .medium))
                        .foregroundStyle(.secondary.opacity(isHovered ? 0.8 : 0.4))
                }
            }
            // ... rest of styling ...
        }
    }
    
    // NEW: Classification-based colors
    private var classificationBackgroundColor: Color {
        switch tx.classification {
        case .expense: return Color.red.opacity(0.12)
        case .income: return Color.green.opacity(0.12)
        case .neutral: return Color.blue.opacity(0.12)
        }
    }
    
    private var classificationForegroundColor: Color {
        switch tx.classification {
        case .expense: return .red
        case .income: return .green
        case .neutral: return .blue
        }
    }
}

// NEW: Classification badge component
struct ClassificationBadge: View {
    let classification: TransactionClassification
    
    var body: some View {
        HStack(spacing: 3) {
            Image(systemName: classification.systemImage)
                .font(.system(size: 9, weight: .medium))
            Text(classification.displayName)
                .font(.system(size: 10, weight: .medium))
        }
        .foregroundStyle(classificationColor)
        .padding(.horizontal, 6)
        .padding(.vertical, 3)
        .background(
            Capsule()
                .fill(classificationColor.opacity(0.15))
        )
    }
    
    private var classificationColor: Color {
        switch classification {
        case .expense: return .red
        case .income: return .green
        case .neutral: return .blue
        }
    }
}
```

### 7.2 Transaction Detail View Updates

#### Properties Detail View (`TransactionDetailView.swift`)
**Update**: Show classification field

```swift
// UI/Properties/Components/TransactionDetailView.swift

struct TransactionDetailView: View {
    let transaction: Transaction
    // ... existing properties ...
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    // NEW: Classification field
                    DetailRow(
                        label: "分类",
                        value: transaction.classification.displayName,
                        icon: transaction.classification.systemImage,
                        color: classificationColor
                    )
                    
                    DetailRow(label: "时间", value: dateTimeString(transaction.date))
                    DetailRow(label: "金额", value: amountString(transaction.amount))
                    DetailRow(label: "商户", value: transaction.merchant)
                    // ... other fields ...
                }
                .padding(20)
            }
            .navigationTitle("交易详情")
        }
    }
    
    private var classificationColor: Color {
        switch transaction.classification {
        case .expense: return .red
        case .income: return .green
        case .neutral: return .blue
        }
    }
}
```

#### Ledger Detail View (`LedgerTransactionDetailView.swift`)
**Update**: Show both first-level classification and second-level category

```swift
// Ledgers/UI/Components/LedgerTransactionDetailView.swift

struct LedgerTransactionDetailView: View {
    // ... existing properties ...
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 20) {
                    // UPDATED: Show first-level classification
                    DetailRow(
                        label: "一级分类",
                        value: transaction.classification.displayName,
                        icon: transaction.classification.systemImage,
                        color: classificationColor
                    )
                    
                    // UPDATED: Show second-level category (clickable)
                    NavigationLink {
                        CategorySelectionView(
                            templateId: ledger.categorySchemeId,  // Renamed from categoryTemplateId
                            categories: loadedCategories.isEmpty ? categories : loadedCategories,
                            selectedCategoryId: selectedCategoryId,
                            transactionClassification: transaction.classification,  // NEW: Pass classification
                            onSelect: { categoryId in
                                // ... existing category selection logic ...
                            }
                        )
                    } label: {
                        DetailRow(
                            label: "二级分类",
                            value: categoryDisplayText,
                            isClickable: true
                        )
                    }
                    
                    DetailRow(label: "账本", value: ledger.name)
                    DetailRow(label: "时间", value: dateTimeString(transaction.date))
                    DetailRow(label: "金额", value: amountString(transaction.amount))
                    // ... other fields ...
                }
                .padding(20)
            }
            .navigationTitle("账单详情")
        }
    }
    
    private var classificationColor: Color {
        switch transaction.classification {
        case .expense: return .red
        case .income: return .green
        case .neutral: return .blue
        }
    }
    
    private var categoryDisplayText: String {
        if let category = category, !category.isTopLevel {
            return category.name
        } else {
            return "未分类"
        }
    }
}
```

### 7.3 Category Selection Updates

#### Category Selection View (`CategorySelectionView.swift`)
**Update**: Filter categories based on transaction classification

```swift
// CategoryTemplates/UI/Components/CategorySelectionView.swift

struct CategorySelectionView: View {
    let templateId: UUID
    let categories: [CategorySchemeCategory]  // Renamed from CategoryTemplateCategory
    let selectedCategoryId: UUID?
    let transactionClassification: TransactionClassification  // NEW: Required parameter
    var onSelect: (UUID?) -> Void
    
    @Environment(\.dismiss) private var dismiss
    @State private var selectedTab: CategoryTab
    
    enum CategoryTab: String, CaseIterable {
        case expense = "支出"
        case income = "收入"
        case transfer = "转账"  // Maps to "资金流动" category
        
        var displayName: String { rawValue }
        
        var categoryName: String {
            switch self {
            case .expense: return "支出"
            case .income: return "收入"
            case .transfer: return "资金流动"
            }
        }
        
        // NEW: Map from TransactionClassification
        init?(from classification: TransactionClassification) {
            switch classification {
            case .expense: self = .expense
            case .income: self = .income
            case .neutral: self = .transfer
            }
        }
    }
    
    init(
        templateId: UUID,
        categories: [CategorySchemeCategory],
        selectedCategoryId: UUID?,
        transactionClassification: TransactionClassification,  // NEW: Required
        onSelect: @escaping (UUID?) -> Void
    ) {
        self.templateId = templateId
        self.categories = categories
        self.selectedCategoryId = selectedCategoryId
        self.transactionClassification = transactionClassification
        self.onSelect = onSelect
        
        // Auto-select tab based on classification
        if let tab = CategoryTab(from: transactionClassification) {
            _selectedTab = State(initialValue: tab)
        } else {
            _selectedTab = State(initialValue: .expense)
        }
    }
    
    // UPDATED: Only show tab matching the classification
    private var availableTabs: [CategoryTab] {
        // Only show the tab that matches the transaction's classification
        if let tab = CategoryTab(from: transactionClassification) {
            return [tab]
        }
        return [.expense]  // Fallback
    }
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                // Tabs - only show the matching tab
                if !availableTabs.isEmpty {
                    HStack(spacing: 0) {
                        ForEach(availableTabs, id: \.self) { tab in
                            CategoryTabButton(
                                title: tab.displayName,
                                isSelected: selectedTab == tab,
                                action: {
                                    // Only one tab available, so no switching needed
                                    selectedTab = tab
                                }
                            )
                        }
                    }
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                }
                
                // Category list - filtered by classification
                ScrollView {
                    // ... existing category list logic ...
                    // Categories are already filtered by parent category matching the tab
                }
            }
            .navigationTitle("选择分类")
            // ... rest of view ...
        }
    }
}
```

### 7.4 Category Management Updates

#### Category Management View (`CategoryManagementView.swift`)
**Update**: Update tab names and ensure alignment with classification

```swift
// CategoryTemplates/UI/Components/CategoryManagementView.swift

struct CategoryManagementView: View {
    @ObservedObject var schemeManager: CategorySchemeManager  // Renamed from CategoryTemplateManager
    let selectedScheme: CategoryScheme  // Renamed from CategoryTemplate
    // ... existing properties ...
    
    enum CategoryTab: String, CaseIterable {
        case expense = "支出"
        case income = "收入"
        case transfer = "资金流动"  // Changed from "转账" to match category name
        
        var displayName: String { rawValue }
    }
    
    // ... rest of view remains similar, just with renamed types ...
}
```

### 7.5 Filtering UI Updates

#### Ledger Rules Editor (Future)
**Update**: Add classification filter to ledger rules

```swift
// Ledgers/UI/Components/LedgerRulesEditor.swift (Future)

struct LedgerRulesEditor: View {
    @Binding var rules: LedgerRules
    
    var body: some View {
        Form {
            // NEW: Classification filter
            Section("一级分类筛选") {
                Toggle("包含支出", isOn: Binding(
                    get: { rules.includeExpense },
                    set: { rules.includeExpense = $0 }
                ))
                Toggle("包含收入", isOn: Binding(
                    get: { rules.includeIncome },
                    set: { rules.includeIncome = $0 }
                ))
                Toggle("包含中性", isOn: Binding(
                    get: { rules.includeNeutral },
                    set: { rules.includeNeutral = $0 }
                ))
            }
            
            // ... other rule sections ...
        }
    }
}
```

#### Ledger Rules Model Update
**Update**: Add classification filtering to `LedgerRules`

```swift
// Core/Domain/Models.swift

public struct LedgerRules: Codable, Hashable {
    // ... existing properties ...
    
    // UPDATED: Use classification instead of amount-based logic
    public var includeExpense: Bool  // Include transactions with .expense classification
    public var includeIncome: Bool   // Include transactions with .income classification
    public var includeNeutral: Bool  // Include transactions with .neutral classification
    
    public init(
        // ... existing parameters ...
        includeExpense: Bool = true,
        includeIncome: Bool = true,
        includeNeutral: Bool = true
    ) {
        // ... existing initialization ...
        self.includeExpense = includeExpense
        self.includeIncome = includeIncome
        self.includeNeutral = includeNeutral
    }
}

extension Ledger {
    /// Filter transactions based on ledger rules
    public func filterTransactions(_ transactions: [Transaction], accounts: [Account]) -> [Transaction] {
        var filtered = transactions
        
        // ... existing account/date/amount filters ...
        
        // UPDATED: Classification filtering
        filtered = filtered.filter { tx in
            switch tx.classification {
            case .expense: return rules.includeExpense
            case .income: return rules.includeIncome
            case .neutral: return rules.includeNeutral
            }
        }
        
        return Transaction.sortByDateDescending(filtered)
    }
}
```

### 7.6 Statistics Updates

#### Statistics Calculations
**Update**: Use classification instead of amount sign

```swift
// Core/Domain/Models.swift

extension Ledger {
    public func calculateStatistics(_ transactions: [Transaction], accounts: [Account]) -> LedgerStatistics {
        let filtered = filterTransactions(transactions, accounts: accounts)
        
        // UPDATED: Use classification instead of amount sign
        let income = filtered
            .filter { $0.classification == .income }
            .reduce(Decimal(0)) { $0 + abs($1.amount) }
        
        let expenses = filtered
            .filter { $0.classification == .expense }
            .reduce(Decimal(0)) { $0 + abs($1.amount) }
        
        let neutral = filtered
            .filter { $0.classification == .neutral }
            .reduce(Decimal(0)) { $0 + abs($1.amount) }
        
        let net = income - expenses
        
        return LedgerStatistics(
            totalTransactions: filtered.count,
            totalIncome: income,
            totalExpenses: expenses,
            totalNeutral: neutral,  // NEW: Track neutral transactions
            net: net
        )
    }
}

public struct LedgerStatistics: Hashable {
    public let totalTransactions: Int
    public let totalIncome: Decimal
    public let totalExpenses: Decimal
    public let totalNeutral: Decimal  // NEW
    public let net: Decimal
}
```

### 7.7 Summary of UI File Changes

#### Files to Update:
1. **Transaction Rows**:
   - `UI/Properties/Components/TransactionRow.swift` - Use classification for icon/color
   - `Ledgers/UI/Components/LedgerTransactionRow.swift` - Use classification, add badge

2. **Transaction Detail Views**:
   - `UI/Properties/Components/TransactionDetailView.swift` - Show classification
   - `Ledgers/UI/Components/LedgerTransactionDetailView.swift` - Show both levels

3. **Category Selection**:
   - `CategoryTemplates/UI/Components/CategorySelectionView.swift` - Filter by classification

4. **Category Management**:
   - `CategoryTemplates/UI/Components/CategoryManagementView.swift` - Update types/names
   - `CategoryTemplates/UI/Components/CategoryTemplateManagementSplitView.swift` - Update types/names

5. **Ledger Filtering**:
   - `Core/Domain/Models.swift` - Update `LedgerRules` and filtering logic
   - `Ledgers/Domain/Services/LedgerFilterService.swift` - Use classification

6. **Statistics**:
   - `Core/Domain/Models.swift` - Update `LedgerStatistics`
   - `Ledgers/UI/Components/LedgerHeaderCard.swift` - Show neutral transactions

7. **New Components**:
   - `UI/Shared/Components/ClassificationBadge.swift` - NEW: Badge component
   - `UI/Shared/Components/DetailRow.swift` - NEW: Enhanced detail row with icon/color

---

## 8. Decisions Made

1. ✅ **Loan Payments**: 
   - In loan account: `neutral` (paying debt reduces debt, not expense from loan's perspective)
   - In bank account: `neutral` (money going out)
   - Classification is account-perspective based
2. ✅ **Brokerage Trades**: `neutral` (stock purchases are investments, not consumption)
3. ✅ **Migration Strategy**: No migration needed - classification is a required field. User will delete and reimport all transactions.
4. ✅ **Filtering**: Ledgers can filter by first-level classification in their rules (included in proposal)
5. ✅ **UI Color Scheme**: Neutral transactions use **blue** color
6. ✅ **Payment Gateway**: ALL transactions are `neutral` (intermediate records that will reflect in real accounts)

---

## Next Steps

1. **Review and confirm** account type classification rules
2. **Confirm** renaming from `CategoryTemplate` to `CategoryScheme`
3. **Approve** this proposal
4. **Begin implementation** with Phase 1 (First-Level Classification)

