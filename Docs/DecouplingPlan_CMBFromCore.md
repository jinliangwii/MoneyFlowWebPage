# Decoupling Plan: Removing CMB Coupling from Core Layer

## Executive Summary

This plan addresses **Critical Issue #1** from ArchitectureRedesignOpinions_v3.md: CMB-specific methods (`loadRawCMBTransactions`, `saveRawCMBTransactions`, `loadImportBatches`, `saveImportBatches`) are currently in the Core layer (`DataStoreProtocol` and `DataStore`), violating the Open/Closed Principle and blocking extensibility.

**Goal:** Move all CMB-specific data operations from Core to the CMB feature module.

**Estimated Time:** 1-2 days (as per ArchitectureRedesignOpinions_v3.md Phase 1)

---

## Current Architecture Problems

### 1. Core Layer Violations
- `Core/Data/Protocols.swift`: Contains CMB-specific methods in `DataStoreProtocol`
- `Core/Data/DataStore.swift`: Contains CMB-specific implementations
- `Core/Data/DataStore.swift`: Imports CMB types (`RawCMBTransaction`, `ImportCMBTransactionBatch`)

### 2. Impact
- Adding new account types (ICBC, CCB, etc.) requires modifying `DataStoreProtocol`
- Core layer depends on feature-specific types
- Testing becomes harder (CMB types leak into core tests)
- Violates Open/Closed Principle

### 3. Current Usage
- `Features/CMB/Services/CMBImportService.swift` calls these methods:
  - `dataStore.loadRawCMBTransactions()` (line 130)
  - `dataStore.saveRawCMBTransactions()` (line 185)
  - `dataStore.loadImportBatches()` (line 195)
  - `dataStore.saveImportBatches()` (line 197)

---

## Recommended Solution: CMB Repository Pattern

Based on ArchitectureRedesignOpinions_v3.md, we'll use **Option 2: Repository Pattern** (recommended over extension pattern for better encapsulation).

### Architecture Overview

```
Before:
┌─────────────────┐
│  DataStore      │ ← Contains CMB methods
│  (Core)         │
└─────────────────┘

After:
┌─────────────────┐     ┌──────────────────┐
│  DataStore      │     │  CMBRepository   │
│  (Core)         │◄────│  (CMB Feature)   │
│  - transactions │     │  - rawTransactions│
│  - accounts      │     │  - importBatches │
└─────────────────┘     └──────────────────┘
```

---

## Implementation Plan

### Phase 1: Create CMB Repository (Step 1-3)

#### Step 1: Create CMBRepository Protocol
**File:** `MoneyFlow/Features/CMB/Data/CMBRepository.swift`

```swift
// CMBRepository.swift
import Foundation

/// Repository for CMB-specific data storage
/// Encapsulates all CMB-related persistence logic
protocol CMBRepositoryProtocol {
    func loadRawTransactions() async throws -> [RawCMBTransaction]
    func saveRawTransactions(_ data: [RawCMBTransaction]) async throws
    func loadImportBatches() async throws -> [ImportCMBTransactionBatch]
    func saveImportBatches(_ batches: [ImportCMBTransactionBatch]) async throws
}
```

#### Step 2: Implement CMBRepository
**File:** `MoneyFlow/Features/CMB/Data/CMBRepository.swift`

```swift
/// Concrete implementation of CMBRepositoryProtocol
/// Uses DataStoreProtocol for generic file operations
actor CMBRepository: CMBRepositoryProtocol {
    private let dataStore: DataStoreProtocol
    private let rawCMBFilename = "rawCMBTransactions.json"
    private let importBatchesFilename = "importBatches.json"
    private let encoder: JSONEncoder
    private let decoder: JSONDecoder
    
    init(dataStore: DataStoreProtocol) {
        self.dataStore = dataStore
        self.encoder = JSONEncoder()
        self.encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        self.encoder.dateEncodingStrategy = .iso8601
        self.decoder = JSONDecoder()
        self.decoder.dateDecodingStrategy = .iso8601
    }
    
    private func documentsURL() throws -> URL {
        let urls = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
        guard let url = urls.first else { throw AppError.dataStoreError("无法访问文档目录") }
        return url
    }
    
    private func fileURL(_ name: String) throws -> URL {
        try documentsURL().appendingPathComponent(name)
    }
    
    func loadRawTransactions() async throws -> [RawCMBTransaction] {
        let url = try fileURL(rawCMBFilename)
        guard FileManager.default.fileExists(atPath: url.path) else { return [] }
        let data = try await Task.detached(priority: .utility) {
            try Data(contentsOf: url)
        }.value
        return try decoder.decode([RawCMBTransaction].self, from: data)
    }
    
    func saveRawTransactions(_ data: [RawCMBTransaction]) async throws {
        let jsonData = try encoder.encode(data)
        let url = try fileURL(rawCMBFilename)
        try await Task.detached(priority: .utility) {
            try jsonData.write(to: url, options: [.atomic])
        }.value
    }
    
    func loadImportBatches() async throws -> [ImportCMBTransactionBatch] {
        let url = try fileURL(importBatchesFilename)
        guard FileManager.default.fileExists(atPath: url.path) else { return [] }
        let data = try await Task.detached(priority: .utility) {
            try Data(contentsOf: url)
        }.value
        return try decoder.decode([ImportCMBTransactionBatch].self, from: data)
    }
    
    func saveImportBatches(_ batches: [ImportCMBTransactionBatch]) async throws {
        let jsonData = try encoder.encode(batches)
        let url = try fileURL(importBatchesFilename)
        try await Task.detached(priority: .utility) {
            try jsonData.write(to: url, options: [.atomic])
        }.value
    }
}
```

#### Step 3: Create CMBRepository Factory
**File:** `MoneyFlow/Features/CMB/Data/CMBRepository.swift` (add to same file)

```swift
extension CMBRepository {
    /// Factory method for creating a shared instance
    /// Can be extended later for dependency injection
    static func create(dataStore: DataStoreProtocol) -> CMBRepositoryProtocol {
        CMBRepository(dataStore: dataStore)
    }
}
```

---

### Phase 2: Update CMBImportService (Step 4-5)

#### Step 4: Inject CMBRepository into CMBImportService
**File:** `MoneyFlow/Features/CMB/Services/CMBImportService.swift`

**Changes:**
1. Add `cmbRepository` property
2. Replace all `dataStore.loadRawCMBTransactions()` calls with `cmbRepository.loadRawTransactions()`
3. Replace all `dataStore.saveRawCMBTransactions()` calls with `cmbRepository.saveRawTransactions()`
4. Replace all `dataStore.loadImportBatches()` calls with `cmbRepository.loadImportBatches()`
5. Replace all `dataStore.saveImportBatches()` calls with `cmbRepository.saveImportBatches()`

**Updated Structure:**
```swift
struct CMBImportService {
    private let cmbRepository: CMBRepositoryProtocol
    
    init(cmbRepository: CMBRepositoryProtocol? = nil, dataStore: DataStoreProtocol = DataStore()) {
        self.cmbRepository = cmbRepository ?? CMBRepository.create(dataStore: dataStore)
    }
    
    func importCMBZipPDF(...) async throws -> ImportResult {
        // ... existing code ...
        
        // Replace: let existingRawTransactions = try await dataStore.loadRawCMBTransactions()
        let existingRawTransactions = try await cmbRepository.loadRawTransactions()
        
        // ... more replacements ...
        
        // Replace: try await dataStore.saveRawCMBTransactions(allRawTransactions)
        try await cmbRepository.saveRawTransactions(allRawTransactions)
        
        // Replace: var allBatches = try await dataStore.loadImportBatches()
        var allBatches = try await cmbRepository.loadImportBatches()
        
        // Replace: try await dataStore.saveImportBatches(allBatches)
        try await cmbRepository.saveImportBatches(allBatches)
    }
}
```

#### Step 5: Update CMBImportService.shared
**File:** `MoneyFlow/Features/CMB/Services/CMBImportService.swift`

**Option A: Keep static shared instance (simpler)**
```swift
static var shared: CMBImportService {
    CMBImportService()
}
```

**Option B: Remove static shared (better for testing)**
- Remove `static let shared = CMBImportService()`
- Update all call sites to create instances (via provider or factory)

---

### Phase 3: Remove CMB from Core (Step 6-8)

#### Step 6: Remove CMB Methods from DataStoreProtocol
**File:** `MoneyFlow/Core/Data/Protocols.swift`

**Remove:**
```swift
// Raw CMB transaction data storage
func loadRawCMBTransactions() async throws -> [RawCMBTransaction]
func saveRawCMBTransactions(_ data: [RawCMBTransaction]) async throws

// Import batch tracking
func loadImportBatches() async throws -> [ImportCMBTransactionBatch]
func saveImportBatches(_ batches: [ImportCMBTransactionBatch]) async throws
```

**Result:**
```swift
protocol DataStoreProtocol {
    func loadTransactions() async throws -> [Transaction]
    func saveTransactions(_ transactions: [Transaction]) async throws
    func loadAccounts() async throws -> [Account]
    func saveAccounts(_ accounts: [Account]) async throws
    // ✅ CMB methods removed
}
```

#### Step 7: Remove CMB Methods from DataStore
**File:** `MoneyFlow/Core/Data/DataStore.swift`

**Remove:**
- Lines 8-9: `rawCMBFilename` and `importBatchesFilename` constants
- Lines 67-83: `saveRawCMBTransactions()` and `loadRawCMBTransactions()` methods
- Lines 85-101: `saveImportBatches()` and `loadImportBatches()` methods

**Remove imports** (if any):
- No direct imports needed (types were only in method signatures)

#### Step 8: Remove CMB Type Imports from Core
**Check for any remaining imports:**
- `MoneyFlow/Core/Data/DataStore.swift` should not import CMB types
- Verify no CMB types in Core/Data/Protocols.swift

---

### Phase 4: Update Call Sites (Step 9-10)

#### Step 9: Find and Update All Call Sites

**Search for usages:**
```bash
grep -r "loadRawCMBTransactions\|saveRawCMBTransactions\|loadImportBatches\|saveImportBatches" MoneyFlow/
```

**Expected call sites:**
- ✅ `CMBImportService.swift` - Already updated in Step 4
- ⚠️ Any tests that use these methods
- ⚠️ Any other services that might use them

#### Step 10: Update Tests
**Files to update:**
- `MoneyFlowTests/CMBPDFParserTests.swift` (if it uses DataStore)
- Any integration tests

**Approach:**
1. Create mock `CMBRepositoryProtocol` for tests
2. Or create test repository that uses in-memory storage

---

### Phase 5: Verification & Cleanup (Step 11-12)

#### Step 11: Verify No CMB Types in Core
**Commands:**
```bash
# Verify no CMB imports in Core
grep -r "RawCMBTransaction\|ImportCMBTransactionBatch" MoneyFlow/Core/

# Verify no CMB methods in DataStoreProtocol
grep -r "loadRawCMB\|saveRawCMB\|loadImportBatches\|saveImportBatches" MoneyFlow/Core/
```

**Expected:** No results

#### Step 12: Run Tests
```bash
# Run all tests
# Verify CMB import still works
# Verify no breaking changes
```

---

## Migration Strategy

### Option A: Big Bang (Recommended for small codebase)
- Complete all phases in sequence
- All changes deployed together
- Requires careful testing

### Option B: Gradual Migration (Safer for larger codebase)
1. **Phase 1-2:** Create repository, update CMBImportService (backward compatible)
2. **Keep DataStore methods temporarily** with deprecation warnings
3. **Phase 3:** Remove DataStore methods after verification
4. **Phase 4-5:** Clean up and verify

---

## File Structure After Refactoring

```
MoneyFlow/
├── Core/
│   └── Data/
│       ├── Protocols.swift          ✅ No CMB methods
│       └── DataStore.swift          ✅ No CMB methods
│
└── Features/
    └── CMB/
        ├── Data/                     🆕 New folder
        │   └── CMBRepository.swift   🆕 Repository implementation
        ├── Domain/
        │   ├── RawCMBTransaction.swift  ✅ Stays here
        │   └── ImportCMBTransactionBatch.swift  ✅ Stays here
        └── Services/
            └── CMBImportService.swift  ✅ Uses CMBRepository
```

---

## Testing Checklist

- [ ] Unit test `CMBRepository` (save/load operations)
- [ ] Integration test CMB import flow end-to-end
- [ ] Verify existing data still loads (backward compatibility)
- [ ] Verify no compilation errors in Core layer
- [ ] Verify no CMB types leak into Core
- [ ] Run existing CMB tests

---

## Benefits After Refactoring

### ✅ Separation of Concerns
- Core layer is feature-agnostic
- CMB feature is self-contained

### ✅ Extensibility
- Adding ICBC/CCB doesn't require touching Core
- Each feature manages its own data

### ✅ Testability
- Can mock `CMBRepositoryProtocol` independently
- Core tests don't need CMB types

### ✅ Open/Closed Principle
- Core is closed for modification
- Features are open for extension

---

## Rollback Plan

If issues arise:

1. **Keep both implementations temporarily:**
   - Keep CMBRepository
   - Keep DataStore CMB methods (marked deprecated)
   - Gradually migrate call sites

2. **Full rollback:**
   - Revert commits from Phase 3
   - Restore CMB methods to DataStore
   - CMBImportService can fall back to DataStore

---

## Future Enhancements

### 1. Generic Feature Repository Pattern
After this refactoring, create a reusable pattern:

```swift
protocol FeatureRepository {
    associatedtype RawDataType: Codable
    associatedtype BatchType: Codable
    
    func loadRawData() async throws -> [RawDataType]
    func saveRawData(_ data: [RawDataType]) async throws
    func loadBatches() async throws -> [BatchType]
    func saveBatches(_ batches: [BatchType]) async throws
}
```

### 2. Feature Data Storage Abstraction
```swift
protocol FeatureDataStorage {
    func save<T: Codable>(_ value: T, key: String) async throws
    func load<T: Codable>(_ type: T.Type, key: String) async throws -> T?
}
```

### 3. Dependency Injection
- Inject `CMBRepository` via `CMBAccountFeatureProvider`
- Better testability
- Clearer dependencies

---

## Estimated Timeline

| Phase | Steps | Time Estimate |
|-------|-------|---------------|
| Phase 1 | Step 1-3 | 2-3 hours |
| Phase 2 | Step 4-5 | 1-2 hours |
| Phase 3 | Step 6-8 | 1 hour |
| Phase 4 | Step 9-10 | 1-2 hours |
| Phase 5 | Step 11-12 | 1 hour |
| **Total** | | **6-9 hours** |

---

## Success Criteria

✅ **Core layer is CMB-free:**
- No CMB types in Core/
- No CMB methods in DataStoreProtocol
- No CMB methods in DataStore

✅ **CMB feature is self-contained:**
- All CMB data operations in CMBRepository
- CMBImportService uses CMBRepository

✅ **No breaking changes:**
- Existing data still loads
- Import functionality works
- Tests pass

✅ **Future extensibility:**
- Can add ICBC/CCB without touching Core
- Pattern is reusable for other features

---

## Next Steps

1. **Review this plan** with the team
2. **Create implementation branch:** `refactor/cmb-repository-decoupling`
3. **Execute Phase 1-2** (backward compatible)
4. **Test thoroughly**
5. **Execute Phase 3** (remove from Core)
6. **Final verification**
7. **Merge**

---

## Questions & Decisions

### Q1: Should we keep `CMBImportService.shared`?
**Decision:** Remove static shared instance for better testability. Update call sites to create instances via provider.

### Q2: Should CMBRepository be a singleton?
**Decision:** No. Create per-instance for better testability. Can be managed by provider/factory.

### Q3: Should we use generic file storage or feature-specific?
**Decision:** Feature-specific for now. Can abstract later if pattern repeats.

---

## References

- **ArchitectureRedesignOpinions_v3.md** - Section 2, Issue 1
- **DataFlowVisualSummary.md** - Current data flow understanding
- **Open/Closed Principle** - SOLID principles
- **Repository Pattern** - Domain-Driven Design pattern

