# Dependency Injection Inconsistencies Analysis

## Overview

This document analyzes the inconsistent dependency injection (DI) patterns across the MoneyFlow codebase and provides recommendations for standardization.

---

## Current State: Three Different DI Patterns

The codebase currently uses **three different DI patterns** across different services, which creates inconsistency and makes testing harder.

---

## Pattern 1: Optional Parameters with Nil Coalescing (Most Common)

**Used in:** Most import services, `AppState`, `PlaidTransactionSyncService`

### Example 1: CMBCreditCardImportService
```swift
struct CMBCreditCardImportService {
    private let cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol
    private let filePersistence: FilePersistenceProtocol
    private let helper = BaseImportServiceHelper()  // ⚠️ Created directly, not injected
    
    init(
        cmbCreditCardRepository: (any CMBCreditCardRepositoryProtocol)? = nil,
        filePersistence: FilePersistenceProtocol? = nil
    ) {
        self.cmbCreditCardRepository = cmbCreditCardRepository ?? CMBCreditCardRepository()
        self.filePersistence = filePersistence ?? FilePersistence()
    }
}
```

**Issues:**
1. ✅ Allows dependency injection (good for testing)
2. ⚠️ Creates concrete instances if not provided (tight coupling)
3. ⚠️ `helper` is created directly, not injected
4. ⚠️ Hard to see all dependencies at a glance

### Example 2: AppState
```swift
@MainActor
final class AppState: ObservableObject {
    private let transactionRepository: TransactionRepositoryProtocol
    private let accountRepository: AccountRepositoryProtocol
    private let analytics: AnalyticsProviding
    private let dataLifecycleManager: DataLifecycleManager?
    
    init(
        transactionRepository: TransactionRepositoryProtocol? = nil,
        accountRepository: AccountRepositoryProtocol? = nil,
        analytics: AnalyticsProviding? = nil,
        dataLifecycleManager: DataLifecycleManager? = nil
    ) {
        // Create concrete instances if protocols not provided
        self.transactionRepository = transactionRepository ?? FileTransactionRepository()
        self.accountRepository = accountRepository ?? FileAccountRepository()
        self.analytics = analytics ?? AnalyticsService()
        
        // Special handling for DataLifecycleManager
        if let providedManager = dataLifecycleManager {
            self.dataLifecycleManager = providedManager
        } else {
            do {
                self.dataLifecycleManager = try DataLifecycleManager()
            } catch {
                print("⚠️ Failed to initialize DataLifecycleManager: \(error.localizedDescription)")
                self.dataLifecycleManager = nil
            }
        }
    }
}
```

**Issues:**
1. ✅ Allows dependency injection
2. ⚠️ Inconsistent pattern: some dependencies use `??`, others use `if let`
3. ⚠️ `DataLifecycleManager` has special error handling that's not obvious
4. ⚠️ Creates dependencies internally, making it hard to test

### Example 3: PlaidTransactionSyncService
```swift
struct PlaidTransactionSyncService {
    private let plaidService: PlaidServiceProtocol
    private let tokenManager: PlaidTokenManagerProtocol
    private let repository: PlaidRepositoryProtocol
    
    init(
        plaidService: PlaidServiceProtocol? = nil,
        tokenManager: PlaidTokenManagerProtocol? = nil,
        repository: PlaidRepositoryProtocol? = nil
    ) {
        self.plaidService = plaidService ?? PlaidService()
        self.tokenManager = tokenManager ?? PlaidTokenManager()
        self.repository = repository ?? PlaidRepository()
    }
}
```

**Same pattern as Example 1** - consistent within this service, but inconsistent with other patterns.

---

## Pattern 2: Required Parameters (Best for Testing)

**Used in:** `FileTransactionRepository`, `FileAccountRepository`

### Example: FileTransactionRepository
```swift
actor FileTransactionRepository: TransactionRepositoryProtocol {
    private let filePersistence: FilePersistenceProtocol
    
    init(filePersistence: FilePersistenceProtocol? = nil) {
        self.filePersistence = filePersistence ?? FilePersistence()
    }
    // ...
}
```

**Wait, this is actually Pattern 1!** Even repositories use optional parameters.

**Better example - what it SHOULD be:**
```swift
// Current (Pattern 1):
init(filePersistence: FilePersistenceProtocol? = nil) {
    self.filePersistence = filePersistence ?? FilePersistence()
}

// Better (Pattern 2 - Required):
init(filePersistence: FilePersistenceProtocol) {
    self.filePersistence = filePersistence
}
```

---

## Pattern 3: No DI - Direct Instantiation

**Used in:** Repositories that create their own dependencies

### Example: CMBCreditCardRepository
```swift
actor CMBCreditCardRepository: CMBCreditCardRepositoryProtocol {
    private let encoder: JSONEncoder
    private let decoder: JSONDecoder
    
    init() {  // ⚠️ No parameters - creates everything internally
        encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        encoder.dateEncodingStrategy = .iso8601
        decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
    }
    // ...
}
```

**Issues:**
1. ❌ Cannot inject dependencies for testing
2. ❌ Cannot swap implementations (e.g., different encoder settings)
3. ❌ Hard to mock in tests
4. ❌ Tight coupling to concrete implementations

### Example: BaseImportServiceHelper
```swift
struct CMBCreditCardImportService {
    private let helper = BaseImportServiceHelper()  // ⚠️ Created directly
    // ...
}
```

**Issues:**
1. ❌ Cannot inject `FilePersistenceProtocol` into helper
2. ❌ Helper creates its own `FilePersistence()` internally
3. ❌ Cannot test with mock persistence

---

## Pattern 4: Configuration-Based (PlaidService)

**Used in:** `PlaidService`

### Example: PlaidService
```swift
struct PlaidService: PlaidServiceProtocol {
    private let baseURL: String
    private let clientId: String
    private let secret: String
    private let environment: PlaidEnvironment
    
    init(
        baseURL: String? = nil,
        clientId: String? = nil,
        secret: String? = nil,
        environment: PlaidEnvironment? = nil
    ) {
        // Use PlaidConfig if available, otherwise use provided values
        self.baseURL = baseURL ?? PlaidConfig.baseURL
        self.clientId = clientId ?? PlaidConfig.clientId
        self.secret = secret ?? PlaidConfig.secret
        self.environment = environment ?? PlaidConfig.environment
    }
}
```

**Issues:**
1. ⚠️ Uses global configuration (`PlaidConfig`) as fallback
2. ⚠️ Mixes configuration values with dependency injection
3. ⚠️ Hard to test with different configurations

---

## Problems Caused by Inconsistency

### 1. **Testing Difficulties**

**Current situation:**
```swift
// To test CMBCreditCardImportService, you need to:
let mockRepository = MockCMBCreditCardRepository()
let mockPersistence = MockFilePersistence()
let service = CMBCreditCardImportService(
    cmbCreditCardRepository: mockRepository,
    filePersistence: mockPersistence
)
// ✅ This works, but...

// The helper is still created directly:
// private let helper = BaseImportServiceHelper()
// ❌ Helper uses real FilePersistence, not mock!
```

**Problem:** Even when you inject mocks, internal dependencies (like `helper`) still use real implementations.

### 2. **Inconsistent API**

Developers have to remember:
- Some services use `? = nil` pattern
- Some create dependencies directly
- Some use global configuration
- Some mix patterns

**Example confusion:**
```swift
// Pattern 1: Optional with default
let service1 = CMBCreditCardImportService()  // Uses real dependencies

// Pattern 3: No DI
let repository = CMBCreditCardRepository()  // No way to inject

// Pattern 4: Configuration-based
let plaidService = PlaidService()  // Uses PlaidConfig
```

### 3. **Hidden Dependencies**

**Example:**
```swift
struct CMBCreditCardImportService {
    private let helper = BaseImportServiceHelper()  // ⚠️ Hidden dependency
    // ...
}
```

The `helper` dependency is not visible in the initializer, making it:
- Hard to test
- Hard to understand dependencies
- Hard to swap implementations

### 4. **Tight Coupling**

**Example:**
```swift
// CMBCreditCardRepository creates everything internally
init() {
    encoder = JSONEncoder()  // ❌ Cannot inject different encoder
    decoder = JSONDecoder()  // ❌ Cannot inject different decoder
}
```

Cannot:
- Use different encoder settings in tests
- Use a mock encoder
- Swap implementations

---

## Recommended Standardized Pattern

### Pattern: Required Protocol Parameters

**Principle:** All dependencies should be **required protocol parameters** with no defaults.

### Example: Standardized Import Service

```swift
struct CMBCreditCardImportService {
    private let cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol
    private let filePersistence: FilePersistenceProtocol
    private let helper: BaseImportServiceHelper
    
    // ✅ All dependencies required - no defaults
    init(
        cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol,
        filePersistence: FilePersistenceProtocol,
        helper: BaseImportServiceHelper? = nil  // Optional, but creates with injected dependency
    ) {
        self.cmbCreditCardRepository = cmbCreditCardRepository
        self.filePersistence = filePersistence
        // Helper gets the injected filePersistence
        self.helper = helper ?? BaseImportServiceHelper(filePersistence: filePersistence)
    }
}
```

### Example: Standardized Repository

```swift
actor CMBCreditCardRepository: CMBCreditCardRepositoryProtocol {
    private let encoder: JSONEncoder
    private let decoder: JSONDecoder
    
    // ✅ Dependencies injected
    init(
        encoder: JSONEncoder? = nil,
        decoder: JSONDecoder? = nil
    ) {
        // Create with standard settings if not provided
        self.encoder = encoder ?? {
            let enc = JSONEncoder()
            enc.outputFormatting = [.prettyPrinted, .sortedKeys]
            enc.dateEncodingStrategy = .iso8601
            return enc
        }()
        
        self.decoder = decoder ?? {
            let dec = JSONDecoder()
            dec.dateDecodingStrategy = .iso8601
            return dec
        }()
    }
}
```

**Better yet - use a factory:**

```swift
actor CMBCreditCardRepository: CMBCreditCardRepositoryProtocol {
    private let encoder: JSONEncoder
    private let decoder: JSONDecoder
    
    // ✅ All dependencies required
    init(
        encoder: JSONEncoder,
        decoder: JSONDecoder
    ) {
        self.encoder = encoder
        self.decoder = decoder
    }
}

// Factory for creating with standard settings
extension CMBCreditCardRepository {
    static func standard() -> CMBCreditCardRepository {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        encoder.dateEncodingStrategy = .iso8601
        
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        
        return CMBCreditCardRepository(encoder: encoder, decoder: decoder)
    }
}
```

---

## Dependency Factory Pattern (Recommended Solution)

Create a centralized factory for creating dependencies:

### Example: AppDependencies Factory

```swift
struct AppDependencies {
    // Core dependencies
    let filePersistence: FilePersistenceProtocol
    let transactionRepository: TransactionRepositoryProtocol
    let accountRepository: AccountRepositoryProtocol
    let analytics: AnalyticsProviding
    
    // Account-specific repositories
    let cmbCreditCardRepository: CMBCreditCardRepositoryProtocol
    let cmbExpressLoanRepository: CMBExpressLoanRepositoryProtocol
    let alipayRepository: Alipay3rdPartyTxRepositoryProtocol
    let weChatRepository: WeChatRepositoryProtocol
    
    // Services
    let zipFileProcessor: ZipFileProcessing
    
    // Factory methods
    static func production() -> AppDependencies {
        let filePersistence = FilePersistence()
        
        return AppDependencies(
            filePersistence: filePersistence,
            transactionRepository: FileTransactionRepository(filePersistence: filePersistence),
            accountRepository: FileAccountRepository(filePersistence: filePersistence),
            analytics: AnalyticsService(),
            cmbCreditCardRepository: CMBCreditCardRepository.standard(),
            cmbExpressLoanRepository: CMBExpressLoanRepository.standard(),
            alipayRepository: Alipay3rdPartyTxRepository.standard(),
            weChatRepository: WeChatRepository.standard(),
            zipFileProcessor: ZipFileProcessingService()
        )
    }
    
    static func test() -> AppDependencies {
        let filePersistence = MockFilePersistence()
        
        return AppDependencies(
            filePersistence: filePersistence,
            transactionRepository: InMemoryTransactionRepository(),
            accountRepository: InMemoryAccountRepository(),
            analytics: MockAnalyticsService(),
            cmbCreditCardRepository: MockCMBCreditCardRepository(),
            cmbExpressLoanRepository: MockCMBExpressLoanRepository(),
            alipayRepository: MockAlipayRepository(),
            weChatRepository: MockWeChatRepository(),
            zipFileProcessor: MockZipFileProcessor()
        )
    }
}
```

### Usage in AppState

```swift
@MainActor
final class AppState: ObservableObject {
    private let transactionRepository: TransactionRepositoryProtocol
    private let accountRepository: AccountRepositoryProtocol
    private let analytics: AnalyticsProviding
    
    // ✅ Single parameter - all dependencies bundled
    init(dependencies: AppDependencies = .production()) {
        self.transactionRepository = dependencies.transactionRepository
        self.accountRepository = dependencies.accountRepository
        self.analytics = dependencies.analytics
    }
}
```

### Usage in Import Services

```swift
struct CMBCreditCardImportService {
    private let cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol
    private let filePersistence: FilePersistenceProtocol
    private let helper: BaseImportServiceHelper
    
    // ✅ All dependencies required
    init(
        cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol,
        filePersistence: FilePersistenceProtocol
    ) {
        self.cmbCreditCardRepository = cmbCreditCardRepository
        self.filePersistence = filePersistence
        self.helper = BaseImportServiceHelper(filePersistence: filePersistence)
    }
    
    // Factory method using dependencies
    static func create(dependencies: AppDependencies) -> CMBCreditCardImportService {
        CMBCreditCardImportService(
            cmbCreditCardRepository: dependencies.cmbCreditCardRepository,
            filePersistence: dependencies.filePersistence
        )
    }
}
```

---

## Migration Strategy

### Phase 1: Add Required Parameters (Backward Compatible)

**Step 1:** Add required parameters while keeping optional ones:

```swift
// Before
init(
    cmbCreditCardRepository: (any CMBCreditCardRepositoryProtocol)? = nil,
    filePersistence: FilePersistenceProtocol? = nil
) {
    self.cmbCreditCardRepository = cmbCreditCardRepository ?? CMBCreditCardRepository()
    self.filePersistence = filePersistence ?? FilePersistence()
}

// After (Phase 1 - Backward Compatible)
init(
    cmbCreditCardRepository: (any CMBCreditCardRepositoryProtocol)? = nil,
    filePersistence: FilePersistenceProtocol? = nil
) {
    // Keep old behavior for backward compatibility
    self.cmbCreditCardRepository = cmbCreditCardRepository ?? CMBCreditCardRepository()
    self.filePersistence = filePersistence ?? FilePersistence()
    // But also inject into helper
    self.helper = BaseImportServiceHelper(filePersistence: self.filePersistence)
}
```

### Phase 2: Create Dependency Factory

**Step 2:** Create `AppDependencies` factory (as shown above)

### Phase 3: Update Call Sites

**Step 3:** Update services to use factory:

```swift
// In AccountFeatureProvider
func performImport(...) async throws -> ImportResult {
    let dependencies = AppDependencies.production()  // Or inject
    let service = CMBCreditCardImportService.create(dependencies: dependencies)
    return try await service.importCMBPDF(...)
}
```

### Phase 4: Remove Optional Parameters (Breaking Change)

**Step 4:** After all call sites updated, make parameters required:

```swift
// Final version
init(
    cmbCreditCardRepository: any CMBCreditCardRepositoryProtocol,
    filePersistence: FilePersistenceProtocol
) {
    self.cmbCreditCardRepository = cmbCreditCardRepository
    self.filePersistence = filePersistence
    self.helper = BaseImportServiceHelper(filePersistence: filePersistence)
}
```

---

## Benefits of Standardization

### 1. **Easier Testing**
```swift
// Clear, explicit dependencies
let testDeps = AppDependencies.test()
let service = CMBCreditCardImportService.create(dependencies: testDeps)
// ✅ All dependencies are mocks
```

### 2. **Consistent API**
```swift
// All services follow same pattern
let service1 = Service1.create(dependencies: deps)
let service2 = Service2.create(dependencies: deps)
let service3 = Service3.create(dependencies: deps)
```

### 3. **Visible Dependencies**
```swift
// Dependencies are explicit in initializer
init(
    repository: RepositoryProtocol,  // ✅ Clear dependency
    persistence: PersistenceProtocol  // ✅ Clear dependency
)
```

### 4. **Flexible Configuration**
```swift
// Easy to create different configurations
let prodDeps = AppDependencies.production()
let testDeps = AppDependencies.test()
let stagingDeps = AppDependencies.staging()  // Future
```

---

## Summary

### Current Problems:
1. ❌ **Three different DI patterns** across codebase
2. ❌ **Hidden dependencies** (created directly, not injected)
3. ❌ **Inconsistent API** (some optional, some required, some none)
4. ❌ **Hard to test** (can't inject all dependencies)
5. ❌ **Tight coupling** (creates concrete instances internally)

### Recommended Solution:
1. ✅ **Single pattern:** Required protocol parameters
2. ✅ **Dependency factory:** `AppDependencies` for creating dependencies
3. ✅ **Explicit dependencies:** All dependencies visible in initializer
4. ✅ **Easy testing:** Factory provides test implementations
5. ✅ **Loose coupling:** Dependencies injected, not created

### Migration:
- **Phase 1:** Add required parameters (backward compatible)
- **Phase 2:** Create dependency factory
- **Phase 3:** Update call sites
- **Phase 4:** Remove optional parameters

---

**Priority:** Medium  
**Estimated Effort:** 6-8 hours  
**Impact:** High (improves testability and maintainability)

