# MoneyFlow Documentation

This folder contains documentation for the MoneyFlow personal finance management app.

## Core Documentation

- **[Architecture.md](Architecture.md)** - Complete architecture overview, UI design, design principles, and recommendations
- **[DataFlow.md](DataFlow.md)** - How data flows through the application from initialization to display
- **[SwiftUIPatterns.md](SwiftUIPatterns.md)** - SwiftUI patterns used throughout the app (@Published, @EnvironmentObject, alerts, etc.)
- **[RefactoringHistory.md](RefactoringHistory.md)** - History of major refactoring phases completed

## Feature Documentation

- **[CMBImport.md](CMBImport.md)** - China Merchants Bank transaction import feature
- **[PDFPageNumberMatching.md](PDFPageNumberMatching.md)** - PDF page numbering implementation details
- **[AccountFeatureProviderArchitecture.md](AccountFeatureProviderArchitecture.md)** - Account feature provider pattern
- **[DecouplingPlan_CMBFromCore.md](DecouplingPlan_CMBFromCore.md)** - Plan to decouple CMB from core layer

## New Architecture Documentation

- **[NewAccountArchitecture_Final.md](NewAccountArchitecture_Final.md)** - Complete new account architecture overview
- **[NewAccountArchitecture_UsageExample.md](NewAccountArchitecture_UsageExample.md)** - Code examples and usage patterns
- **[NewAccountArchitecture_IntegrationGuide.md](NewAccountArchitecture_IntegrationGuide.md)** - Step-by-step integration guide
- **[NewAccountArchitecture_Complete.md](NewAccountArchitecture_Complete.md)** - Implementation summary and status

## Guides

- **[TestingGuide.md](TestingGuide.md)** - How to run tests (unit and UI tests)
- **[DebugMode.md](DebugMode.md)** - Debug mode feature documentation
- **[AddingNewAccountType.md](AddingNewAccountType.md)** - Step-by-step guide for adding new account types

## Quick Reference

### Architecture Overview
- Feature-based organization (Core/ + Features/)
- Protocol-based dependency inversion
- Account Provider Pattern for extensibility
- Single source of truth (AppState)

### Key Concepts
- **@Published** + **@EnvironmentObject** for reactive updates
- **@StateObject** vs **@ObservedObject** for view models
- **AccountFeatureProvider** pattern for account-specific features
- Async data access via actor-based DataStore

### Common Tasks
- Adding a new account type → See `AddingNewAccountType.md`
- Understanding account provider pattern → See `AccountFeatureProviderArchitecture.md`
- Understanding data flow → See `DataFlow.md`
- Debugging UI updates → See `SwiftUIPatterns.md`
- Running tests → See `TestingGuide.md`
- Using new data source architecture → See `NewAccountArchitecture_UsageExample.md`
- Integrating new architecture → See `NewAccountArchitecture_IntegrationGuide.md`

## Document Relationships

```
Architecture.md (overview)
    ├── DataFlow.md (how data moves)
    ├── SwiftUIPatterns.md (UI patterns)
    ├── AccountFeatureProviderArchitecture.md (extensibility pattern)
    └── DecouplingPlan_CMBFromCore.md (planned improvements)

CMBImport.md (feature)
    ├── PDFPageNumberMatching.md (implementation detail)
    └── AccountFeatureProviderArchitecture.md (uses provider pattern)

AddingNewAccountType.md (guide)
    └── AccountFeatureProviderArchitecture.md (references provider pattern)
```

