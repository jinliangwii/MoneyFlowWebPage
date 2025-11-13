# New Account Architecture - Quick Reference

**Date:** 2025-01-XX

---

## 🚀 Quick Start

### Extract Accounts from Data Source

```swift
// Create data source
let dataSource = CMBCreditCardPDFDataSource(pdfURL: pdfURL)

// Extract accounts
let accounts = try await dataSource.extractAccounts(
    sourceURL: pdfURL,
    parameters: nil
)

// Create Account instances
let moneyFlowAccounts = AccountCreator.createAccounts(from: accounts)
```

### Extract Transactions

```swift
// Extract transactions for specific account
let transactions = try await dataSource.extractTransactions(
    forAccountIdentifier: accountMetadata.accountIdentifier,
    sourceURL: pdfURL,
    accountId: accountId,
    importBatchId: UUID(),
    parameters: nil
)
```

### Use Factory

```swift
// Auto-detect and create
let dataSource = try DataSourceFactory.createDataSource(
    sourceType: .pdf,
    sourceURL: fileURL,
    additionalInfo: ["pdfType": "creditCard"]
)
```

---

## 📋 Data Source Types

| Type | Class | File Format |
|------|-------|-------------|
| CMB Credit Card | `CMBCreditCardPDFDataSource` | PDF |
| CMB Personal Checking | `CMBPersonalCheckingPDFDataSource` | PDF |
| CMB Express Loan | `CMBExpressLoanPDFDataSource` | PDF |
| Plaid | `PlaidAPIDataSource` | API |
| Alipay | `AlipayCSVDataSource` | ZIP+CSV |
| WeChat | `WeChatExcelDataSource` | Excel |

---

## 🔑 Key Protocols

### DataSourceProtocol
```swift
protocol DataSourceProtocol {
    var sourceType: DataSourceType { get }
    func extractAccounts(...) async throws -> [AccountMetadata]
    func extractTransactions(...) async throws -> [any RawTransactionProtocol]
    func getSourceInfo(...) async throws -> SourceInfo
}
```

### AccountTraits
```swift
protocol AccountTraits {
    func calculateBalance(...) -> Decimal
    func validateBalance(_ balance: Decimal) -> Bool
    func shouldIncludeInNetWorth() -> Bool
    func getAccountFields() -> [String: Any]
}
```

---

## 📦 Key Types

### AccountMetadata
- `accountType: AccountType`
- `institution: String`
- `accountNumber: String?`
- `currencyCode: String`
- `balance: Decimal?`
- `accountIdentifier: String` (unique within source)

### SourceInfo
- `fileName: String`
- `sourceType: DataSourceType`
- `totalAccounts: Int`
- `startDate: Date?`
- `endDate: Date?`
- `totalPages: Int?`

---

## 🛠️ Helper Functions

### AccountCreator
```swift
AccountCreator.createAccount(from: AccountMetadata) -> Account
AccountCreator.createAccounts(from: [AccountMetadata]) -> [Account]
AccountCreator.updateAccount(_: Account, with: AccountMetadata) -> Account
```

### DataSourceFactory
```swift
DataSourceFactory.createDataSource(sourceType:sourceURL:additionalInfo:) -> DataSourceProtocol
DataSourceFactory.detectSourceType(from: URL) -> DataSourceType?
```

---

## 📍 File Locations

### Core Files
- `MoneyFlow/Core/Domain/Accounts/AccountTraits.swift`
- `MoneyFlow/Core/Data/Sources/DataSourceProtocol.swift`
- `MoneyFlow/Core/Data/Sources/DataSourceFactory.swift`
- `MoneyFlow/Core/Data/Sources/AccountCreator.swift`

### Data Sources
- `MoneyFlow/Core/Data/Sources/CMBCreditCardPDFDataSource.swift`
- `MoneyFlow/Core/Data/Sources/CMBPersonalCheckingPDFDataSource.swift`
- `MoneyFlow/Core/Data/Sources/CMBExpressLoanPDFDataSource.swift`
- `MoneyFlow/Core/Data/Sources/PlaidAPIDataSource.swift`
- `MoneyFlow/Core/Data/Sources/AlipayCSVDataSource.swift`
- `MoneyFlow/Core/Data/Sources/WeChatExcelDataSource.swift`

---

## 🔗 Related Documentation

- **Full Architecture**: `NewAccountArchitecture_Final.md`
- **Usage Examples**: `NewAccountArchitecture_UsageExample.md`
- **Integration Guide**: `NewAccountArchitecture_IntegrationGuide.md`
- **Complete Summary**: `NewAccountArchitecture_Complete.md`

---

## 💡 Common Patterns

### Single Account Source
```swift
let accounts = try await dataSource.extractAccounts(...)
let account = AccountCreator.createAccount(from: accounts[0])
```

### Multi-Account Source
```swift
let accounts = try await dataSource.extractAccounts(...)
// Show selection UI
let selected = accounts.filter { /* user selection */ }
let moneyFlowAccounts = AccountCreator.createAccounts(from: selected)
```

### With Parameters
```swift
// Alipay (password)
let accounts = try await dataSource.extractAccounts(
    sourceURL: zipURL,
    parameters: ["password": "zip-password"]
)

// Plaid (access token)
let accounts = try await dataSource.extractAccounts(
    sourceURL: nil,
    parameters: ["accessToken": token]
)
```

---

**Quick Reference - For detailed information, see full documentation files.**

