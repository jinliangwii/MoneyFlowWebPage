# ImportBatch Consolidation - Final Plan

## Decision: Non-Generic Struct (Simplest Approach)

**Why?** All accounts use the same structure - no need for generics or metadata structs.

## Final Structure

```
DataSources/Shared/Models/
  └── ImportBatch.swift                    (Simple non-generic struct)
```

**Location**: `DataSources/Shared/Models/` - This is a shared model used by DataSources repositories, not by the Integration layer.

**Just one file!** Maximum simplicity.

## Implementation

### 1. Create Non-Generic ImportBatch

```swift
// Integration/Domain/ImportBatch.swift
import Foundation

/// Tracks information about each import batch
/// All accounts use the same structure
public struct ImportBatch: Identifiable, Codable, Hashable {
    public let id: UUID
    public let sourceFileName: String
    public let importedAt: Date
    public let accountId: UUID
    public let totalRawRecords: Int
    public let successfulImports: Int
    public let duplicateCount: Int
    
    public init(
        id: UUID = UUID(),
        sourceFileName: String,
        importedAt: Date = Date(),
        accountId: UUID,
        totalRawRecords: Int,
        successfulImports: Int,
        duplicateCount: Int
    ) {
        self.id = id
        self.sourceFileName = sourceFileName
        self.importedAt = importedAt
        self.accountId = accountId
        self.totalRawRecords = totalRawRecords
        self.successfulImports = successfulImports
        self.duplicateCount = duplicateCount
    }
}
```

## Repository Updates

### Before
```swift
protocol CMBCreditCardRepositoryProtocol:
    ImportBatchRepository<ImportCMBCreditCardTransactionBatch>

protocol WeChatRepositoryProtocol:
    ImportBatchRepository<ImportWeChatTransactionBatch>
```

### After
```swift
// All accounts use the same type
protocol CMBCreditCardRepositoryProtocol:
    ImportBatchRepository<ImportBatch>

protocol WeChatRepositoryProtocol:
    ImportBatchRepository<ImportBatch>
```

## Batch Creation

### Before (WeChat/Alipay with dates)
```swift
let batch = ImportWeChatTransactionBatch(
    accountId: accountId,
    sourceFileName: fileName,
    totalRawRecords: total,
    successfulImports: success,
    duplicateCount: duplicates,
    excelStartDate: excelInfo?.startDate,  // Remove this
    excelEndDate: excelInfo?.endDate       // Remove this
)
```

### After (All accounts)
```swift
let batch = ImportBatch(
    accountId: accountId,
    sourceFileName: fileName,
    totalRawRecords: total,
    successfulImports: success,
    duplicateCount: duplicates
)
```

## Implementation Checklist

- [x] Create `ImportBatch.swift` (non-generic struct) ✅
- [x] Update 5 repository protocols ✅
- [x] Update 5 repository implementations ✅
- [x] Delete 5 old struct files ✅
- [x] Remove empty account folders ✅
- [x] Build and verify ✅

**Status: COMPLETE** ✅

## Files to Update

### Create
- `DataSources/Shared/Models/ImportBatch.swift`

### Update (5 repositories)
- `DataSources/.../CMBCreditCard/Repositories/CMBCreditCardRepository.swift`
- `DataSources/.../CMBPersonalChecking/Repositories/CMBPersonalCheckingRepository.swift`
- `DataSources/.../CMBExpressLoan/Repositories/CMBExpressLoanRepository.swift`
- `DataSources/.../WeChat/Repositories/WeChatRepository.swift`
- `DataSources/.../Alipay/Repositories/Alipay3rdPartyTxRepository.swift`

### Delete (5 old files)
- `Integration/Domain/CMBCreditCard/ImportCMBCreditCardTransactionBatch.swift`
- `Integration/Domain/CMBPersonalChecking/ImportCMBPersonalCheckingTransactionBatch.swift`
- `Integration/Domain/CMBExpressLoan/ImportCMBExpressLoanTransactionBatch.swift`
- `Integration/Domain/WeChat/ImportWeChatTransactionBatch.swift`
- `Integration/Domain/Alipay3rdPartyTx/ImportAlipay3rdPartyTxTransactionBatch.swift`

### Move
- `Integration/Domain/ImportBatch.swift` → `DataSources/Shared/Models/ImportBatch.swift`

## Benefits

1. ✅ **Simplest Possible** - No generics, no metadata structs
2. ✅ **One File** - Just `ImportBatch.swift`
3. ✅ **Clean API** - `ImportBatch(...)` instead of `ImportBatch<EmptyImportBatchMetadata>(...)`
4. ✅ **Hashable by Default** - No conditional conformance
5. ✅ **~85% Code Reduction** - 5 structs → 1 struct

## Why This Approach?

1. **YAGNI Principle** - Don't add complexity until needed
2. **All Accounts Identical** - No account-specific fields
3. **Future-Proof** - Can add metadata later if needed (optional field or refactor)
4. **Maximum Simplicity** - One struct, one file

## Migration Notes

### JSON Compatibility
- Old JSON files will decode successfully (extra fields ignored)
- New JSON files use simplified structure
- No breaking changes for existing data

### Batch Creation
- Remove `excelStartDate`/`excelEndDate` from WeChat batch creation
- Remove `csvStartDate`/`csvEndDate` from Alipay batch creation
- All accounts use same creation pattern

## Implementation Complete ✅

All tasks completed successfully:
- ✅ Created single non-generic `ImportBatch` struct
- ✅ Moved to `DataSources/Shared/Models/` (correct location - used by DataSources repositories)
- ✅ Updated all 5 repository protocols to use `ImportBatch`
- ✅ Updated all 5 repository implementations
- ✅ Deleted all 5 account-specific batch struct files
- ✅ Removed empty account folders
- ✅ Build verified - no errors

**Result**: Simplified from 5 account-specific structs to 1 unified struct. ~85% code reduction achieved.

**Location Rationale**: `ImportBatch` is only used by DataSources repositories (via `ImportBatchRepository` protocol), not by the Integration orchestrator. Therefore, it belongs in `DataSources/Shared/Models/` as a shared DataSources model, not in `Integration/Domain/`.

