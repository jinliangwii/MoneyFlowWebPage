# MoneyFlow Architecture Documentation

## Overview

MoneyFlow is a personal finance management app built with SwiftUI for managing transactions from bank accounts and summarizing cash usage. The app provides dashboards for overview, account management, transaction lists, and CSV import.

### Scope and Goals
- Build a personal finance app that imports transactions from bank accounts and summarizes cash usage
- Provide dashboards for overview/summary, properties (accounts), transactions list, and CSV import
- Match a clean card-based UI style with charts (bar, donut, line) and a left sidebar for navigation

### Major Screens
- **Sidebar** (NavigationSplitView): 概览 (Dashboard), 资产 (Properties), 统计 (Summary), 设置 (Settings)
- **Dashboard**: Summary cards, income/expense bar chart, category donut chart, cumulative net line chart
- **Properties**: Two-column layout with account list and account detail view
- **Summary**: Metrics grid, weekly breakdowns, net trend analysis
- **Import**: CSV import interface and CMB (China Merchants Bank) ZIP/PDF import

---

## Current Architecture

### Folder Structure

```
MoneyFlow/
├── App/                      # App entry point
│   ├── MoneyFlowApp.swift
│   └── ContentView.swift
│
├── Core/                     # Core app functionality
│   ├── Domain/               # Domain models
│   │   ├── Models.swift
│   │   ├── Errors.swift
│   │   └── Money.swift
│   ├── State/                 # App state
│   │   └── AppState.swift
│   ├── Data/                  # Data access
│   │   ├── DataStore.swift
│   │   └── Protocols.swift
│   ├── DesignSystem/         # UI components
│   │   ├── Card.swift
│   │   ├── Colors.swift
│   │   ├── Formatters.swift
│   │   └── InstitutionIcons.swift
│   └── Services/              # Shared services
│       ├── Analytics/
│       │   └── AnalyticsService.swift
│       ├── Import/
│       │   └── ImportService.swift
│       └── ZipFileProcessingService.swift
│
└── Features/                  # Feature modules
    ├── Dashboard/
    ├── Properties/
    ├── Summary/
    ├── CMB/                   # China Merchants Bank feature
    └── AccountProviders/      # Account provider infrastructure
```

### Key Design Principles

1. **Feature-Based Organization**: Features are isolated in their own folders
2. **Protocol-Based Design**: Dependency inversion via protocols (`DataStoreProtocol`, `AnalyticsProviding`)
3. **Account Provider Pattern**: Extensible pattern for account-specific features (CMB)
4. **Single Source of Truth**: `AppState` centralizes all app-wide state
5. **Reactive Updates**: SwiftUI's `@Published` and `@EnvironmentObject` for automatic UI updates

---

## Architecture Strengths ✅

### What's Working Well

1. **Feature-Based Organization**: Clear separation with `Features/` and `Core/` folders
2. **Account Provider Pattern**: Excellent extensibility for account-specific features
3. **Protocol-Based Design**: Dependency inversion via protocols
4. **Async Data Access**: `DataStore` as an actor with async operations
5. **ViewModels**: Dashboard and Summary have dedicated ViewModels
6. **Component Extraction**: Well-extracted components in Properties feature

---

## Critical Issues 🔴

### Issue 1: CMB Coupling in Core Layer

**Problem:** `DataStoreProtocol` and `DataStore` contain CMB-specific methods:
- `loadRawCMBTransactions()`
- `saveRawCMBTransactions(_:)`
- `loadImportBatches()`
- `saveImportBatches(_:)`

**Impact:**
- Core layer depends on feature-specific types
- Adding new account types requires modifying `DataStoreProtocol`
- Violates Open/Closed Principle

**Solution:** Create `CMBRepository` pattern (see `DecouplingPlan_CMBFromCore.md`)

### Issue 2: ViewModel State Management

**Problem:** ViewModels are manually updated via `update()` calls in views with multiple `onChange` handlers.

**Recommendation:** ViewModels should observe `AppState` directly using Combine:

```swift
appState.$transactions
    .combineLatest(appState.$accounts)
    .debounce(for: .milliseconds(100), scheduler: DispatchQueue.main)
    .sink { [weak self] transactions, accounts in
        self?.update(transactions: transactions, accounts: accounts)
    }
```

### Issue 3: Analytics Cache Management

**Problem:** Cache keys can collide, no invalidation, unbounded growth.

**Recommendation:**
- Use all transaction IDs for cache keys
- Add cache invalidation
- Add size limits
- Track cache entries with transaction IDs

---

## Architecture Patterns

### Account Feature Provider Pattern

Allows account-specific functionality (like CMB import) to be decoupled from general app logic.

**Key Components:**
- `AccountFeatureProvider` protocol - Base protocol for account features
- `AccountImportProvider` protocol - Extends provider for import functionality
- `AccountFeatureRegistry` - Central registry for finding providers

**Usage:**
```swift
if let featureProvider = AccountFeatureRegistry.shared.featureProvider(for: account),
   let actionButtons = featureProvider.makeActionButtons(for: account) {
    actionButtons
}
```

See `AccountFeatureProviderArchitecture.md` for details.

---

## Recommended Improvements

### High Priority

1. **Remove CMB coupling from Core** - Extract to `CMBRepository`
2. **Fix ViewModel observation** - Use Combine instead of manual updates
3. **Improve Analytics cache** - Better keys, invalidation, size limits

### Medium Priority

1. **Repository Pattern** - Introduce repositories for data access abstraction
2. **Use Case Pattern** - Encapsulate business logic in use cases
3. **Unified Error Handling** - Create `ErrorHandler` protocol

### Low Priority

1. **DI Container** - Nice to have, not critical
2. **State Splitting** - Only if AppState grows too large
3. **Navigation Coordinator** - Only if navigation becomes complex

---

## Future Enhancements

### For Adding New Account Types

Follow the CMB pattern:
1. Create feature folder: `Features/ICBC/`
2. Implement `AccountFeatureProvider`
3. Register in `AccountFeatureRegistry`

### For Multi-Platform

Views are platform-agnostic. For platform-specific adapters:
```swift
#if os(iOS)
typealias PlatformNavigation = UINavigationController
#elseif os(macOS)
typealias PlatformNavigation = NSWindow
#endif
```

---

## Testing Strategy

### Current State
- Some test files exist (`CMBPDFParserTests`, etc.)
- Protocols enable mocking

### Recommendations
1. Unit test ViewModels with mock AppState
2. Test Account Providers independently
3. Integration tests for DataStore persistence

---

## Related Documents

- `AccountFeatureProviderArchitecture.md` - Provider pattern details
- `DecouplingPlan_CMBFromCore.md` - Plan to remove CMB coupling
- `DataFlow.md` - Complete data flow documentation
- `SwiftUIPatterns.md` - SwiftUI implementation patterns

---

## Conclusion

The architecture is solid with good patterns in place. Main improvements focus on:
1. **Decoupling**: Remove feature-specific code from Core layer
2. **Observability**: Better reactive patterns for ViewModels
3. **Reliability**: Fix caching and error handling
4. **Testability**: Repository and Use Case patterns

These changes will make the codebase more maintainable and scalable as features and account types are added.

