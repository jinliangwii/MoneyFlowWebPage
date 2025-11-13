# SwiftUI Patterns in MoneyFlow

This document explains key SwiftUI patterns used throughout the MoneyFlow app.

---

## Observable Pattern with @Published and @EnvironmentObject

### How It Works

**AppState (ObservableObject):**
```swift
@MainActor
final class AppState: ObservableObject {
    @Published private(set) var transactions: [Transaction] = []
    @Published private(set) var accounts: [Account] = []
    @Published var alertState: AlertState?
}
```

**Views Observe via @EnvironmentObject:**
```swift
struct DashboardView: View {
    @EnvironmentObject private var app: AppState
    
    var body: some View {
        Text("Transactions: \(app.transactions.count)")  // Auto-updates!
    }
}
```

**Injected at App Root:**
```swift
@main
struct MoneyFlowApp: App {
    @StateObject private var appState = AppState()
    
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(appState)  // Available to all child views
        }
    }
}
```

### Key Points

- When `@Published` properties change, all views observing via `@EnvironmentObject` automatically update
- No manual refresh needed for direct property access
- SwiftUI handles all updates reactively

---

## @ObservedObject vs @StateObject

### When to Use @StateObject

**Creating and owning the object:**
```swift
struct DashboardView: View {
    @StateObject private var viewModel = DashboardViewModel()  // ✅ Creates and owns
}
```

### When to Use @ObservedObject

**Receiving the object from a parent:**
```swift
private struct SummaryCardsRow: View {
    @ObservedObject var viewModel: DashboardViewModel  // ✅ Observes (passed from parent)
    
    var body: some View {
        MetricCard(value: viewModel.totalExpenses)  // Auto-updates when @Published properties change
    }
}
```

### Why @ObservedObject is Required

**Without @ObservedObject (broken):**
```swift
private struct SummaryCardsRow: View {
    let viewModel: DashboardViewModel  // ❌ Regular property - no observation
    
    var body: some View {
        MetricCard(value: viewModel.totalExpenses)  // ❌ Won't update!
    }
}
```

**With @ObservedObject (working):**
```swift
private struct SummaryCardsRow: View {
    @ObservedObject var viewModel: DashboardViewModel  // ✅ Observes changes
    
    var body: some View {
        MetricCard(value: viewModel.totalExpenses)  // ✅ Auto-updates!
    }
}
```

---

## Alert Handling with .alert(item:)

### Pattern

**ContentView.swift:**
```swift
.alert(item: $appState.alertState) { alert in
    Alert(
        title: Text(alert.title),
        message: Text(alert.message),
        dismissButton: .default(Text(alert.dismissButton))
    )
}
```

### AlertState Structure

**Location:** `Core/Domain/Errors.swift`

```swift
struct AlertState: Identifiable {
    let id = UUID()
    let title: String
    let message: String
    let dismissButton: String = "确定"
    
    static func from(_ error: AppError) -> AlertState {
        AlertState(
            title: "错误",
            message: error.errorDescription ?? "未知错误"
        )
    }
}
```

### Usage

**Triggering an Alert:**
```swift
// In AppState
catch {
    alertState = AlertState.from(AppError.dataStoreError(error.localizedDescription))
}
```

**How It Works:**
1. When `alertState` is set to non-`nil`, SwiftUI automatically displays the alert
2. The closure receives the `AlertState` and constructs the SwiftUI `Alert`
3. When dismissed, SwiftUI sets `alertState` back to `nil`

---

## ViewModel Pattern

### Current Pattern (Manual Updates)

**DashboardView:**
```swift
struct DashboardView: View {
    @EnvironmentObject private var app: AppState
    @StateObject private var viewModel = DashboardViewModel()
    
    var body: some View {
        // View code
    }
    .onChange(of: app.transactions) { _, newTransactions in
        Task { @MainActor in
            viewModel.update(transactions: newTransactions, accounts: app.accounts)
        }
    }
    .onChange(of: app.accounts) { _, newAccounts in
        Task { @MainActor in
            viewModel.update(transactions: app.transactions, accounts: newAccounts)
        }
    }
}
```

### Recommended Pattern (Reactive)

**DashboardViewModel should observe AppState:**
```swift
@MainActor
class DashboardViewModel: ObservableObject {
    @Published var totalExpenses: Decimal = 0
    @Published var monthlySummaries: [MonthlySummary] = []
    
    private var cancellables = Set<AnyCancellable>()
    
    init(appState: AppState) {
        appState.$transactions
            .combineLatest(appState.$accounts)
            .debounce(for: .milliseconds(100), scheduler: DispatchQueue.main)
            .sink { [weak self] transactions, accounts in
                self?.update(transactions: transactions, accounts: accounts)
            }
            .store(in: &cancellables)
    }
    
    private func update(transactions: [Transaction], accounts: [Account]) {
        totalExpenses = transactions.filter { $0.amount < 0 }
            .reduce(Decimal(0)) { $0 + $1.amount }
        // ... more updates
    }
}
```

**Alternative: Using onReceive in View:**
```swift
.onReceive(app.$transactions.combineLatest(app.$accounts)) { transactions, accounts in
    viewModel.update(transactions: transactions, accounts: accounts)
}
```

---

## Data Flow Pattern

### Complete Flow Example

**When AppState Updates:**
```swift
// 1. User action triggers update
func add(_ items: [Transaction]) {
    transactions.append(contentsOf: items)  // @Published property changes
    Task {
        await save()
    }
}
```

**What Happens Automatically:**
```
1. transactions.append() executes
   ↓
2. @Published wrapper detects change
   ↓
3. Combine publishes change notification
   ↓
4. SwiftUI detects change (via @EnvironmentObject subscription)
   ↓
5. All views observing AppState re-render:
   - DashboardView (if visible)
   - PropertiesView (if visible)
   - SummaryView (if visible)
   ↓
6. Views re-compute body
   ↓
7. Only changed parts of UI update (SwiftUI diffing)
```

---

## Key Takeaways

1. **Always use `@ObservedObject`** (not `let`) when you want a view to react to changes
2. **Only `@Published` properties** trigger view updates
3. **`@ObservedObject` doesn't own** the object - it just observes it
4. **Multiple views can observe** the same object - all will update when `@Published` properties change
5. **ViewModels should observe AppState** via Combine instead of manual `onChange` handlers

---

## Common Pitfalls

### 1. Using `let` instead of `@ObservedObject`
```swift
// ❌ Wrong - won't update
let viewModel: DashboardViewModel

// ✅ Correct - will update
@ObservedObject var viewModel: DashboardViewModel
```

### 2. Forgetting to mark properties as `@Published`
```swift
// ❌ Won't trigger view updates
var totalExpenses: Decimal = 0

// ✅ Will trigger view updates
@Published var totalExpenses: Decimal = 0
```

### 3. Using `@StateObject` for objects created elsewhere
```swift
// ❌ Wrong - object already created by parent
@StateObject var viewModel: DashboardViewModel

// ✅ Correct - observing object passed from parent
@ObservedObject var viewModel: DashboardViewModel
```


