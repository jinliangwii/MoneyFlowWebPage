---
layout: default
title: MoneyFlow
---

<div class="logo-container">
  <img src="{{ '/assets/images/logo.png' | relative_url }}" alt="MoneyFlow Logo">
</div>

<div class="hero-section">
  <h1>MoneyFlow 💰</h1>
  <p style="font-size: 1.2rem; margin-bottom: 0;">A modern personal finance management app built with SwiftUI for macOS</p>
</div>

## Overview

MoneyFlow helps you track transactions, manage accounts, and gain insights into your financial health with a clean, intuitive interface. Take control of your finances with powerful tools designed for the modern user.

## Current Features

<div class="feature-grid">
  <div class="feature-card">
    <h3>📊 Dashboard Overview</h3>
    <p>Visualize your financial data with interactive charts including income/expense bars, category donuts, and cumulative net trends.</p>
  </div>
  
  <div class="feature-card">
    <h3>🏦 Account Management</h3>
    <p>Track multiple bank accounts and financial institutions with a clean two-column layout for easy navigation.</p>
  </div>
  
  <div class="feature-card">
    <h3>📈 Transaction Tracking</h3>
    <p>Import and categorize transactions from various sources with support for CSV, ZIP, and PDF formats.</p>
  </div>
  
  <div class="feature-card">
    <h3>📥 Import Support</h3>
    <p>Seamlessly import transactions via CSV files, ZIP archives, and PDF documents from supported banks.</p>
  </div>
  
  <div class="feature-card">
    <h3>🎯 Category Management</h3>
    <p>Organize transactions with customizable categories and rules for better financial organization.</p>
  </div>
  
  <div class="feature-card">
    <h3>📱 Modern UI</h3>
    <p>Enjoy a clean, card-based interface with intuitive navigation built with SwiftUI for macOS.</p>
  </div>
</div>

## Future Plans 🚀

<div class="future-plans">
  <h2>Coming Soon</h2>
  <p style="text-align: center; margin-bottom: 2rem;">We're constantly working to make MoneyFlow even better. Here's what's on the horizon:</p>
  
  <div class="plan-grid">
    <div class="plan-card">
      <h3>🔗 Plaid Integration <span class="badge">In Development</span></h3>
      <p>Connect your bank accounts securely with Plaid integration for automatic transaction syncing.</p>
      <ul>
        <li>Secure bank account linking</li>
        <li>Automatic transaction import</li>
        <li>Real-time balance updates</li>
        <li>Support for major US banks</li>
      </ul>
    </div>
    
    <div class="plan-card">
      <h3>🤖 AI-Powered Categorization <span class="badge">Planned</span></h3>
      <p>Let artificial intelligence automatically categorize your transactions with high accuracy.</p>
      <ul>
        <li>Smart transaction categorization</li>
        <li>Learning from your preferences</li>
        <li>Merchant recognition</li>
        <li>Custom category suggestions</li>
      </ul>
    </div>
    
    <div class="plan-card">
      <h3>📊 Advanced Transaction Analysis <span class="badge">Planned</span></h3>
      <p>Gain deeper insights into your spending patterns and financial health.</p>
      <ul>
        <li>Spending pattern analysis</li>
        <li>Budget recommendations</li>
        <li>Financial health scoring</li>
        <li>Predictive analytics</li>
        <li>Custom report generation</li>
      </ul>
    </div>
  </div>
</div>

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

<a href="https://github.com/jinliangwii/MoneyFlow" class="btn-primary">View on GitHub</a>
<a href="Docs/README.md" class="btn-primary">Browse Documentation</a>

## License

This project is licensed under the MIT License.

## Links

- [GitHub Repository](https://github.com/jinliangwii/MoneyFlow)
- [Documentation Index](Docs/README.md)
