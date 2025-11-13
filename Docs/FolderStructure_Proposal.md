# Folder Structure Revision Proposal

## Current Issues

1. **Parsers vs Data Sources Separation**: Parsers are in `Accounts/{Account}/Services/` but Data Sources are in `Core/Data/Sources/`. Since data sources now wrap parsers, they should be co-located.

2. **Provider Naming Inconsistency**: `Features/Providers/` vs `Accounts/{Account}/Provider/` (singular vs plural)

3. **Mixed Concerns**: UI components are split between `UI/` and `Accounts/{Account}/UI/`

4. **Data Source Location**: Data sources are in Core but are account-specific implementations

## Proposed Structure

```
MoneyFlow/
├── App/                              # App entry point
│   ├── MoneyFlowApp.swift
│   └── ContentView.swift
│
├── Core/                              # Core infrastructure (framework-level)
│   ├── Domain/                        # Core domain models
│   │   ├── Models.swift               # Account, Transaction, etc.
│   │   ├── Errors.swift
│   │   ├── Money.swift
│   │   └── Accounts/
│   │       └── AccountTraits.swift    # Account type behavior
│   │
│   ├── Data/                          # Core data infrastructure
│   │   ├── Persistence/
│   │   │   └── FilePersistence.swift
│   │   ├── Repositories/
│   │   │   ├── AccountRepositoryProtocol.swift
│   │   │   ├── TransactionRepositoryProtocol.swift
│   │   │   └── [implementations]
│   │   ├── Protocols.swift
│   │   └── DataLifecycleManager.swift
│   │
│   ├── State/
│   │   └── AppState.swift
│   │
│   ├── Dependencies/
│   │   └── AppDependencies.swift
│   │
│   ├── ErrorHandling/
│   │   └── ErrorHandling.swift
│   │
│   ├── DesignSystem/                  # Shared UI components
│   │   ├── Card.swift
│   │   ├── Colors.swift
│   │   ├── Formatters.swift
│   │   └── InstitutionIcons.swift
│   │
│   └── UseCases/                      # Core business use cases
│       ├── CalculateDashboardMetricsUseCase.swift
│       ├── CalculateSummaryMetricsUseCase.swift
│       └── [metrics types]
│
├── DataSources/                       # Data source implementations (NEW)
│   ├── Protocol/
│   │   ├── DataSourceProtocol.swift
│   │   ├── DataSourceFactory.swift
│   │   ├── DataSourceIntegration.swift
│   │   └── AccountCreator.swift
│   │
│   └── Implementations/               # Account-specific data sources
│       ├── CMBCreditCard/
│       │   ├── CMBCreditCardPDFDataSource.swift
│       │   └── Parsers/
│       │       └── CMBCreditCardPDFParser.swift
│       │
│       ├── CMBPersonalChecking/
│       │   ├── CMBPersonalCheckingPDFDataSource.swift
│       │   └── Parsers/
│       │       └── CMBPersonalCheckingPDFParser.swift
│       │
│       ├── CMBExpressLoan/
│       │   ├── CMBExpressLoanPDFDataSource.swift
│       │   └── Parsers/
│       │       └── CMBExpressLoanPDFParser.swift
│       │
│       ├── Alipay/
│       │   ├── AlipayCSVDataSource.swift
│       │   └── Parsers/
│       │       └── Alipay3rdPartyTxCSVParser.swift
│       │
│       ├── WeChat/
│       │   ├── WeChatExcelDataSource.swift
│       │   └── Parsers/
│       │       └── WeChatExcelParser.swift
│       │
│       └── Plaid/
│           └── PlaidAPIDataSource.swift
│
├── Accounts/                          # Account-specific business logic
│   ├── Shared/                        # Shared account infrastructure
│   │   ├── Import/
│   │   │   ├── BaseImportService.swift
│   │   │   ├── BaseImportServiceHelper.swift
│   │   │   ├── ImportServiceProtocols.swift
│   │   │   ├── ImportBatchManager.swift
│   │   │   ├── DuplicateDetectionStrategies.swift
│   │   │   ├── SequenceNumberStrategies.swift
│   │   │   ├── TransactionFilter.swift
│   │   │   ├── RepositoryTypeEraser.swift
│   │   │   └── ZipFileSourceParser.swift
│   │   │
│   │   └── AccountMetadataManager.swift
│   │
│   ├── CMBCreditCard/
│   │   ├── Domain/
│   │   │   ├── RawCMBCreditCardTransaction.swift
│   │   │   └── ImportCMBCreditCardTransactionBatch.swift
│   │   ├── Data/
│   │   │   └── CMBCreditCardRepository.swift
│   │   ├── Services/
│   │   │   ├── CMBCreditCardImportService.swift
│   │   │   ├── CMBCreditCardTransactionConverter.swift
│   │   │   └── [utility files]
│   │   └── UI/
│   │       ├── CMBCreditCardImportView.swift
│   │       └── CMBCreditCardAccountImportConfigurationView.swift
│   │
│   ├── CMBPersonalChecking/
│   │   └── [same structure]
│   │
│   ├── CMBExpressLoan/
│   │   └── [same structure]
│   │
│   ├── Alipay3rdPartyTx/
│   │   └── [same structure]
│   │
│   ├── WeChat/
│   │   └── [same structure]
│   │
│   ├── Plaid/
│   │   └── [same structure]
│   │
│   └── DIY/
│       └── [same structure]
│
├── Providers/                         # Account providers (renamed from Features/Providers)
│   ├── Protocol/
│   │   ├── AccountProvider.swift
│   │   ├── AccountFeatureProvider.swift
│   │   └── ProviderRegistry.swift
│   │
│   └── Implementations/               # Provider implementations
│       ├── CMBCreditCard/
│       │   ├── CMBCreditCardAccountProvider.swift
│       │   └── CMBCreditCardAccountFeatureProvider.swift
│       │
│       ├── CMBPersonalChecking/
│       │   └── [provider files]
│       │
│       ├── CMBExpressLoan/
│       │   └── [provider files]
│       │
│       ├── Alipay3rdPartyTx/
│       │   └── [provider files]
│       │
│       ├── WeChat/
│       │   └── [provider files]
│       │
│       ├── Plaid/
│       │   └── [provider files]
│       │
│       └── DIY/
│           └── [provider files]
│
├── Features/                          # Feature-based UI and business logic
│   ├── Dashboard/
│   │   ├── DashboardView.swift
│   │   ├── DashboardViewModel.swift
│   │   └── Components/
│   │
│   ├── Properties/
│   │   ├── PropertiesView.swift
│   │   ├── AddAccountView.swift
│   │   └── Components/
│   │
│   ├── Summary/
│   │   ├── SummaryView.swift
│   │   ├── SummaryViewModel.swift
│   │   └── Components/
│   │
│   ├── Settings/
│   │   └── SettingsView.swift
│   │
│   └── Services/                      # Feature-level services
│       ├── AnalyticsService.swift
│       └── ZipFileProcessingService.swift
│
└── UI/                                # Shared UI components
    ├── Components/
    │   └── AccountSelectionView.swift
    ├── Sidebar/
    │   └── Sidebar.swift
    └── Debug/
        └── [debug views]
```

## Key Changes

### 1. New `DataSources/` Folder
- **Why**: Data sources are account-specific implementations, not core infrastructure
- **Structure**: 
  - `Protocol/` for shared protocols and factories
  - `Implementations/{Account}/` for account-specific data sources
  - Parsers co-located with data sources in `Parsers/` subfolder

### 2. Renamed `Features/Providers/` → `Providers/`
- **Why**: Providers are a distinct architectural layer, not a feature
- **Structure**: `Protocol/` and `Implementations/{Account}/`

### 3. Clearer Separation
- **Core/**: Framework-level infrastructure (domain models, repositories, state)
- **DataSources/**: Data extraction and parsing implementations
- **Accounts/**: Account-specific business logic (import services, converters)
- **Providers/**: Account creation and feature registration
- **Features/**: Feature-based UI and business logic
- **UI/**: Shared UI components

### 4. Co-location of Related Files
- Parsers are now with their data sources
- Providers are grouped by account type
- Import infrastructure is in `Accounts/Shared/Import/`

## Migration Plan

1. **Phase 1**: Create new `DataSources/` structure
   - Move `Core/Data/Sources/` → `DataSources/Implementations/`
   - Move parsers from `Accounts/{Account}/Services/` → `DataSources/Implementations/{Account}/Parsers/`

2. **Phase 2**: Reorganize Providers
   - Move `Features/Providers/` → `Providers/`
   - Move `Accounts/{Account}/Provider/` → `Providers/Implementations/{Account}/`

3. **Phase 3**: Clean up Accounts structure
   - Ensure all account folders follow consistent structure
   - Move shared import infrastructure to `Accounts/Shared/Import/`

4. **Phase 4**: Update imports and verify build

## Benefits

1. **Clearer Architecture**: Each layer has a clear purpose
2. **Better Discoverability**: Related files are co-located
3. **Easier Maintenance**: Changes to parsing only affect DataSources folder
4. **Consistent Naming**: Singular/plural inconsistencies resolved
5. **Scalability**: Easy to add new account types following the pattern

