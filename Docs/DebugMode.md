# Debug Mode Documentation

## Overview

Debug Mode is a development feature that allows developers to manually load sample data into the MoneyFlow app during testing and development. This feature is **only available in Debug builds** and will not appear in Release builds.

## Purpose

Previously, the app automatically seeded sample accounts and transactions when the data store was empty. This behavior has been removed to provide a clean initial state. Debug Mode provides a controlled way to load example data only when needed during development.

## Features

### Debug Mode Toggle

- **Location**: Settings → Debug Section
- **Availability**: Only visible in Debug builds (`#if DEBUG`)
- **Function**: Enables/disables debug mode and automatically loads/clears sample data
  - **When Enabled**: Automatically loads all sample accounts and transactions
  - **When Disabled**: Automatically clears all accounts and transactions

### Sample Data Loading

When Debug Mode is enabled, three buttons become available:

1. **加载示例账户** (Load Sample Accounts)
   - Loads 5 sample accounts:
     - 高盛储蓄 (Marcus deposit account)
     - Chase Checking 6153 (Chase checking account)
     - Apple Card (Apple credit card)
     - E*TRADE(6776) (E*TRADE brokerage account)
     - Apple Savings (Apple savings account)

2. **加载示例交易** (Load Sample Transactions)
   - Loads sample transactions linked to existing accounts
   - Requires accounts to be present first
   - Creates transactions for:
     - Whole Foods (groceries)
     - Uber (transport)
     - Salary (income)
     - Netflix (entertainment)
     - Apple Card Payment (other)

3. **加载所有示例数据** (Load All Sample Data)
   - Convenience button that loads both accounts and transactions
   - Equivalent to pressing both buttons above

## Implementation Details

### Code Structure

#### AppState.swift

Debug mode methods are conditionally compiled:

```swift
#if DEBUG
/// Loads sample accounts. Only available in debug builds.
func loadSampleAccounts() {
    accounts = Account.sample
    Task {
        await save()
    }
}

/// Loads sample transactions linked to existing accounts. Only available in debug builds.
func loadSampleTransactions() {
    transactions = Transaction.sampleLinked(accounts: accounts)
    Task {
        await save()
    }
}

/// Loads both sample accounts and transactions. Only available in debug builds.
func loadSampleData() {
    loadSampleAccounts()
    loadSampleTransactions()
}
#endif
```

#### Views_SettingsView.swift

The debug section is conditionally compiled:

```swift
#if DEBUG
Section("调试") {
    Toggle(isOn: $debugModeEnabled) {
        Text("调试模式")
    }
    .onChange(of: debugModeEnabled) { _, newValue in
        app.loadSampleData()  // Automatically loads/clears based on toggle state
    }
    
    // Debug buttons appear here (text changes based on toggle state)
}
#endif
```

### Data Persistence

- Sample data loaded via Debug Mode is saved to the data store using the standard persistence mechanism
- Data persists across app launches
- To clear data, delete the app or manually remove accounts/transactions

### Changes from Previous Implementation

**Before:**
- `AppState.load()` automatically seeded sample data when data store was empty
- Sample data appeared on first launch automatically

**After:**
- `AppState.load()` only loads existing persisted data
- No automatic seeding
- Sample data can only be loaded manually via Debug Mode (Debug builds only)

**Removed Features:**
- Automatic sample data seeding in `AppState.load()`
- "Load Sample Data" button from ImportView

## Usage

### For Developers

1. Build the app in **Debug** configuration
2. Navigate to **Settings**
3. Find the **调试** (Debug) section
4. Toggle **调试模式** (Debug Mode):
   - **Toggle ON**: Automatically loads all sample accounts and transactions
   - **Toggle OFF**: Automatically clears all accounts and transactions
5. Alternatively, use the manual buttons for granular control:
   - **加载示例账户**: Load only sample accounts
   - **加载示例交易**: Load only sample transactions (requires accounts first)
   - **加载所有示例数据**: Load both accounts and transactions

### For Testers

Debug Mode is not available in Release builds. Testers will need to:
- Manually add accounts via the Add Account view
- Manually add transactions via Import or other means
- Or use a Debug build for testing with sample data

## Technical Notes

### Build Configuration

- Debug Mode features are guarded by `#if DEBUG` compiler directives
- In Release builds, these sections are completely removed from the compiled binary
- No runtime checks are needed - the code simply doesn't exist in Release builds

### Sample Data Location

Sample data definitions remain in `Models.swift`:
- `Account.sample` - Static property returning sample accounts
- `Transaction.sampleLinked(accounts:)` - Function that creates sample transactions linked to provided accounts

These are now only accessed via Debug Mode methods, but remain available for other debug/testing purposes.

## UI Tests

Comprehensive UI tests have been created to verify Debug Mode functionality. The tests are located in `MoneyFlowUITests/DebugModeUITests.swift`.

### Test Coverage

The UI tests verify:

1. **Debug Mode Visibility**
   - `testDebugModeSectionVisibleInDebugBuild()` - Verifies debug section exists in debug builds
   - `testDebugModeButtonsVisibleWhenEnabled()` - Verifies buttons appear when toggle is enabled
   - `testDebugModeButtonsTextChangesWithToggle()` - Verifies button text changes based on toggle state

2. **Debug Mode Functionality**
   - `testEnablingDebugModeLoadsSampleData()` - Verifies enabling toggle loads sample data
   - `testDisablingDebugModeClearsData()` - Verifies disabling toggle clears data
   - `testManualLoadSampleAccounts()` - Verifies manual account loading works
   - `testManualLoadSampleTransactions()` - Verifies manual transaction loading works

3. **Release Build Verification**
   - `testDebugModeNotVisibleInReleaseBuild()` - Verifies debug mode does NOT exist in release builds
     - **Note**: This test will fail in Debug builds (expected) and pass in Release builds

### Running the Tests

#### Debug Build Tests
```bash
# Run all debug mode tests (Debug build)
xcodebuild test -scheme MoneyFlow -destination 'platform=macOS' -only-testing:MoneyFlowUITests/DebugModeUITests
```

#### Release Build Verification
To verify debug mode is NOT included in Release builds:

1. Build the app in Release configuration
2. Run UI tests against the Release build:
```bash
xcodebuild test -scheme MoneyFlow -configuration Release -destination 'platform=macOS' -only-testing:MoneyFlowUITests/DebugModeUITests/testDebugModeNotVisibleInReleaseBuild
```

The `testDebugModeNotVisibleInReleaseBuild` test should pass when run against a Release build, confirming that debug mode elements are not present.

### Test Notes

- All tests use `#if DEBUG` guarded code blocks, so they're designed to run in Debug builds
- The release build test serves as documentation and verification that debug mode is properly excluded
- Tests include proper waiting for UI elements and data loading/clearing operations
- Helper methods are provided for navigation between app sections

## Future Enhancements

Potential improvements for Debug Mode:
- Add ability to clear all data
- Add ability to generate random sample data
- Add ability to load different sample data sets
- Add data export/import functionality
- Add debugging information display (data counts, last save time, etc.)

