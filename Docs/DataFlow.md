# MoneyFlow Data Flow Documentation

## Overview

This document explains how data flows through the MoneyFlow application from various entry points (user actions, imports, initialization) through storage, state management, processing, and finally display.

---

## Key Principles

1. **Single Source of Truth**: `AppState` holds all app-wide state (`accounts`, `transactions`)
2. **Reactive Updates**: `@Published` properties trigger SwiftUI updates automatically
3. **Async Data Access**: `DataStore` is an actor (thread-safe), all disk I/O happens off main thread
4. **Layered Architecture**: Clear boundaries between views, view models, state, services, and data
5. **Feature Provider Pattern**: Account-specific features (like CMB import) are isolated

---

## App Initialization Flow

```
MoneyFlowApp (App Entry)
    ↓
    Creates AppState
    ↓
    AppState.init()
        ↓
        Creates DataStore (actor)
        Creates AnalyticsService
        ↓
        Task { await load() }
            ↓
            DataStore.loadAccounts()
            DataStore.loadTransactions()
                ↓
                FileManager reads JSON files
                ↓
                Decode JSON → [Account], [Transaction]
                ↓
                AppState.accounts = [...]
                AppState.transactions = [...]
                ↓
        @Published properties trigger SwiftUI updates
```

**Code Flow:**

1. **App Launch** (`MoneyFlowApp.swift`):
```swift
@main
struct MoneyFlowApp: App {
    @StateObject private var appState = AppState()  // Creates AppState
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appState)  // Injects AppState into environment
        }
    }
}
```

2. **AppState Initialization** (`AppState.swift`):
```swift
@MainActor
final class AppState: ObservableObject {
    @Published private(set) var transactions: [Transaction] = []
    @Published private(set) var accounts: [Account] = []
    
    private let store: DataStoreProtocol
    private let analytics: AnalyticsProviding
    
    init(store: DataStoreProtocol? = nil, analytics: AnalyticsProviding? = nil) {
        self.store = store ?? DataStore()
        self.analytics = analytics ?? AnalyticsService()
        
        // Load data asynchronously
        Task {
            await load()
        }
    }
    
    func load() async {
        accounts = try await store.loadAccounts()
        transactions = try await store.loadTransactions()
    }
}
```

3. **DataStore Loads from Disk** (`DataStore.swift`):
```swift
actor DataStore: DataStoreProtocol {
    func loadTransactions() async throws -> [Transaction] {
        let url = try fileURL("transactions.json")
        guard FileManager.default.fileExists(atPath: url.path) else { return [] }
        
        let data = try await Task.detached(priority: .utility) {
            try Data(contentsOf: url)  // Read from disk (off main thread)
        }.value
        
        return try decoder.decode([Transaction].self, from: data)  // Decode JSON
    }
}
```

---

## Adding an Account Flow

```
User taps "新增" button
    ↓
AddAccountView appears
    ↓
User selects AccountType → AccountProvider
    ↓
User fills configuration form
    ↓
User taps "添加" → save()
    ↓
AddAccountView validates
    ↓
AccountProvider.createAccount(draft) → Account
    ↓
AppState.addAccount(account)
    ↓
AppState.accounts.append(account)
    ↓
Task { await save() }
    ↓
DataStore.saveAccounts(accounts)
    ↓
Encode [Account] → JSON
    ↓
Write to disk (accounts.json)
    ↓
@Published accounts triggers SwiftUI update
    ↓
All views observing AppState automatically refresh
```

---

## CMB Transaction Import Flow

```
User opens CMB import view
    ↓
User selects ZIP file with password
    ↓
CMBImportView calls AppState.importAccountTransactions()
    ↓
AppState finds CMBAccountFeatureProvider via registry
    ↓
CMBAccountFeatureProvider.performImport()
    ↓
CMBImportService.importCMBZipPDF()
    ↓
Step 1: ZipFileProcessingService.extractZipFile()
    ↓ Extract ZIP → PDF file
    ↓
Step 2: CMBPDFParser.getInfo()
    ↓ Parse PDF metadata
    ↓
Step 3: CMBPDFParser.parseTransactions()
    ↓ Extract text from each page
    ↓ Parse → [RawCMBTransaction]
    ↓
Step 4: Balance verification
    ↓
Step 5: Load existing data
    ↓ DataStore.loadRawCMBTransactions()
    ↓
Step 6: Duplicate detection
    ↓ Generate fingerprint for each raw transaction
    ↓ Compare with existing
    ↓ Filter out duplicates
    ↓
Step 7: Convert RawCMBTransaction → Transaction
    ↓
Step 8: Save to DataStore
    ↓ DataStore.saveRawCMBTransactions(allRawTransactions)
    ↓
Step 9: Update AppState
    ↓ AppState.transactions.append(contentsOf: newTransactions)
    ↓ AppState.save()
    ↓
@Published triggers SwiftUI updates
    ↓
Views automatically refresh
```

---

## Data Display Flow

```
User navigates to Dashboard
    ↓
DashboardView appears
    ↓
DashboardView observes AppState via @EnvironmentObject
    ↓
DashboardViewModel created
    ↓
View calls viewModel.update(transactions, accounts)
    ↓
DashboardViewModel computes metrics:
    - totalExpenses
    - totalNet
    - monthlySummaries (via AnalyticsService)
    - categoryTotals (via AnalyticsService)
    - cumulativeNetSeries (via AnalyticsService)
    ↓
@Published properties in ViewModel trigger UI updates
    ↓
Views render:
    - SummaryCardsRow (displays totals)
    - IncomeExpenseBarChart (monthly summaries)
    - CategoryDonutAndList (category totals)
    - NetTrendLineChart (cumulative series)
    ↓
When AppState changes:
    ↓
onChange(of: app.transactions) triggers
    ↓
viewModel.update() called again
    ↓
UI updates automatically
```

---

## Data Persistence

### File Structure
```
Documents/
├── accounts.json          # [Account] array (JSON)
├── transactions.json      # [Transaction] array (JSON)
├── rawCMBTransactions.json  # [RawCMBTransaction] array (CMB-specific)
└── importBatches.json     # [ImportCMBTransactionBatch] array (CMB-specific)
```

### JSON Format
- All data encoded as JSON
- ISO8601 date encoding
- Pretty printed for readability
- Atomic writes (prevents corruption)

### Save Flow
```
AppState method called (e.g., addAccount)
    ↓
@Published property updated
    ↓
SwiftUI updates views immediately
    ↓
Task { await save() }
    ↓
DataStore.save()
    ↓
Encode to JSON
    ↓
Atomic write to disk (off main thread)
```

---

## Error Handling Flow

```
Error occurs (e.g., in DataStore.save())
    ↓
catch block captures error
    ↓
AppState.alertState = AlertState.from(error)
    ↓
@Published alertState triggers SwiftUI update
    ↓
View shows alert via .alert(item: $app.alertState)
```

**Code Example:**
```swift
// In AppState
func save() async {
    do {
        try await store.saveTransactions(transactions)
    } catch {
        alertState = AlertState.from(AppError.dataStoreError(error.localizedDescription))
    }
}

// In ContentView
.alert(item: $appState.alertState) { alert in
    Alert(title: Text(alert.title), message: Text(alert.message))
}
```

---

## Performance Optimizations

### Caching
- `AnalyticsService` caches computation results
- Cache key based on transaction set
- Avoids recomputing for same data

### Async Operations
- All disk I/O happens off main thread
- `DataStore` is an actor (ensures thread-safety)
- UI updates happen on main thread via `@MainActor`

### Debouncing (Future Improvement)
- Currently, every change triggers save
- Could debounce saves to reduce disk writes
- Example: Save after 500ms of inactivity

---

## Data Flow Patterns

### Pattern 1: User Action → State Update → Persistence
```
User Action
    ↓
AppState method (e.g., addAccount)
    ↓
Update @Published property
    ↓
SwiftUI updates views
    ↓
Async save to DataStore
    ↓
Write to disk
```

### Pattern 2: Import → Processing → State Update → Persistence
```
Import Trigger
    ↓
Feature Provider (e.g., CMBAccountFeatureProvider)
    ↓
Import Service (e.g., CMBImportService)
    ↓
Process data (parse, validate, deduplicate)
    ↓
Return ImportResult
    ↓
AppState updates transactions/accounts
    ↓
SwiftUI updates views
    ↓
Async save to DataStore
```

### Pattern 3: Display → Compute → Cache → Render
```
View appears
    ↓
ViewModel.update() called
    ↓
AnalyticsService.compute()
    ↓
Check cache
    ↓
If cached: return cached result
If not: compute, cache, return
    ↓
ViewModel @Published properties update
    ↓
Views re-render automatically
```

---

## State Synchronization

### How State Stays Consistent

1. **Immediate State Updates**: Mutations update `@Published` properties immediately, SwiftUI reacts synchronously
2. **Async Persistence**: Persistence happens in background via `Task`, doesn't block UI updates
3. **Load on App Start**: `AppState.init()` automatically loads persisted data
4. **No Manual Sync Needed**: SwiftUI's reactive system handles all updates, `@Published` ensures consistency

---

## Summary

The data flow in MoneyFlow follows a clean, reactive architecture:

1. **Single Source of Truth**: `AppState` centralizes state
2. **Reactive Updates**: SwiftUI automatically updates when state changes
3. **Layered Separation**: Clear boundaries between views, view models, state, services, and data
4. **Async Operations**: Non-blocking I/O operations
5. **Feature Isolation**: Account-specific features don't pollute core layer

This architecture ensures:
- **Testability**: Each layer can be tested independently
- **Maintainability**: Clear responsibilities for each component
- **Scalability**: Easy to add new features (follow existing patterns)
- **Performance**: Caching and async operations optimize performance
