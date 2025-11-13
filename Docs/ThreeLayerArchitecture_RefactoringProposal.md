# Three-Layer Architecture Refactoring Proposal

## Executive Summary

This proposal reorganizes the account-related code into three clear layers:
1. **Data Source Layer** - Raw data extraction from files/APIs
2. **Model Layer (Account Providers)** - Account creation and feature definitions
3. **Import/Integration Layer** - Orchestrates data flow from sources to accounts

## Current Problems

### Code Scattering Issues

1. **UI Code Spread Across Multiple Locations**:
   - Import views: `Accounts/{Account}/UI/` (e.g., `CMBCreditCardImportView.swift`)
   - Configuration views: `Accounts/{Account}/UI/` (e.g., `CMBCreditCardAccountImportConfigurationView.swift`)
   - Properties pane: `UI/Properties/Components/`
   - Debug views: `UI/Debug/`
   - Account-specific UI logic embedded in generic components (e.g., Alipay processing rules in `AccountHeaderCard.swift`)

2. **Cross-Layer Dependencies**:
   - Feature Providers (`Providers/`) depend on Import Services (`Accounts/{Account}/Services/`)
   - Import Services depend on Data Sources (`DataSources/`)
   - UI components directly reference account-specific logic

3. **Unclear Boundaries**:
   - Import services mix data extraction logic with business logic
   - Feature providers contain UI code (buttons, views)
   - Data sources contain repository logic

## Proposed Three-Layer Architecture

```
MoneyFlow/
├── DataSources/                          # Layer 1: Data Source Layer
│   ├── Protocol/
│   │   ├── DataSourceProtocol.swift
│   │   ├── RawTransactionProtocol.swift
│   │   └── DataSourceFactory.swift
│   │
│   └── Implementations/
│       ├── CMBCreditCard/
│       │   ├── CMBCreditCardPDFDataSource.swift
│       │   ├── Models/
│       │   │   └── RawCMBCreditCardTransaction.swift
│       │   ├── Parsers/
│       │   │   ├── CMBCreditCardPDFParser.swift
│       │   │   └── [other parsers]
│       │   └── Repositories/
│       │       └── CMBCreditCardRepository.swift
│       │
│       ├── CMBPersonalChecking/
│       ├── CMBExpressLoan/
│       ├── Alipay/
│       ├── WeChat/
│       └── Plaid/
│
├── Providers/                            # Layer 2: Model Layer (Account Providers)
│   ├── Protocol/
│   │   ├── AccountProvider.swift          # Account creation
│   │   ├── AccountFeatureProvider.swift    # Account features
│   │   └── ProviderRegistry.swift
│   │
│   └── Implementations/
│       ├── CMBCreditCard/
│       │   ├── CMBCreditCardAccountProvider.swift
│       │   └── CMBCreditCardAccountFeatureProvider.swift
│       │
│       ├── CMBPersonalChecking/
│       ├── CMBExpressLoan/
│       ├── Alipay/
│       ├── WeChat/
│       ├── Plaid/
│       └── DIY/
│
├── Integration/                          # Layer 3: Import/Integration Layer
│   ├── Services/                          # Import orchestration
│   │   ├── BaseImportService.swift
│   │   ├── ImportServiceProtocols.swift
│   │   └── [Account-specific import services]
│   │       ├── CMBCreditCard/
│   │       │   ├── CMBCreditCardImportService.swift
│   │       │   └── CMBCreditCardTransactionConverter.swift
│   │       ├── CMBPersonalChecking/
│   │       ├── CMBExpressLoan/
│   │       ├── Alipay/
│   │       ├── WeChat/
│   │       └── Plaid/
│   │
│   └── Domain/                            # Import domain models
│       └── [Account-specific batch models]
│           ├── CMBCreditCard/
│           │   └── ImportCMBCreditCardTransactionBatch.swift
│           └── [other accounts]
│
├── Core/                                  # Core domain models (shared)
│   └── Domain/
│       └── Models.swift                   # Account, Transaction, etc. (location to be confirmed)
│
└── UI/                                    # All UI components (presentation layer)
    ├── Properties/
    │   ├── PropertiesView.swift
    │   ├── AddAccountView.swift
    │   └── Components/
    │       ├── AccountDetailPane.swift
    │       ├── AccountHeaderCard.swift
    │       └── [other components]
    │
    ├── Import/                            # Import UI (consolidated)
    │   ├── Components/                    # Generic import components
    │   │   ├── FileSelectionView.swift
    │   │   ├── ProgressIndicator.swift
    │   │   └── ImportResultView.swift
    │   │
    │   └── AccountSpecific/                # Account-specific import views
    │       ├── CMBCreditCard/
    │       │   ├── CMBCreditCardImportView.swift
    │       │   └── CMBCreditCardImportConfigurationView.swift
    │       ├── CMBPersonalChecking/
    │       ├── CMBExpressLoan/
    │       ├── Alipay/
    │       ├── WeChat/
    │       └── Plaid/
    │
    └── Debug/                              # Debug views
        └── [debug views]
```

## Layer Responsibilities

### Layer 1: Data Source Layer (`DataSources/`)

**Purpose**: Extract raw data from external sources (PDFs, CSVs, APIs, etc.)

**Responsibilities**:
- Parse source files/APIs
- Extract account metadata
- Extract raw transactions
- Provide source information (date ranges, page counts, etc.)
- Save raw transactions to repositories

**What it DOES**:
- ✅ File parsing (PDF, CSV, Excel, etc.)
- ✅ API communication (Plaid, etc.)
- ✅ Raw data extraction
- ✅ Account metadata extraction
- ✅ Repository persistence

**What it DOES NOT**:
- ❌ Business logic (duplicate detection, filtering)
- ❌ Transaction conversion to domain models
- ❌ UI components
- ❌ Account creation/management

**Key Files**:
- `DataSourceProtocol.swift` - Protocol for all data sources
- `{Account}PDFDataSource.swift` - Account-specific data source implementations
- `{Account}Repository.swift` - Raw transaction persistence

### Layer 2: Model Layer (`Providers/`)

**Purpose**: Define account types, their creation, and their features

**Responsibilities**:
- Account type definitions
- Account creation UI and validation
- Feature definitions (what features an account type supports)
- Feature provider registration

**What it DOES**:
- ✅ Account creation flow (UI + validation)
- ✅ Feature capability definitions (import, sync, etc.)
- ✅ Account type metadata (icons, display names, etc.)
- ✅ Feature provider registration

**What it DOES NOT**:
- ❌ Import implementation (delegates to Integration Layer)
- ❌ Data extraction (delegates to Data Source Layer)
- ❌ UI implementation (UI is in UI/ folder, Providers provide data/config)

**Key Files**:
- `AccountProvider.swift` - Account creation protocol
- `AccountFeatureProvider.swift` - Feature definition protocol
- `{Account}AccountProvider.swift` - Account creation implementation
- `{Account}AccountFeatureProvider.swift` - Feature definition implementation

**Example - Feature Provider (No Implementation Details)**:
```swift
struct CMBCreditCardAccountFeatureProvider: AccountFeatureProvider {
    func makeImportView(for account: Account) -> AnyView? {
        // Returns view from Integration Layer, not implementation
        AnyView(CMBCreditCardImportView(account: account))
    }
    
    func performImport(...) async throws -> ImportResult {
        // Delegates to Integration Layer service
        let service = CMBCreditCardImportService(...)
        return try await service.import(...)
    }
}
```

### Layer 3: Integration Layer (`Integration/`)

**Purpose**: Orchestrate data flow from Data Sources to Accounts

**Responsibilities**:
- Import orchestration (using Data Sources)
- Transaction conversion (raw → domain)
- Duplicate detection
- Business logic (filtering, validation)
- Provides data/services for UI to consume

**What it DOES**:
- ✅ Coordinates Data Source Layer and Model Layer
- ✅ Implements import business logic
- ✅ Converts raw transactions to domain transactions
- ✅ Handles duplicate detection
- ✅ Manages import batches
- ✅ Provides import services that UI consumes

**What it DOES NOT**:
- ❌ File parsing (delegates to Data Source Layer)
- ❌ Account creation (delegates to Model Layer)
- ❌ Feature definitions (delegates to Model Layer)
- ❌ UI components (UI stays in UI/ folder)

**Key Files**:
- `BaseImportService.swift` - Common import flow
- `{Account}ImportService.swift` - Account-specific import logic
- `{Account}TransactionConverter.swift` - Raw → Domain conversion
- `Import{Account}TransactionBatch.swift` - Import batch models

**Note**: UI components consume services from this layer but live in `UI/` folder. Integration layer provides data/logic to control/customize how UI displays.

## Migration Strategy

### Phase 1: Consolidate UI (Low Risk)

**Goal**: Move all account-specific import UI to `UI/Import/AccountSpecific/`

**Steps**:
1. Create `UI/Import/AccountSpecific/` structure
2. Move import views from `Accounts/{Account}/UI/` to `UI/Import/AccountSpecific/{Account}/`
3. Move configuration views to same location
4. Update imports in Feature Providers
5. Remove empty `Accounts/{Account}/UI/` folders

**Files to Move**:
- `Accounts/CMBCreditCard/UI/CMBCreditCardImportView.swift` → `UI/Import/AccountSpecific/CMBCreditCard/`
- `Accounts/CMBCreditCard/UI/CMBCreditCardAccountImportConfigurationView.swift` → `UI/Import/AccountSpecific/CMBCreditCard/`
- (Repeat for all accounts)

**Note**: All UI stays in `UI/` folder. Integration layer provides services/data that UI consumes.

### Phase 2: Move Import Services (Medium Risk)

**Goal**: Move import services to `Integration/Services/`

**Steps**:
1. Create `Integration/Services/` structure
2. Move `Accounts/{Account}/Services/{Account}ImportService.swift` to `Integration/Services/{Account}/`
3. Move `Accounts/{Account}/Services/{Account}TransactionConverter.swift` to same location
4. Move `Accounts/Shared/Import/` to `Integration/Services/Shared/`
5. Update imports in Feature Providers and UI
6. Update `AppDependencies` if needed

**Files to Move**:
- `Accounts/CMBCreditCard/Services/CMBCreditCardImportService.swift` → `Integration/Services/CMBCreditCard/`
- `Accounts/Shared/Import/BaseImportService.swift` → `Integration/Services/Shared/`
- (Repeat for all accounts)

### Phase 3: Move Domain Models (Low Risk)

**Goal**: Move import batch models to `Integration/Domain/`

**Steps**:
1. Create `Integration/Domain/` structure
2. Move `Accounts/{Account}/Domain/Import{Account}TransactionBatch.swift` to `Integration/Domain/{Account}/`
3. Update imports in Import Services
4. Remove empty `Accounts/{Account}/Domain/` folders (if only import batches)

**Files to Move**:
- `Accounts/CMBCreditCard/Domain/ImportCMBCreditCardTransactionBatch.swift` → `Integration/Domain/CMBCreditCard/`
- (Repeat for all accounts)

### Phase 4: Clean Up Accounts Folder (Low Risk) ✅ COMPLETE

**Goal**: Remove or repurpose `Accounts/` folder

**Completed Actions**:
- Moved `Accounts/Shared/Integration/` files to `Integration/Shared/`
- Moved `Accounts/Shared/` utility files to `Integration/Shared/`
- Removed empty account folders (CMBCreditCard, CMBExpressLoan, WeChat, Shared)
- Kept account folders with documentation (CMBPersonalChecking, Plaid)

**Final State**: `Accounts/` folder now only contains:
- Documentation files (README.md, SETUP.md)
- Empty folders (Alipay3rdPartyTx - only .DS_Store)

## Benefits

### 1. Clear Separation of Concerns

- **Data Sources**: Only care about extracting data
- **Providers**: Only care about account definitions and features
- **Integration**: Only cares about orchestrating the flow

### 2. Reduced Coupling

- Feature Providers no longer need to know import implementation details
- Import Services no longer need to know account creation details
- Data Sources are completely independent

### 3. Easier Testing

- Each layer can be tested independently
- Mock boundaries are clear
- Integration tests focus on layer interactions

### 4. Better Maintainability

- All import-related code in one place (`Integration/`)
- All data extraction code in one place (`DataSources/`)
- All account definitions in one place (`Providers/`)

### 5. Scalability

- Adding new account types: Add to all three layers independently
- Adding new data sources: Only touch Data Source Layer
- Adding new import features: Only touch Integration Layer

## Example: Adding a New Account Type

### Step 1: Data Source Layer
Create `DataSources/Implementations/NewBank/NewBankPDFDataSource.swift`

### Step 2: Model Layer
Create:
- `Providers/Implementations/NewBank/NewBankAccountProvider.swift`
- `Providers/Implementations/NewBank/NewBankAccountFeatureProvider.swift`

### Step 3: Integration Layer
Create:
- `Integration/Services/NewBank/NewBankImportService.swift`
- `Integration/Domain/NewBank/ImportNewBankTransactionBatch.swift`

### Step 4: UI Layer
Create:
- `UI/Import/AccountSpecific/NewBank/NewBankImportView.swift`
- `UI/Import/AccountSpecific/NewBank/NewBankImportConfigurationView.swift` (if needed)

## Migration Checklist

- [ ] Phase 1: Consolidate UI
  - [ ] Create `UI/Import/AccountSpecific/` structure
  - [ ] Move all import views from `Accounts/{Account}/UI/`
  - [ ] Move all configuration views
  - [ ] Update imports in Feature Providers
  - [ ] Test UI functionality

- [ ] Phase 2: Move Import Services
  - [ ] Create `Integration/Services/` structure
  - [ ] Move all import services
  - [ ] Move shared import code
  - [ ] Update imports
  - [ ] Test import functionality

- [ ] Phase 3: Move Domain Models
  - [ ] Create `Integration/Domain/` structure
  - [ ] Move import batch models
  - [ ] Update imports
  - [ ] Test domain models

- [ ] Phase 4: Clean Up
  - [ ] Review remaining `Accounts/` content
  - [ ] Decide on `Accounts/` folder future
  - [ ] Update documentation
  - [ ] Final testing

## Decisions Made

1. **Account Lifecycle Management**: 
   - **Decision**: Stays in `Providers/` for now
   - Account creation/editing/deletion logic lives in the Model Layer (Providers)

2. **Shared Domain Models**: 
   - **Decision**: Need more info, but looks like `Core/Domain/Models.swift` related
   - Account-agnostic domain models (e.g., `Account`, `Transaction`) appear to be core domain models
   - **Note**: Final location to be confirmed, but likely `Core/Domain/Models.swift`

3. **UI Components in Properties Pane**: 
   - **Decision**: All UI stays in `UI/` folder
   - Integration layer uses data to control/customize how UI shows
   - Account-specific UI components in `UI/Properties/Components/` stay in Properties
   - Integration layer provides data/services that UI consumes, but doesn't contain UI components

4. **Debug Views**: 
   - **Decision**: UI goes to `UI/` folder
   - Debug views stay in `UI/Debug/`
   - All UI components remain in the `UI/` folder structure

## Architecture Principles

### UI Layer Separation
- **All UI stays in `UI/` folder** - This is the presentation layer
- **Integration layer provides data/logic** - Services and domain models that UI consumes
- **UI controls presentation** - How data is displayed, user interactions
- **Integration controls business logic** - What data to show, how to process it

### Data Flow
```
Data Source Layer → Integration Layer → UI Layer
     (raw data)    (business logic)   (presentation)
```

1. **Data Source Layer** extracts raw data
2. **Integration Layer** processes data (conversion, validation, business rules)
3. **UI Layer** displays data and handles user interactions

### Example: Import Flow
1. User clicks import button in `UI/Properties/Components/AccountHeaderCard.swift`
2. Feature Provider (`Providers/`) provides import view reference
3. Import View (`UI/Import/AccountSpecific/{Account}/`) is displayed
4. Import View calls Import Service (`Integration/Services/{Account}/`)
5. Import Service uses Data Source (`DataSources/Implementations/{Account}/`)
6. Data flows back: Data Source → Import Service → UI updates

## Potential Downsides & Trade-offs

### 1. **Increased File Navigation Complexity**

**Issue**: Files are now spread across more folders, making it harder to find related code.

**Impact**:
- To understand CMB Credit Card import, you need to look in:
  - `DataSources/Implementations/CMBCreditCard/` (data extraction)
  - `Providers/Implementations/CMBCreditCard/` (account definition)
  - `Integration/Services/CMBCreditCard/` (import logic)
  - `UI/Import/AccountSpecific/CMBCreditCard/` (UI)
  - `Core/Domain/Models.swift` (domain models)

**Mitigation**:
- Use IDE "Go to Definition" and "Find References"
- Good naming conventions help
- Consider IDE workspace/bookmark features

### 2. **Longer Import Paths**

**Issue**: Import statements get longer and more verbose.

**Example**:
```swift
// Before (co-located):
import Foundation  // Everything in same folder

// After (separated):
import MoneyFlow.Integration.Services.CMBCreditCard
import MoneyFlow.DataSources.Implementations.CMBCreditCard
import MoneyFlow.UI.Import.AccountSpecific.CMBCreditCard
```

**Impact**: 
- More typing
- More imports to manage
- Potential for circular dependencies if not careful

**Mitigation**:
- Use typealiases for common paths
- Group related imports
- Swift's module system helps prevent circular dependencies

### 3. **Feature Provider Complexity**

**Issue**: Feature Providers become "glue code" that bridges multiple layers.

**Current Pattern**:
```swift
struct CMBCreditCardAccountFeatureProvider {
    func makeImportView(...) -> AnyView? {
        // Must reference UI in UI/ folder
        AnyView(CMBCreditCardImportView(...))
    }
    
    func performImport(...) async throws -> ImportResult {
        // Must reference Integration service
        let service = CMBCreditCardImportService(...)
        return try await service.import(...)
    }
}
```

**Impact**:
- Feature Providers need to know about both Integration and UI layers
- They become a dependency hub
- Changes in one layer might require updates in Providers

**Mitigation**:
- Keep Feature Providers thin - they're just coordinators
- Use dependency injection to reduce coupling
- Consider if some logic should move to Integration layer

### 4. **Migration Disruption**

**Issue**: Moving files breaks git history and requires careful coordination.

**Impact**:
- Git blame/history becomes fragmented
- Team members need to update their local branches
- Risk of merge conflicts during migration
- Temporary broken state during migration phases

**Mitigation**:
- Do migration in phases (as proposed)
- Use git moves (`git mv`) to preserve history
- Coordinate with team
- Have rollback plan

### 5. **Potential Over-Engineering**

**Issue**: For a small codebase, this might be more structure than needed.

**Questions to Consider**:
- How many account types do you have? (Currently: 6)
- How many developers work on this? (Small team = less need for strict boundaries)
- How often do you add new account types? (Frequent = more benefit)

**Impact**:
- More boilerplate to add new accounts
- More cognitive overhead to understand structure
- Might slow down development initially

**Mitigation**:
- This structure pays off as you scale
- Template/boilerplate generators can help
- Good documentation reduces cognitive load

### 6. **Testing Complexity**

**Issue**: More layers means more boundaries to mock in tests.

**Impact**:
- Integration tests need to mock Data Source, Integration Service, and UI
- Unit tests need to understand layer boundaries
- More setup code for tests

**Mitigation**:
- Clear protocols at each layer boundary
- Dependency injection makes mocking easier
- Test utilities/helpers for common patterns

### 7. **UI-Integration Coupling Risk**

**Issue**: Even though UI is separate, it still needs to know about Integration services.

**Example**:
```swift
// UI/Import/AccountSpecific/CMBCreditCard/CMBCreditCardImportView.swift
struct CMBCreditCardImportView: View {
    // Still needs to import Integration layer
    let importService = CMBCreditCardImportService(...)
}
```

**Impact**:
- UI can't be completely independent
- Changes to Integration services might require UI updates
- Not a true "presentation-only" layer

**Mitigation**:
- This is expected - UI consumes services
- Use ViewModels/ViewState to decouple further if needed
- Keep UI focused on presentation, business logic in Integration

### 8. **Learning Curve**

**Issue**: New developers need to understand the three-layer architecture.

**Impact**:
- "Where does this code go?" questions
- Need to understand layer responsibilities
- Might put code in wrong layer initially

**Mitigation**:
- Clear documentation (like this proposal)
- Code review to catch misplacements
- Examples/templates for each layer
- Onboarding guide

### 9. **Account-Specific Logic Still Scattered**

**Issue**: Even after refactoring, account-specific code is still split across 4 locations.

**Impact**:
- To understand one account type, you still need to look in multiple places
- Harder to see "the big picture" of one account type

**Mitigation**:
- This is a trade-off: we prioritize layer separation over account cohesion
- Documentation can help (one doc per account type)
- IDE bookmarks/favorites for each account type

### 10. **Potential Performance Overhead**

**Issue**: More layers = more indirection = potential performance cost.

**Impact**:
- Negligible in practice (Swift's compiler optimizes)
- More function calls through layers
- More protocol conformance checks

**Mitigation**:
- Profile if performance becomes an issue
- Swift's compiler is very good at optimizing
- The overhead is likely negligible compared to I/O operations

## When This Architecture Makes Sense

✅ **Good fit if**:
- You have 5+ account types (you have 6)
- You plan to add more account types
- Multiple developers work on the codebase
- You want clear separation for testing
- You value maintainability over simplicity

❌ **Might be overkill if**:
- You have 1-2 account types
- Solo developer
- Codebase is very small (< 10K LOC)
- You rarely add new account types

## Alternative: Simpler Structure

If the downsides are too much, consider a simpler approach:

```
MoneyFlow/
├── Accounts/
│   └── {Account}/
│       ├── DataSource.swift
│       ├── ImportService.swift
│       ├── Provider.swift
│       └── ImportView.swift
└── UI/
    └── Properties/
```

**Trade-off**: Less separation, but everything for one account is co-located.

## Recommendation

Given your current state (6 account types, ongoing development), **the three-layer architecture is worth it** because:

1. ✅ You're already experiencing code scattering (the problem this solves)
2. ✅ You have enough accounts to benefit from structure
3. ✅ The migration can be done incrementally (low risk)
4. ✅ The downsides are manageable with good tooling/documentation

The main risk is **migration disruption** - mitigate this by:
- Doing it in phases
- Good communication with team
- Comprehensive testing after each phase

## Next Steps

1. ✅ Review this proposal
2. ✅ Decisions made on architecture questions
3. ✅ Downsides identified and mitigated
4. Create detailed migration plan for Phase 1
5. Begin Phase 1 migration (lowest risk, highest visibility)

