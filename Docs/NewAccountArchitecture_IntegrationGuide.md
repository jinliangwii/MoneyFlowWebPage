# New Account Architecture - Integration Guide

**Date:** 2025-01-XX

---

## Overview

This guide explains how to integrate the new data source architecture into existing import services and UI.

---

## Integration Strategy

### Option 1: Gradual Migration (Recommended)

Keep existing import services working while gradually adopting new data sources:

1. **New Features**: Use new data sources for new features
2. **One Service at a Time**: Migrate import services one by one
3. **Parallel Support**: Support both old and new paths during transition
4. **Remove Old Code**: Once all services migrated, remove old code

### Option 2: Full Replacement

Replace all import services at once (higher risk, faster completion).

---

## Integration Steps

### Step 1: Update Import Service to Support Data Source

```swift
// Add extension to existing import service
extension CMBCreditCardImportService {
    func importFromDataSource(
        dataSource: CMBCreditCardPDFDataSource,
        accountId: UUID,
        onProgress: @escaping (ImportProgress) -> Void
    ) async throws -> ImportResult {
        // Extract accounts
        let accounts = try await dataSource.extractAccounts(
            sourceURL: pdfURL,
            parameters: nil
        )
        
        // Extract transactions
        let importBatchId = UUID()
        let rawTransactions = try await dataSource.extractTransactions(
            forAccountIdentifier: accounts[0].accountIdentifier,
            sourceURL: pdfURL,
            accountId: accountId,
            importBatchId: importBatchId,
            parameters: nil
        )
        
        // Convert to service-specific raw transaction type
        let cmbTransactions = rawTransactions.compactMap { $0 as? RawCMBCreditCardTransaction }
        
        // Continue with existing import logic...
        // (filter, deduplicate, convert, save)
    }
}
```

### Step 2: Update UI to Use Data Source

```swift
// In account import configuration view
func importAccount() {
    Task {
        // Create data source
        let dataSource = CMBCreditCardPDFDataSource(pdfURL: selectedFileURL)
        
        // Extract accounts
        let accounts = try await dataSource.extractAccounts(
            sourceURL: selectedFileURL,
            parameters: nil
        )
        
        // For single-account sources, auto-select
        // For multi-account sources, show selection UI
        if accounts.count == 1 {
            await importAccount(accounts[0])
        } else {
            // Show account selection UI
            await showAccountSelection(accounts)
        }
    }
}

func importAccount(_ metadata: AccountMetadata) async {
    // Create or update account
    let account: Account
    if let existing = findExistingAccount(by: metadata.accountIdentifier) {
        account = AccountCreator.updateAccount(existing, with: metadata)
    } else {
        account = AccountCreator.createAccount(from: metadata)
    }
    
    // Save account
    await saveAccount(account)
    
    // Import transactions
    let dataSource = CMBCreditCardPDFDataSource(pdfURL: selectedFileURL)
    let transactions = try await dataSource.extractTransactions(
        forAccountIdentifier: metadata.accountIdentifier,
        sourceURL: selectedFileURL,
        accountId: account.id,
        importBatchId: UUID(),
        parameters: nil
    )
    
    // Process transactions through import service
    // ...
}
```

### Step 3: Add Multi-Account Selection UI

```swift
// For multi-account sources (e.g., Plaid)
struct AccountSelectionView: View {
    let accounts: [AccountMetadata]
    let onSelect: ([AccountMetadata]) -> Void
    
    @State private var selectedAccounts: Set<UUID> = []
    
    var body: some View {
        List {
            ForEach(accounts) { account in
                AccountSelectionRow(
                    account: account,
                    isSelected: selectedAccounts.contains(account.id),
                    onToggle: {
                        if selectedAccounts.contains(account.id) {
                            selectedAccounts.remove(account.id)
                        } else {
                            selectedAccounts.insert(account.id)
                        }
                    }
                )
            }
        }
        .toolbar {
            Button("Import Selected") {
                let selected = accounts.filter { selectedAccounts.contains($0.id) }
                onSelect(selected)
            }
        }
    }
}
```

---

## Migration Checklist

### For Each Import Service

- [ ] Create data source implementation (if not exists)
- [ ] Add extension method to import service
- [ ] Update UI to use data source for account extraction
- [ ] Add account selection UI (if multi-account)
- [ ] Test with existing data
- [ ] Update error handling
- [ ] Update progress reporting
- [ ] Remove old code (after all services migrated)

### For UI Updates

- [ ] Update file picker to detect source type
- [ ] Add account selection view for multi-account sources
- [ ] Update import flow to use AccountCreator
- [ ] Add loading states for account extraction
- [ ] Update error messages
- [ ] Add success feedback

---

## Example: Full Integration Flow

```swift
// Complete integration example
class AccountImportViewModel: ObservableObject {
    @Published var accounts: [AccountMetadata] = []
    @Published var isExtracting = false
    @Published var error: Error?
    
    func extractAccounts(from fileURL: URL) async {
        isExtracting = true
        error = nil
        
        do {
            // Detect source type
            guard let sourceType = DataSourceFactory.detectSourceType(from: fileURL) else {
                throw DataSourceError.unsupportedFormat("Unknown file type")
            }
            
            // Create data source
            let dataSource = try DataSourceFactory.createDataSource(
                sourceType: sourceType,
                sourceURL: fileURL,
                additionalInfo: [
                    "pdfType": detectPDFType(from: fileURL) // if PDF
                ]
            )
            
            // Extract accounts
            let extractedAccounts = try await dataSource.extractAccounts(
                sourceURL: fileURL,
                parameters: getParameters(for: sourceType)
            )
            
            await MainActor.run {
                self.accounts = extractedAccounts
                self.isExtracting = false
            }
        } catch {
            await MainActor.run {
                self.error = error
                self.isExtracting = false
            }
        }
    }
    
    func importSelectedAccounts(_ selected: [AccountMetadata]) async {
        for metadata in selected {
            // Create account
            let account = AccountCreator.createAccount(from: metadata)
            await saveAccount(account)
            
            // Import transactions
            await importTransactions(for: account, metadata: metadata)
        }
    }
    
    private func getParameters(for sourceType: DataSourceType) -> [String: Any]? {
        switch sourceType {
        case .csv:
            return ["password": zipPassword] // For Alipay
        case .plaid:
            return ["accessToken": plaidAccessToken]
        default:
            return nil
        }
    }
}
```

---

## Benefits of Integration

1. **Unified Code Path**: All imports use same architecture
2. **Multi-Account Support**: Natural support for multiple accounts
3. **Better UX**: Account selection before import
4. **Easier Testing**: Protocol-based design is easier to test
5. **Future-Proof**: Easy to add new data sources

---

## Testing Strategy

1. **Unit Tests**: Test data sources in isolation
2. **Integration Tests**: Test full import flow
3. **UI Tests**: Test account selection and import
4. **Regression Tests**: Ensure existing imports still work

---

## Rollback Plan

If issues arise:
1. Keep old import services intact
2. Feature flag to switch between old/new
3. Gradual rollout (percentage of users)
4. Monitor error rates

---

## Timeline Estimate

- **Per Import Service**: 4-6 hours
- **UI Updates**: 8-10 hours
- **Testing**: 4-6 hours
- **Total**: 16-22 hours for full integration

---

## Next Steps

1. Start with one import service (e.g., CMB Credit Card)
2. Test thoroughly
3. Roll out to other services
4. Update UI for multi-account support
5. Remove old code

