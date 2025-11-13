---
layout: default
title: MoneyFlow
---

# MoneyFlow 💰

A modern personal finance management app built with SwiftUI for macOS.

## Overview

MoneyFlow helps you track transactions, manage accounts, and gain insights into your financial health with a clean, intuitive interface.

## Features

- 📊 **Dashboard Overview** - Visualize your financial data with interactive charts
- 🏦 **Account Management** - Track multiple bank accounts and financial institutions  
- 📈 **Transaction Tracking** - Import and categorize transactions from various sources
- 📥 **Import Support** - Import transactions via CSV, ZIP, and PDF formats
- 🎯 **Category Management** - Organize transactions with customizable categories
- 📱 **Modern UI** - Clean, card-based interface with intuitive navigation

## Quick Start

### Requirements

- macOS 13.0+ (Ventura or later)
- Xcode 15.0+
- Swift 5.9+

### Installation

```bash
git clone https://github.com/jinliangwii/MoneyFlow.git
cd MoneyFlow
open MoneyFlow.xcodeproj
```

Then build and run (⌘R) in Xcode.

## Documentation

### Core Documentation

- [Architecture Overview](Docs/Architecture.md) - Complete architecture documentation
- [Data Flow](Docs/DataFlow.md) - How data flows through the application
- [SwiftUI Patterns](Docs/SwiftUIPatterns.md) - UI patterns and best practices
- [Refactoring History](Docs/RefactoringHistory.md) - History of major refactoring phases

### Feature Documentation

- [CMB Import](Docs/CMBImport.md) - China Merchants Bank transaction import
- [Account Feature Provider Architecture](Docs/AccountFeatureProviderArchitecture.md) - Extensibility pattern
- [Adding New Account Types](Docs/AddingNewAccountType.md) - Step-by-step guide

### New Architecture

- [New Account Architecture](Docs/NewAccountArchitecture_Final.md) - Complete overview
- [Usage Examples](Docs/NewAccountArchitecture_UsageExample.md) - Code examples
- [Integration Guide](Docs/NewAccountArchitecture_IntegrationGuide.md) - Step-by-step integration

### Guides

- [Testing Guide](Docs/TestingGuide.md) - How to run tests
- [Debug Mode](Docs/DebugMode.md) - Debug mode documentation

## Project Structure

```
MoneyFlow/
├── MoneyFlow/              # Main app source code
│   ├── Core/              # Core functionality
│   ├── UI/                # User interface
│   ├── Ledgers/           # Ledger management
│   ├── DataSources/       # Data source implementations
│   └── Providers/         # Feature providers
├── Docs/                  # Documentation
├── MoneyFlowTests/        # Unit tests
└── MoneyFlowUITests/      # UI tests
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License.

## Links

- [GitHub Repository](https://github.com/jinliangwii/MoneyFlow)
- [Documentation Index](Docs/README.md)
