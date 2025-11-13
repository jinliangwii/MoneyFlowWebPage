# Data Lifecycle Management

## Overview

This document describes the data lifecycle management system implemented to prevent unbounded growth of JSON files and handle legacy data structures.

## Problem Statement

Without proper data management, the following issues occur:

1. **Unbounded Growth**: Raw transaction files and import batches accumulate indefinitely
2. **Legacy Data**: Old data structures from previous app versions persist
3. **Storage Bloat**: Large JSON files consume excessive disk space
4. **No Versioning**: No mechanism to migrate data when structures change

## Solution Architecture

### Components

1. **DataLifecycleManager** (`Core/Data/DataLifecycleManager.swift`)
   - Manages data versioning and migration
   - Implements retention policies
   - Performs periodic cleanup
   - Tracks data statistics

2. **RawTransactionCleanupHelper** (`Accounts/Shared/RawTransactionCleanupHelper.swift`)
   - Provides default cleanup implementation for raw transaction repositories
   - Filters processed transactions based on age

3. **Enhanced Repositories**
   - All raw transaction repositories implement cleanup methods
   - Import batch repositories support cleanup

## Data Retention Policy

### Default Policy

```swift
RetentionPolicy(
    rawTransactionRetentionDays: 90,      // Keep processed raw transactions for 90 days
    importBatchRetentionDays: 365,       // Keep import batches for 1 year
    keepUnprocessedRawTransactions: true // Keep unprocessed transactions indefinitely
)
```

### Rationale

- **Raw Transactions (90 days)**: After processing, raw transactions are only needed for debugging. 90 days provides a reasonable window for troubleshooting while preventing unbounded growth.
- **Import Batches (365 days)**: Import history is useful for auditing and troubleshooting. 1 year provides sufficient history.
- **Unprocessed Transactions**: Keep indefinitely for debugging import issues.

## Data Versioning

### Version Management

- Current version: `1`
- Version info stored in `dataVersion.json`
- Automatic migration on app launch if version mismatch detected

### Migration Process

1. Check current data version on app launch
2. If version < current, run migration steps
3. Update version file after successful migration
4. Log migration activities

### Adding New Migrations

When data structures change:

1. Increment `currentDataVersion` in `DataLifecycleManager`
2. Add migration method: `migrateToVersionN()`
3. Update `migrateData()` to call new migration
4. Test migration with old data

Example:
```swift
// In DataLifecycleManager
static let currentDataVersion = 2  // Increment

private func migrateData(from oldVersion: Int, to newVersion: Int) async throws {
    if oldVersion < 2 && newVersion >= 2 {
        try await migrateToVersion2()
    }
}

private func migrateToVersion2() async throws {
    // Migration logic here
}
```

## Automatic Cleanup

### Periodic Cleanup

- Runs automatically once per week (configurable)
- Triggered on app launch if 7+ days since last cleanup
- Cleans up:
  - Processed raw transactions older than retention period
  - Old import batches
  - Legacy data structures

### Manual Cleanup

Available in debug builds:
- `performDataMaintenance()` - Manually trigger cleanup
- `getDataStatistics()` - Get data size statistics

### Cleanup During Account Deletion

When accounts are cleared:
- All account-related JSON files are deleted
- Data lifecycle cleanup is triggered
- Ensures no orphaned data remains

## File Management

### Files Cleaned Up

When accounts are cleared, these files are removed:
- `*AccountMetadata.json` - Account-specific metadata
- `*ImportBatches.json` - Import batch history
- `raw*Transactions.json` - Raw transaction data
- `plaid*.json` - Plaid-specific data

### Files Preserved

- `accounts.json` - Core account data (saved as empty array)
- `transactions.json` - Core transaction data
- `dataVersion.json` - Version tracking

## Implementation Details

### Raw Transaction Cleanup

Repositories implement cleanup via protocol extension:

```swift
extension RawTransactionRepository where RawTransactionType: CleanupableRawTransaction {
    func cleanupProcessedRawTransactions(olderThan date: Date, keepUnprocessed: Bool) async throws -> Int {
        // Default implementation filters and saves
    }
}
```

### Import Batch Cleanup

Generic cleanup for import batch files:
- Decodes JSON arrays
- Filters by `importedAt` date
- Re-encodes and saves filtered data

### Data Statistics

Monitor data growth:
```swift
struct DataStatistics {
    var totalJSONFiles: Int
    var totalSizeBytes: Int64
    var rawTransactionFiles: Int
    var rawTransactionSizeBytes: Int64
    var importBatchFiles: Int
    var importBatchSizeBytes: Int64
    // ... more fields
}
```

## Usage

### Automatic (Default)

No action required - cleanup runs automatically:
- On app launch (if needed)
- When accounts are cleared
- Periodically (weekly)

### Manual (Debug Only)

```swift
// Trigger cleanup manually
await appState.performDataMaintenance()

// Get statistics
if let stats = await appState.getDataStatistics() {
    print("Total size: \(stats.totalSizeMB) MB")
    print("Raw transactions: \(stats.rawTransactionSizeMB) MB")
}
```

## Best Practices

1. **Version Increments**: Always increment version when changing data structures
2. **Migration Testing**: Test migrations with real data from previous versions
3. **Retention Tuning**: Adjust retention periods based on actual usage patterns
4. **Monitoring**: Use data statistics to monitor growth trends
5. **Backup**: Consider backing up data before major migrations

## Future Enhancements

Potential improvements:
- Configurable retention policies per account type
- Data compression for old data
- Archival to separate files
- User-configurable retention periods
- Export/import functionality for data migration

## Troubleshooting

### Cleanup Not Running

- Check `lastDataCleanupDate` in UserDefaults
- Verify DataLifecycleManager initialization
- Check console logs for errors

### Data Growing

- Verify cleanup methods are implemented in repositories
- Check retention policy settings
- Review data statistics to identify large files

### Migration Issues

- Check `dataVersion.json` for current version
- Review migration logs
- Test migrations with sample data first

