# Repository Pattern Explained

## Overview

The **Repository Pattern** is a design pattern that abstracts data access logic behind an interface. It provides a clean separation between business logic and data persistence, making the codebase more maintainable and testable.

---

## Current Architecture

### Current Flow
```
AppState (Business Logic + State Management)
    ↓
DataStoreProtocol (Low-level persistence)
    ↓
DataStore (File I/O, JSON encoding/decoding)
```

### Current Issues

1. **AppState is doing too much:**
   - State management (`@Published` properties)
   - Business logic (adding/removing transactions)
   - Direct data access coordination (`store.loadTransactions()`)
   - Cache invalidation
   - Error handling

2. **DataStore is too low-level:**
   - Raw file operations
   - JSON encoding/decoding
   - File path management
   - No query methods (e.g., "get transactions by account")

3. **Hard to test:**
   - AppState tightly coupled to DataStore
   - Cannot easily mock data access for testing

---

## Repository Pattern Architecture

### Proposed Flow
```
AppState (State Management only)
    ↓
TransactionRepository (Domain-focused data access)
AccountRepository (Domain-focused data access)
    ↓
RepositoryProtocol (Abstract interface)
    ↓
FileRepository (Implementation - JSON files)
MemoryRepository (Implementation - in-memory for tests)
DatabaseRepository (Future - CoreData/SQLite)
```

### Benefits

1. **Separation of Concerns:**
   - `AppState`: Only manages state
   - `Repository`: Handles all data access logic
   - `DataStore`: Only handles low-level file I/O

2. **Better Query Abstraction:**
   ```swift
   // Instead of:
   let accountTransactions = transactions.filter { $0.accountId == id }
   
   // You can have:
   let accountTransactions = await transactionRepository.findByAccount(id)
   ```

3. **Easier Testing:**
   - Mock repositories for unit tests
   - In-memory repositories for fast tests
   - No file I/O in tests

4. **Flexibility:**
   - Swap implementations (file → database) without changing business logic
   - Multiple storage backends

---

## Example Implementation

### 1. Repository Protocols

```swift
// TransactionRepositoryProtocol.swift
protocol TransactionRepositoryProtocol {
    // Basic CRUD
    func findAll() async throws -> [Transaction]
    func save(_ transactions: [Transaction]) async throws
    
    // Query methods
    func findByAccount(_ accountId: UUID) async throws -> [Transaction]
    func findByDateRange(_ start: Date, _ end: Date) async throws -> [Transaction]
    func findByCategory(_ category: String) async throws -> [Transaction]
    
    // Single entity operations
    func findById(_ id: UUID) async throws -> Transaction?
    func delete(_ id: UUID) async throws
    
    // Bulk operations
    func deleteByAccount(_ accountId: UUID) async throws
}
```

```swift
// AccountRepositoryProtocol.swift
protocol AccountRepositoryProtocol {
    func findAll() async throws -> [Account]
    func save(_ accounts: [Account]) async throws
    func findById(_ id: UUID) async throws -> Account?
    func save(_ account: Account) async throws
    func delete(_ id: UUID) async throws
}
```

### 2. Concrete Repository Implementation

```swift
// FileTransactionRepository.swift
actor FileTransactionRepository: TransactionRepositoryProtocol {
    private let dataStore: DataStoreProtocol
    
    init(dataStore: DataStoreProtocol = DataStore()) {
        self.dataStore = dataStore
    }
    
    func findAll() async throws -> [Transaction] {
        return try await dataStore.loadTransactions()
    }
    
    func save(_ transactions: [Transaction]) async throws {
        try await dataStore.saveTransactions(transactions)
    }
    
    func findByAccount(_ accountId: UUID) async throws -> [Transaction] {
        let all = try await findAll()
        return all.filter { $0.accountId == accountId }
    }
    
    func findByDateRange(_ start: Date, _ end: Date) async throws -> [Transaction] {
        let all = try await findAll()
        return all.filter { $0.date >= start && $0.date <= end }
    }
    
    func findByCategory(_ category: String) async throws -> [Transaction] {
        let all = try await findAll()
        return all.filter { $0.category == category }
    }
    
    func findById(_ id: UUID) async throws -> Transaction? {
        let all = try await findAll()
        return all.first { $0.id == id }
    }
    
    func delete(_ id: UUID) async throws {
        var all = try await findAll()
        all.removeAll { $0.id == id }
        try await save(all)
    }
    
    func deleteByAccount(_ accountId: UUID) async throws {
        var all = try await findAll()
        all.removeAll { $0.accountId == accountId }
        try await save(all)
    }
}
```

### 3. In-Memory Repository (for Testing)

```swift
// InMemoryTransactionRepository.swift
actor InMemoryTransactionRepository: TransactionRepositoryProtocol {
    private var transactions: [Transaction] = []
    
    func findAll() async throws -> [Transaction] {
        return transactions
    }
    
    func save(_ transactions: [Transaction]) async throws {
        self.transactions = transactions
    }
    
    func findByAccount(_ accountId: UUID) async throws -> [Transaction] {
        return transactions.filter { $0.accountId == accountId }
    }
    
    // ... implement other methods similarly
}
```

### 4. Updated AppState

```swift
@MainActor
final class AppState: ObservableObject {
    @Published private(set) var transactions: [Transaction] = []
    @Published private(set) var accounts: [Account] = []
    @Published var alertState: AlertState?
    
    private let transactionRepository: TransactionRepositoryProtocol
    private let accountRepository: AccountRepositoryProtocol
    private let analytics: AnalyticsProviding
    
    init(
        transactionRepository: TransactionRepositoryProtocol? = nil,
        accountRepository: AccountRepositoryProtocol? = nil,
        analytics: AnalyticsProviding? = nil
    ) {
        self.transactionRepository = transactionRepository ?? FileTransactionRepository()
        self.accountRepository = accountRepository ?? FileAccountRepository()
        self.analytics = analytics ?? AnalyticsService()
        
        Task {
            await load()
        }
    }
    
    func load() async {
        do {
            accounts = try await accountRepository.findAll()
            transactions = try await transactionRepository.findAll()
            
            let currentTransactionIds = Set(transactions.map { $0.id })
            analytics.invalidateStaleEntries(currentTransactionIds: currentTransactionIds)
        } catch {
            alertState = AlertState.from(AppError.dataStoreError(error.localizedDescription))
        }
    }
    
    func add(_ items: [Transaction]) {
        let newTransactionIds = Set(items.map { $0.id })
        transactions.append(contentsOf: items)
        analytics.invalidateCache(for: newTransactionIds)
        
        Task {
            do {
                try await transactionRepository.save(transactions)
            } catch {
                alertState = AlertState.from(AppError.dataStoreError(error.localizedDescription))
            }
        }
    }
    
    func transactions(for accountId: UUID) -> [Transaction] {
        // Could use repository, but for reactive updates, keep in memory
        return transactions.filter { $0.accountId == accountId }
    }
    
    // Future: Could add methods like:
    // func transactionsByDateRange(_ start: Date, _ end: Date) async throws -> [Transaction] {
    //     return try await transactionRepository.findByDateRange(start, end)
    // }
}
```

---

## Comparison: Current vs Repository Pattern

### Current Approach

```swift
// AppState needs to know about file storage details
func load() async {
    accounts = try await store.loadAccounts()
    transactions = try await store.loadTransactions()
}

// Filtering logic scattered in AppState
func transactions(for accountId: UUID, year: Int, month: Int) -> [Transaction] {
    let calendar = Calendar.current
    return transactions.filter { tx in
        tx.accountId == accountId && 
        calendar.component(.year, from: tx.date) == year && 
        calendar.component(.month, from: tx.date) == month
    }.sorted { $0.date > $1.date }
}
```

### With Repository Pattern

```swift
// AppState delegates to repository
func load() async {
    accounts = try await accountRepository.findAll()
    transactions = try await transactionRepository.findAll()
}

// Query logic encapsulated in repository
func transactions(for accountId: UUID, year: Int, month: Int) -> [Transaction] {
    // Could be:
    // return try await transactionRepository.findByAccountAndMonth(accountId, year, month)
    // Or keep in-memory for performance (already loaded)
    return transactions.filter { ... }
}
```

---

## Key Advantages

### 1. **Better Abstraction**
- Business logic doesn't know about files, databases, or network calls
- Easy to swap implementations

### 2. **Improved Testability**
```swift
// In tests:
let mockRepository = InMemoryTransactionRepository()
let appState = AppState(transactionRepository: mockRepository)
// Test without touching filesystem
```

### 3. **Query Encapsulation**
- Complex queries live in repositories, not scattered in AppState
- Reusable across features

### 4. **Feature-Specific Repositories**
- You already have `CMBRepository` for CMB-specific data
- Pattern extends naturally to other features

### 5. **Future-Proof**
- Easy to add caching layer
- Easy to switch to CoreData, SQLite, or cloud storage
- Can add optimistic updates, conflict resolution, etc.

---

## Implementation Strategy

### Phase 1: Create Repository Protocols
1. Define `TransactionRepositoryProtocol`
2. Define `AccountRepositoryProtocol`

### Phase 2: Implement File Repositories
1. Implement `FileTransactionRepository` using existing `DataStore`
2. Implement `FileAccountRepository` using existing `DataStore`
3. Keep `DataStore` for low-level file operations

### Phase 3: Refactor AppState
1. Replace `DataStoreProtocol` with repositories
2. Move query logic from `AppState` to repositories
3. Keep `AppState` focused on state management

### Phase 4: Add In-Memory Repositories
1. For testing
2. For debugging
3. For preview data

---

## Example: You Already Have This Pattern!

Look at `CMBRepository` - it's already using the Repository Pattern! 

```swift
protocol CMBRepositoryProtocol {
    func loadRawTransactions() async throws -> [RawCMBTransaction]
    func saveRawTransactions(_ data: [RawCMBTransaction]) async throws
    // ...
}

actor CMBRepository: CMBRepositoryProtocol {
    // Implementation details hidden
}
```

The suggestion is to apply this same pattern to `Transaction` and `Account` data as well.

---

## When to Use Repositories

✅ **Use repositories when:**
- You have complex query requirements
- You want to swap storage backends
- You need better testability
- You have multiple features accessing the same data differently

❌ **Don't over-engineer:**
- Simple CRUD operations might not need repositories
- If you only ever use file storage, repositories add abstraction you may not need
- Start simple, add repositories when complexity grows

---

## Conclusion

The Repository Pattern provides a clean abstraction layer between your business logic (`AppState`) and data persistence (`DataStore`). While your current architecture works, repositories would:

1. Make the code more testable
2. Separate concerns better
3. Enable easier refactoring
4. Match the pattern you're already using for CMB data

The pattern is marked as "Medium Priority" because your current architecture works fine, but repositories would improve maintainability as the codebase grows.

