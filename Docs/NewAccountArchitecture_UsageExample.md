# New Account Architecture - Usage Examples

**Date:** 2025-01-XX

---

## Overview

This document provides practical examples of using the new unified data source architecture.

---

## Example 1: Extract Accounts from CMB Credit Card PDF

```swift
// Create data source
let pdfURL = URL(fileURLWithPath: "/path/to/statement.pdf")
let dataSource = CMBCreditCardPDFDataSource(pdfURL: pdfURL)

// Extract accounts
let accounts = try await dataSource.extractAccounts(
    sourceURL: pdfURL,
    parameters: nil
)

// accounts is [AccountMetadata] - typically one account for PDFs
for account in accounts {
    print("Found account: \(account.displayName)")
    print("  Type: \(account.accountType)")
    print("  Balance: \(account.balance ?? 0)")
    print("  Identifier: \(account.accountIdentifier)")
}

// Create Account instance
let moneyFlowAccount = AccountCreator.createAccount(from: accounts[0])
```

---

## Example 2: Extract Accounts from Plaid (Multiple Accounts)

```swift
// Create data source
let plaidService = PlaidService()
let dataSource = PlaidAPIDataSource(plaidService: plaidService)

// Extract accounts (multiple accounts per item)
let accounts = try await dataSource.extractAccounts(
    sourceURL: nil,  // API source, no file URL
    parameters: [
        "accessToken": accessToken
    ]
)

// accounts is [AccountMetadata] - can be multiple accounts
print("Found \(accounts.count) accounts from Plaid")

// Create Account instances for all accounts
let moneyFlowAccounts = AccountCreator.createAccounts(from: accounts)

// User can select which accounts to import
for account in moneyFlowAccounts {
    print("  - \(account.name) (\(account.type))")
}
```

---

## Example 3: Extract Transactions for Specific Account

```swift
// After extracting accounts and user selects one
let selectedAccount = accounts[0]  // User selected this account
let accountId = UUID()  // Existing or new account ID
let importBatchId = UUID()

// Extract transactions for the selected account
let transactions = try await dataSource.extractTransactions(
    forAccountIdentifier: selectedAccount.accountIdentifier,
    sourceURL: pdfURL,
    accountId: accountId,
    importBatchId: importBatchId,
    parameters: nil
)

// transactions is [any RawTransactionProtocol]
print("Extracted \(transactions.count) transactions")
```

---

## Example 4: Using DataSourceFactory

```swift
// Auto-detect and create data source
let fileURL = URL(fileURLWithPath: "/path/to/statement.pdf")

// Detect source type
if let sourceType = DataSourceFactory.detectSourceType(from: fileURL) {
    print("Detected source type: \(sourceType)")
}

// Create data source with additional info
let dataSource = try DataSourceFactory.createDataSource(
    sourceType: .pdf,
    sourceURL: fileURL,
    additionalInfo: [
        "pdfType": "creditCard"  // or "personalChecking", "expressLoan"
    ]
)

// Use the data source
let accounts = try await dataSource.extractAccounts(
    sourceURL: fileURL,
    parameters: nil
)
```

---

## Example 5: Alipay ZIP with Password

```swift
// Create data source
let zipURL = URL(fileURLWithPath: "/path/to/alipay.zip")
let dataSource = AlipayCSVDataSource()

// Extract accounts (requires password)
let accounts = try await dataSource.extractAccounts(
    sourceURL: zipURL,
    parameters: [
        "password": "your-zip-password"
    ]
)

// Create account
let account = AccountCreator.createAccount(from: accounts[0])
```

---

## Example 6: Complete Import Flow

```swift
// Step 1: Create data source
let dataSource = CMBCreditCardPDFDataSource(pdfURL: pdfURL)

// Step 2: Extract accounts
let accountMetadataList = try await dataSource.extractAccounts(
    sourceURL: pdfURL,
    parameters: nil
)

// Step 3: User selects account (or auto-select if only one)
let selectedMetadata = accountMetadataList[0]

// Step 4: Create or get existing account
let accountId: UUID
if let existingAccount = findExistingAccount(by: selectedMetadata.accountIdentifier) {
    accountId = existingAccount.id
    // Update account with new metadata
    let updatedAccount = AccountCreator.updateAccount(
        existingAccount,
        with: selectedMetadata
    )
    // Save updated account
} else {
    // Create new account
    let newAccount = AccountCreator.createAccount(from: selectedMetadata)
    accountId = newAccount.id
    // Save new account
}

// Step 5: Extract transactions
let importBatchId = UUID()
let transactions = try await dataSource.extractTransactions(
    forAccountIdentifier: selectedMetadata.accountIdentifier,
    sourceURL: pdfURL,
    accountId: accountId,
    importBatchId: importBatchId,
    parameters: nil
)

// Step 6: Process transactions (using existing import service)
// The transactions are [any RawTransactionProtocol]
// They can be converted to Transaction using account-specific converters
```

---

## Example 7: Multi-Account Source (Plaid)

```swift
// Plaid can return multiple accounts
let dataSource = PlaidAPIDataSource(plaidService: plaidService)
let accountMetadataList = try await dataSource.extractAccounts(
    sourceURL: nil,
    parameters: ["accessToken": accessToken]
)

// Show user all accounts for selection
for metadata in accountMetadataList {
    print("\(metadata.displayName) - \(metadata.accountType)")
    print("  Balance: \(metadata.balance ?? 0)")
}

// User selects multiple accounts
let selectedAccounts = accountMetadataList.filter { /* user selection */ }

// Create accounts for selected ones
let accounts = AccountCreator.createAccounts(from: selectedAccounts)

// Import transactions for each selected account
for (metadata, account) in zip(selectedAccounts, accounts) {
    let transactions = try await dataSource.extractTransactions(
        forAccountIdentifier: metadata.accountIdentifier,
        sourceURL: nil,
        accountId: account.id,
        importBatchId: UUID(),
        parameters: [
            "accessToken": accessToken,
            "startDate": startDate,
            "endDate": endDate
        ]
    )
    
    // Process transactions for this account
    processTransactions(transactions, for: account)
}
```

---

## Key Benefits

1. **Unified Interface**: All data sources use the same protocol
2. **Multi-Account Support**: One source can contain multiple accounts
3. **Flexible Parameters**: Pass passwords, tokens, date ranges as needed
4. **Type Safety**: Protocol ensures all sources implement required methods
5. **Easy Extension**: Add new data sources by implementing `DataSourceProtocol`

---

## Next Steps

- Integrate into existing import services
- Update UI to use new data sources
- Add account selection UI for multi-account sources
- Create integration tests

