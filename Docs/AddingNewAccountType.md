# Adding a New Account Type Guide

This guide documents the process of adding a new account type to MoneyFlow, using WeChat Payment Gateway Account as an example.

## Overview

Adding a new account type requires:
1. Domain models (transaction data structures)
2. Data layer (repository for persistence)
3. Services (parsing and import logic)
4. Providers (account creation and feature providers)
5. Registration (register in ProviderRegistry and AccountFeatureRegistry)
6. Optional: Logo and debug pane

## Step-by-Step Process

### 1. Create Folder Structure

Create the following folder structure under `MoneyFlow/Accounts/{AccountName}/`:

```
Accounts/{AccountName}/
├── Domain/
│   ├── Raw{AccountName}Transaction.swift
│   └── Import{AccountName}TransactionBatch.swift
├── Data/
│   └── {AccountName}Repository.swift
├── Services/
│   ├── {AccountName}Parser.swift (or {AccountName}ExcelParser.swift, etc.)
│   └── {AccountName}ImportService.swift
├── Provider/
│   ├── {AccountName}AccountProvider.swift
│   └── {AccountName}AccountFeatureProvider.swift
└── UI/                    # Optional: Account-specific UI
    └── {AccountName}Views.swift
```

**Example**: For WeChat, created `Accounts/WeChat/` with subfolders.

### 2. Create Domain Models

#### `Raw{AccountName}Transaction.swift`
- Model for raw transaction data extracted from source files
- Preserves original format-specific fields as strings
- Includes parsed fields (date, amount, income/expense flags)
- Follows pattern from `RawAlipayTransaction.swift` or `RawCMBTransaction.swift`

#### `Import{AccountName}TransactionBatch.swift`
- Tracks import batches
- Stores metadata: source file name, dates, counts, etc.
- Follows pattern from `ImportAlipayTransactionBatch.swift`

### 3. Create Data Layer

#### `{AccountName}Repository.swift`
- Protocol: `{AccountName}RepositoryProtocol`
- Implementation: `{AccountName}Repository` (actor)
- Methods:
  - `loadRawTransactions()` / `saveRawTransactions()`
  - `loadImportBatches()` / `saveImportBatches()`
  - `saveAccountMetadata()` / `loadAccountMetadata()`
- Follows pattern from `AlipayRepository.swift` or `CMBRepository.swift`

### 4. Create Services

#### `{AccountName}Parser.swift` (or similar)
- Parses source files (CSV, Excel, PDF, etc.)
- Extracts metadata (account info, date ranges, summaries)
- Parses transactions into `Raw{AccountName}Transaction`
- Example patterns:
  - CSV: `AlipayCSVParser.swift`
  - Excel: `WeChatExcelParser.swift` (extracts ZIP, parses XML)
  - PDF: `CMBPDFParser.swift`

#### `{AccountName}ImportService.swift`
- Implements import logic
- Parameters struct: `{AccountName}ImportParameters: AccountImportParameters`
- Main method: `import{AccountName}(...) async throws -> ImportResult`
- Steps:
  1. Parse source file
  2. Verify account metadata (email/nickname match)
  3. Parse transactions
  4. Filter invalid transactions
  5. Detect duplicates
  6. Convert to `Transaction` objects
  7. Save data
- Follows pattern from `AlipayImportService.swift`

### 5. Create Providers

#### `{AccountName}AccountProvider.swift`
- Implements `AccountProvider` protocol
- Properties: `id`, `name`, `displayName`, `icon`, `accountType`, `isDIY`
- Methods:
  - `makeConfigurationView()`: Account creation UI
  - `validate()`: Validate account configuration
  - `createAccount()`: Create `Account` from draft
- Follows pattern from `AlipayPaymentGateAccountProvider.swift`

#### `{AccountName}AccountFeatureProvider.swift`
- Implements `AccountFeatureProvider` and `AccountImportProvider`
- Properties: `id`, `supportedInstitution`
- Methods:
  - `supports(account:)`: Check if provider supports account
  - `makeActionButtons()`: Import button UI
  - `makeImportView()`: Import sheet UI
  - `performImport()`: Execute import using `{AccountName}ImportService`
- Follows pattern from `AlipayPaymentGateAccountFeatureProvider.swift`

### 6. Register Providers

#### In `Features/Providers/ProviderRegistry.swift`
Add to `registerProviders()`:
```swift
register({AccountName}AccountProvider(), for: .{accountType})
```

#### In `Features/Providers/AccountFeatureProvider.swift`
Add to `AccountFeatureRegistry.registerProviders()`:
```swift
let {accountName}Provider = {AccountName}AccountFeatureProvider()
register({accountName}Provider)
```

### 7. Optional: Add Logo

1. **Convert logo image**:
   ```bash
   python3 convert_logo.py {AccountName}Logo ~/Downloads/{logo_file}.png 44
   ```

2. **Update `InstitutionIcons.swift`**:
   - Add to `customLogoIdentifiers`: `["...", "{AccountName}Logo"]`
   - Add detection in `symbol()`: Check for institution name keywords
   - Add color in `color()`: Use official brand color

### 8. Optional: Add Debug Pane

Create `{AccountName}AutoImportView.swift` in `UI/Debug/`:
- Auto-detect source files from test data directory
- Allow file selection
- Find or create account automatically
- Import transactions with progress/error reporting
- Follows pattern from `AlipayAutoImportView.swift`

Add to `DebugPaneView.swift`:
```swift
{AccountName}AutoImportView()
```

## Key Patterns

### Account Type Detection
- Payment Gateway accounts: `type == .paymentGate`
- Store source identifier in `account.metadata["source"]`
- Use institution name for matching

### Import Parameters
- Create `{AccountName}ImportParameters: AccountImportParameters`
- Contains source file URL and any required credentials

### Duplicate Detection
- Use fingerprint: `date + amount + orderId + accountId`
- Check both raw transactions and existing transactions
- Compare by date, amount, and order ID in notes

### Transaction Conversion
- Income: positive amount
- Expense: negative amount
- Neutral: determine direction from transaction type/category
- Payment Gateway accounts: always `balance = 0`

### Sequence Numbers
- Track per-second to handle same-time transactions
- Increment for each transaction at same time
- Preserve order within same day

## File Templates

See existing implementations:
- **Alipay** (CSV/ZIP): `Accounts/Alipay/`
- **WeChat** (Excel): `Accounts/WeChat/`
- **CMB** (PDF/ZIP): `Accounts/CMB/`

## Checklist

- [ ] Create folder structure
- [ ] Domain models (RawTransaction, ImportBatch)
- [ ] Repository (protocol + implementation)
- [ ] Parser service
- [ ] Import service
- [ ] Account provider
- [ ] Feature provider
- [ ] Register in ProviderRegistry
- [ ] Register in AccountFeatureRegistry
- [ ] (Optional) Add logo
- [ ] (Optional) Add debug pane
- [ ] Test account creation
- [ ] Test import functionality

