# DataSource Raw Data Repository Proposal

## Current Architecture

**Current Flow:**
1. DataSource extracts raw transactions → returns `[any RawTransactionProtocol]`
2. Import Service receives raw transactions
3. Import Service converts to service-specific types
4. Import Service saves using repository via `getRawRepository()`

**Current Issues:**
- Import services still need to know about repositories
- DataSources extract but don't persist
- Separation of concerns could be clearer

## Proposed Architecture

**New Flow:**
1. DataSource extracts raw transactions
2. DataSource saves raw transactions to repository (if provided)
3. Import Service receives already-saved raw transactions (or just IDs)
4. Import Service focuses on business logic (duplicate detection, conversion, etc.)

## Benefits

1. **Clearer Separation:**
   - DataSources: Extract + Store raw data
   - Import Services: Business logic (duplicate detection, conversion, balance calculation)

2. **Simpler Import Services:**
   - Don't need to manage repositories
   - Just focus on processing already-extracted data

3. **Better Encapsulation:**
   - DataSources own the entire data extraction and storage pipeline
   - Import services don't need to know about storage details

4. **Easier Testing:**
   - Can test data sources independently (extraction + storage)
   - Can test import services with pre-saved data

## Implementation Options

### Option 1: DataSource with Repository (Recommended)

```swift
protocol DataSourceProtocol {
    // ... existing methods ...
    
    /// Extract and save raw transactions
    /// - Returns: IDs or references to saved transactions
    func extractAndSaveTransactions(
        forAccountIdentifier accountIdentifier: String,
        sourceURL: URL?,
        accountId: UUID,
        importBatchId: UUID,
        repository: some RawTransactionRepositoryProtocol, // NEW
        parameters: [String: Any]?
    ) async throws -> [TransactionReference] // NEW: Reference to saved data
}
```

**Pros:**
- Clear ownership: DataSource handles extraction + storage
- Import services become simpler
- Better separation of concerns

**Cons:**
- DataSource needs to know about repository protocol
- Need to handle different raw transaction types

### Option 2: DataSource with Storage Callback

```swift
protocol DataSourceProtocol {
    // ... existing methods ...
    
    /// Extract transactions with optional storage callback
    func extractTransactions(
        forAccountIdentifier accountIdentifier: String,
        sourceURL: URL?,
        accountId: UUID,
        importBatchId: UUID,
        parameters: [String: Any]?,
        onExtract: ((any RawTransactionProtocol) async throws -> Void)? = nil // NEW
    ) async throws -> [any RawTransactionProtocol]
}
```

**Pros:**
- Flexible: DataSource can save or not
- Doesn't require repository in protocol

**Cons:**
- Less clear ownership
- Callback pattern can be complex

### Option 3: DataSource Returns Saved References

```swift
protocol DataSourceProtocol {
    // ... existing methods ...
    
    /// Extract and optionally save transactions
    /// Returns saved transaction references if storage is provided
    func extractTransactions(
        forAccountIdentifier accountIdentifier: String,
        sourceURL: URL?,
        accountId: UUID,
        importBatchId: UUID,
        parameters: [String: Any]?,
        storage: RawTransactionStorage? = nil // NEW: Optional storage
    ) async throws -> DataSourceExtractionResult // NEW: Contains transactions + save status
}
```

**Pros:**
- Optional storage (backward compatible)
- Clear return type

**Cons:**
- More complex return type
- Still need to handle storage in DataSource

## Recommended Approach: Option 1 with Type-Specific DataSources

Since each DataSource is already account-specific, we can make them handle their own repository:

```swift
// CMBCreditCardPDFDataSource.swift
public actor CMBCreditCardPDFDataSource: DataSourceProtocol {
    private let repository: (any CMBCreditCardRepositoryProtocol)?
    
    public init(
        sourceURL: URL,
        repository: (any CMBCreditCardRepositoryProtocol)? = nil
    ) {
        // ...
        self.repository = repository
    }
    
    public func extractAndSaveTransactions(
        forAccountIdentifier accountIdentifier: String,
        sourceURL: URL?,
        accountId: UUID,
        importBatchId: UUID,
        parameters: [String: Any]?
    ) async throws -> [RawCMBCreditCardTransaction] {
        // Extract
        let rawTransactions = try await extractTransactions(...)
        
        // Save if repository provided
        if let repository = repository {
            try await repository.saveRawTransactions(rawTransactions)
        }
        
        return rawTransactions
    }
}
```

## Migration Plan

1. **Add repository parameter to DataSource initializers** (optional for backward compatibility)
2. **Add `extractAndSaveTransactions` method** to DataSourceProtocol
3. **Update Import Services** to use new method and remove repository management
4. **Update UI views** to pass repositories to DataSources
5. **Remove `getRawRepository()` from Import Services**

## Benefits Summary

✅ **DataSources own data extraction AND storage**
✅ **Import Services focus on business logic only**
✅ **Clearer separation of concerns**
✅ **Easier to test and maintain**
✅ **Better encapsulation**

