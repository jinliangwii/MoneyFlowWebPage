---
layout: default
title: Documentation
permalink: /docs/
---

# MoneyFlow Documentation

Welcome to the MoneyFlow documentation. This page provides an overview of all available documentation.

## Core Documentation

### Architecture & Design

- **[Architecture Overview](Docs/Architecture.md)** - Complete architecture documentation, UI design, design principles, and recommendations
- **[Data Flow](Docs/DataFlow.md)** - How data flows through the application from initialization to display
- **[SwiftUI Patterns](Docs/SwiftUIPatterns.md)** - SwiftUI patterns used throughout the app (@Published, @EnvironmentObject, alerts, etc.)
- **[Refactoring History](Docs/RefactoringHistory.md)** - History of major refactoring phases completed

### Feature Documentation

- **[CMB Import](Docs/CMBImport.md)** - China Merchants Bank transaction import feature
- **[PDF Page Number Matching](Docs/PDFPageNumberMatching.md)** - PDF page numbering implementation details
- **[Account Feature Provider Architecture](Docs/AccountFeatureProviderArchitecture.md)** - Account feature provider pattern
- **[Decoupling Plan](Docs/DecouplingPlan_CMBFromCore.md)** - Plan to decouple CMB from core layer

## New Architecture Documentation

- **[New Account Architecture - Final](Docs/NewAccountArchitecture_Final.md)** - Complete new account architecture overview
- **[New Account Architecture - Usage Examples](Docs/NewAccountArchitecture_UsageExample.md)** - Code examples and usage patterns
- **[New Account Architecture - Integration Guide](Docs/NewAccountArchitecture_IntegrationGuide.md)** - Step-by-step integration guide
- **[New Account Architecture - Complete](Docs/NewAccountArchitecture_Complete.md)** - Implementation summary and status
- **[New Account Architecture - Quick Reference](Docs/NewAccountArchitecture_QuickReference.md)** - Quick reference guide
- **[New Account Architecture - Roadmap](Docs/NewAccountArchitecture_Roadmap.md)** - Future roadmap

## Guides

- **[Testing Guide](Docs/TestingGuide.md)** - How to run tests (unit and UI tests)
- **[Debug Mode](Docs/DebugMode.md)** - Debug mode feature documentation
- **[Adding New Account Type](Docs/AddingNewAccountType.md)** - Step-by-step guide for adding new account types

## Integration & Setup

- **[Plaid Setup Guide](Docs/PlaidSetupGuide.md)** - Plaid integration setup
- **[Plaid Quick Setup](Docs/PlaidQuickSetup.md)** - Quick setup guide for Plaid
- **[SQLite Setup Guide](Docs/SQLite_Setup_Guide.md)** - SQLite database setup

## Architecture Proposals

- **[Three Layer Architecture](Docs/ThreeLayerArchitecture_RefactoringProposal.md)** - Refactoring proposal
- **[Repository Pattern](Docs/RepositoryPattern.md)** - Repository pattern documentation
- **[Dependency Injection Analysis](Docs/DependencyInjectionAnalysis.md)** - DI analysis and recommendations

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

---

[← Back to Home](/) | [View on GitHub](https://github.com/jinliangwii/MoneyFlow)
