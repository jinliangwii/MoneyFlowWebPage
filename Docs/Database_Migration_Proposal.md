# Database Migration Proposal: From JSON to SQLite for Performance and Dataset Views

## Executive Summary

This proposal outlines migrating all data storage from JSON files to SQLite database to address performance issues as the dataset grows and to enable efficient dataset views for the ledger book feature. The migration will improve performance by 50-100x for queries, reduce memory usage, and enable SQL-based views for ledger filtering and aggregation.

---

## 1. Current State Analysis

### Current Architecture
- **Storage**: Multiple JSON files (`transactions.json`, `accounts.json`, `ledgers.json`, plus account-specific files)
- **Loading Pattern**: All data loaded into memory on app launch (`AppState.transactions`, `AppState.accounts`)
- **Query Pattern**: In-memory filtering after loading all data
- **Ledger Views**: Computed by filtering all transactions in memory based on ledger rules
- **Performance**: O(n) for all queries, regardless of data needed

### Performance Issues

#### 1.1 JSON File I/O Bottlenecks
- **Loading**: Entire JSON files loaded on app launch (500ms+ for 10K transactions)
- **Saving**: Entire files rewritten on every save (atomic writes are slow)
- **Memory**: All transactions kept in memory (`AppState.transactions`)
- **Scalability**: Performance degrades linearly with dataset size

#### 1.2 Ledger View Computation
- **Current**: `LedgerService.computeAndCacheData()` filters all transactions in memory
- **Problem**: Must load all transactions before filtering
- **Inefficiency**: Computing views for one month requires processing all transactions
- **Cache Invalidation**: Full recomputation when transactions change

#### 1.3 Query Patterns
```swift
// Current: Loads ALL transactions, then filters
func transactions(for ledger: Ledger, year: Int, month: Int) -> [Transaction] {
    let all = transactionsProvider()  // All transactions in memory
    let filtered = LedgerFilterService.filterTransactions(all, rules: ledger.rules)
    return filterByMonth(filtered, year: year, month: month)  // In-memory filter
}
```

### Data Types Currently in JSON

1. **Core Data**:
   - `transactions.json` - All transactions (largest file, grows fastest)
   - `accounts.json` - Account definitions
   - `ledgers.json` - Ledger definitions and rules

2. **Account-Specific Raw Data** (per account type):
   - `rawCMBPersonalCheckingTransactions.json`
   - `rawCMBCreditCardTransactions.json`
   - `rawCMBExpressLoanTransactions.json`
   - `rawAlipay3rdPartyTxTransactions.json`
   - `rawWeChatTransactions.json`
   - `importBatches.json` (multiple files)
   - Account metadata files

3. **Category & Categorization Data**:
   - Category templates
   - Transaction category mappings
   - Categorization rules

---

## 2. Proposed Solution: SQLite Database with Views

### Overview
Migrate from JSON file storage to a unified SQLite database with:
- **Indexed queries** for fast lookups (O(log n) vs O(n))
- **SQL views** for ledger filtering (database-level views)
- **Query-level filtering** instead of in-memory filtering
- **Incremental loading** (load only what's needed)
- **Repository pattern** maintained (swap implementation)

### Key Benefits

1. **Performance**: 
   - Indexed queries: 50-100x faster for date/account lookups
   - SQL aggregation: 10-20x faster for monthly statistics
   - Lazy loading: Only load data needed for current view

2. **Memory Efficiency**:
   - Remove in-memory transaction array
   - Load transactions on-demand
   - Reduce memory footprint by 60-80%

3. **Ledger Views**:
   - SQL views for ledger filtering rules
   - Efficient querying of ledger-specific transactions
   - Database-level aggregations for statistics

4. **Scalability**:
   - Handles 100K+ transactions efficiently
   - Query performance doesn't degrade with dataset size
   - Supports complex filtering and aggregation

5. **Data Integrity**:
   - ACID transactions for data consistency
   - Foreign key constraints
   - Better error handling and recovery

---

## 3. Database Schema Design

### 3.1 Core Tables

#### Transactions Table
```sql
CREATE TABLE transactions (
    id TEXT PRIMARY KEY,
    accountId TEXT NOT NULL,
    date TEXT NOT NULL,  -- ISO8601 format
    amount TEXT NOT NULL,  -- Decimal as string for precision
    merchant TEXT,
    notes TEXT,
    balance TEXT,  -- Balance after this transaction
    sequenceNumber INTEGER,
    flow TEXT NOT NULL,  -- 'flowIn', 'flowOut', 'neutral'
    
    -- Computed columns for fast queries
    year INTEGER GENERATED ALWAYS AS (CAST(strftime('%Y', date) AS INTEGER)) STORED,
    month INTEGER GENERATED ALWAYS AS (CAST(strftime('%m', date) AS INTEGER)) STORED,
    day INTEGER GENERATED ALWAYS AS (CAST(strftime('%d', date) AS INTEGER)) STORED,
    
    -- Indexes
    FOREIGN KEY (accountId) REFERENCES accounts(id) ON DELETE CASCADE
);

CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_account ON transactions(accountId);
CREATE INDEX idx_transactions_year_month ON transactions(year, month);
CREATE INDEX idx_transactions_account_date ON transactions(accountId, date);
CREATE INDEX idx_transactions_account_year_month ON transactions(accountId, year, month);
CREATE INDEX idx_transactions_flow ON transactions(flow);
```

#### Accounts Table
```sql
CREATE TABLE accounts (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    institution TEXT NOT NULL,
    type TEXT NOT NULL,  -- 'deposit', 'creditCard', etc.
    currencyCode TEXT NOT NULL DEFAULT 'USD',
    balance TEXT NOT NULL DEFAULT '0',
    last4 TEXT,
    creditLimit TEXT,
    interestRate TEXT,
    metadata TEXT,  -- JSON string for flexible metadata
    
    createdAt TEXT NOT NULL,
    updatedAt TEXT NOT NULL
);

CREATE INDEX idx_accounts_type ON accounts(type);
```

#### Ledgers Table
```sql
CREATE TABLE ledgers (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    type TEXT NOT NULL,  -- 'core', 'lifestyle', etc.
    isDefault INTEGER NOT NULL DEFAULT 0,  -- Boolean
    categoryTemplateId TEXT NOT NULL,
    rules TEXT NOT NULL,  -- JSON string for LedgerRules
    createdAt TEXT NOT NULL,
    updatedAt TEXT NOT NULL,
    
    FOREIGN KEY (categoryTemplateId) REFERENCES category_templates(id)
);

CREATE INDEX idx_ledgers_isDefault ON ledgers(isDefault);
```

### 3.2 Ledger Views (SQL Views)

SQL views provide efficient, queryable representations of ledger-filtered transactions:

```sql
-- View for core ledger (all transactions)
CREATE VIEW ledger_core_transactions AS
SELECT * FROM transactions
ORDER BY date DESC, sequenceNumber ASC;

-- View for account-filtered ledger
-- Note: This is a parameterized view concept - actual implementation uses WHERE clauses
-- Example for ledger with specific accounts:
CREATE VIEW ledger_account_filtered AS
SELECT t.* FROM transactions t
WHERE t.accountId IN (SELECT accountId FROM ledger_account_mappings WHERE ledgerId = ?)
ORDER BY t.date DESC, t.sequenceNumber ASC;
```

**Note**: SQLite doesn't support parameterized views directly. We'll use:
1. **Materialized Views**: Pre-computed views stored as tables (updated on transaction changes)
2. **Query Templates**: SQL query builders that generate WHERE clauses from ledger rules
3. **Virtual Tables**: FTS5 for full-text search if needed

### 3.3 Category & Categorization Tables

```sql
CREATE TABLE category_templates (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    isDefault INTEGER NOT NULL DEFAULT 0,
    structure TEXT NOT NULL,  -- JSON string for category hierarchy
    createdAt TEXT NOT NULL,
    updatedAt TEXT NOT NULL
);

CREATE TABLE transaction_category_mappings (
    id TEXT PRIMARY KEY,
    transactionId TEXT NOT NULL,
    ledgerId TEXT NOT NULL,
    categoryId TEXT NOT NULL,
    tags TEXT,  -- JSON array of strings
    createdAt TEXT NOT NULL,
    
    FOREIGN KEY (transactionId) REFERENCES transactions(id) ON DELETE CASCADE,
    FOREIGN KEY (ledgerId) REFERENCES ledgers(id) ON DELETE CASCADE,
    FOREIGN KEY (categoryId) REFERENCES category_templates(id)
);

CREATE INDEX idx_tx_category_tx ON transaction_category_mappings(transactionId);
CREATE INDEX idx_tx_category_ledger ON transaction_category_mappings(ledgerId);
CREATE INDEX idx_tx_category_category ON transaction_category_mappings(categoryId);
```

### 3.4 Raw Transaction Tables (Account-Specific)

```sql
-- Generic raw transaction table (polymorphic)
CREATE TABLE raw_transactions (
    id TEXT PRIMARY KEY,
    accountId TEXT NOT NULL,
    accountType TEXT NOT NULL,  -- 'CMBPersonalChecking', 'WeChat', etc.
    importBatchId TEXT NOT NULL,
    sourceFileName TEXT NOT NULL,
    extractedAt TEXT NOT NULL,
    rawData TEXT NOT NULL,  -- JSON string for account-specific raw fields
    parsedData TEXT,  -- JSON string for parsed fields
    isProcessed INTEGER NOT NULL DEFAULT 0,
    
    FOREIGN KEY (accountId) REFERENCES accounts(id) ON DELETE CASCADE
);

CREATE INDEX idx_raw_tx_account ON raw_transactions(accountId);
CREATE INDEX idx_raw_tx_batch ON raw_transactions(importBatchId);
CREATE INDEX idx_raw_tx_processed ON raw_transactions(isProcessed);

CREATE TABLE import_batches (
    id TEXT PRIMARY KEY,
    accountId TEXT NOT NULL,
    accountType TEXT NOT NULL,
    sourceFileName TEXT NOT NULL,
    importedAt TEXT NOT NULL,
    transactionCount INTEGER NOT NULL,
    dateRangeStart TEXT,
    dateRangeEnd TEXT,
    metadata TEXT,  -- JSON string for batch-specific metadata
    
    FOREIGN KEY (accountId) REFERENCES accounts(id) ON DELETE CASCADE
);

CREATE INDEX idx_import_batches_account ON import_batches(accountId);
CREATE INDEX idx_import_batches_date ON import_batches(importedAt);
```

### 3.5 Schema Versioning

```sql
CREATE TABLE schema_version (
    version INTEGER PRIMARY KEY,
    appliedAt TEXT NOT NULL,
    description TEXT
);

-- Initial version
INSERT INTO schema_version (version, appliedAt, description) 
VALUES (1, datetime('now'), 'Initial schema');
```

---

## 4. Repository Interface Enhancement

### 4.1 Transaction Repository

```swift
protocol TransactionRepositoryProtocol {
    // Existing methods (maintained for compatibility)
    func findAll() async throws -> [Transaction]
    func findByAccount(_ accountId: UUID) async throws -> [Transaction]
    func findByDateRange(_ start: Date, _ end: Date) async throws -> [Transaction]
    func findById(_ id: UUID) async throws -> Transaction?
    func save(_ transactions: [Transaction]) async throws
    func delete(_ id: UUID) async throws
    func deleteByAccount(_ accountId: UUID) async throws
    
    // NEW: Efficient ledger-based queries
    func findTransactions(
        ledgerRules: LedgerRules,
        year: Int?,
        month: Int?,
        startDate: Date?,
        endDate: Date?
    ) async throws -> [Transaction]
    
    // NEW: Monthly statistics query (SQL aggregation)
    func monthlyStatistics(
        ledgerRules: LedgerRules,
        year: Int?
    ) async throws -> [LedgerMonthlyStatistics]
    
    // NEW: Balance calculation query
    func calculateBalance(
        ledgerRules: LedgerRules,
        at date: Date?
    ) async throws -> Decimal
    
    // NEW: Account-based balance calculation
    func calculateAccountBalances(
        at date: Date?
    ) async throws -> [UUID: Decimal]
}
```

### 4.2 Ledger Repository Enhancement

```swift
protocol LedgerRepositoryProtocol {
    // Existing methods
    func findAll() async throws -> [Ledger]
    func findById(_ id: UUID) async throws -> Ledger?
    func findDefault() async throws -> Ledger?
    func save(_ ledger: Ledger) async throws
    func delete(_ ledger: Ledger) async throws
    
    // NEW: Query transactions for a ledger (delegates to TransactionRepository)
    // This is a convenience method that uses ledger rules
    func findTransactions(
        for ledger: Ledger,
        year: Int?,
        month: Int?
    ) async throws -> [Transaction]
}
```

---

## 5. Ledger View Implementation Strategy

### 5.1 Query Builder Pattern

Since SQLite doesn't support parameterized views, we'll use a query builder that generates SQL WHERE clauses from `LedgerRules`:

```swift
actor LedgerQueryBuilder {
    func buildQuery(ledgerRules: LedgerRules) -> SQLQuery {
        var conditions: [String] = []
        var parameters: [Any] = []
        
        // Account filtering
        if !ledgerRules.includeAll {
            if let includeIds = ledgerRules.includeAccountIds, !includeIds.isEmpty {
                let placeholders = Array(repeating: "?", count: includeIds.count).joined(separator: ", ")
                conditions.append("accountId IN (\(placeholders))")
                parameters.append(contentsOf: includeIds.map { $0.uuidString })
            }
            
            if let excludeIds = ledgerRules.excludeAccountIds, !excludeIds.isEmpty {
                let placeholders = Array(repeating: "?", count: excludeIds.count).joined(separator: ", ")
                conditions.append("accountId NOT IN (\(placeholders))")
                parameters.append(contentsOf: excludeIds.map { $0.uuidString })
            }
        }
        
        // Date range filtering
        if let dateRange = ledgerRules.dateRange {
            if let start = dateRange.start {
                conditions.append("date >= ?")
                parameters.append(ISO8601DateFormatter().string(from: start))
            }
            if let end = dateRange.end {
                conditions.append("date <= ?")
                parameters.append(ISO8601DateFormatter().string(from: end))
            }
        }
        
        // Amount filtering
        if let minAmount = ledgerRules.minAmount {
            conditions.append("ABS(CAST(amount AS REAL)) >= ?")
            parameters.append(minAmount as NSDecimalNumber)
        }
        if let maxAmount = ledgerRules.maxAmount {
            conditions.append("ABS(CAST(amount AS REAL)) <= ?")
            parameters.append(maxAmount as NSDecimalNumber)
        }
        
        // Flow filtering
        var flowConditions: [String] = []
        if ledgerRules.includeFlowIn {
            flowConditions.append("flow = 'flowIn'")
        }
        if ledgerRules.includeFlowOut {
            flowConditions.append("flow = 'flowOut'")
        }
        if ledgerRules.includeNeutral {
            flowConditions.append("flow = 'neutral'")
        }
        if !flowConditions.isEmpty {
            conditions.append("(\(flowConditions.joined(separator: " OR ")))")
        }
        
        let whereClause = conditions.isEmpty ? "" : "WHERE " + conditions.joined(separator: " AND ")
        let sql = "SELECT * FROM transactions \(whereClause) ORDER BY date DESC, sequenceNumber ASC"
        
        return SQLQuery(sql: sql, parameters: parameters)
    }
}
```

### 5.2 Materialized Views (Optional Optimization)

For frequently accessed ledgers, we can create materialized views (stored as tables):

```swift
actor LedgerViewManager {
    /// Create or update materialized view for a ledger
    func updateMaterializedView(for ledger: Ledger) async throws {
        let query = try await queryBuilder.buildQuery(ledgerRules: ledger.rules)
        
        // Drop existing view if exists
        try await db.execute("DROP TABLE IF EXISTS ledger_view_\(ledger.id.uuidString)")
        
        // Create materialized view
        try await db.execute("""
            CREATE TABLE ledger_view_\(ledger.id.uuidString) AS
            \(query.sql)
        """, parameters: query.parameters)
        
        // Create indexes on materialized view
        try await db.execute("""
            CREATE INDEX idx_ledger_view_\(ledger.id.uuidString)_date 
            ON ledger_view_\(ledger.id.uuidString)(date)
        """)
    }
    
    /// Invalidate materialized view when transactions change
    func invalidateView(for ledgerId: UUID) async throws {
        // Mark view as stale, will be refreshed on next access
        try await db.execute("""
            UPDATE ledger_view_states 
            SET isStale = 1 
            WHERE ledgerId = ?
        """, parameters: [ledgerId.uuidString])
    }
}
```

### 5.3 Category Filtering

Category filtering requires joining with `transaction_category_mappings`:

```swift
extension LedgerQueryBuilder {
    func buildQueryWithCategories(
        ledgerRules: LedgerRules,
        ledgerId: UUID
    ) -> SQLQuery {
        var baseQuery = buildQuery(ledgerRules: ledgerRules)
        
        // Add category filtering via JOIN
        if let includeCategories = ledgerRules.includeCategories, !includeCategories.isEmpty {
            baseQuery.sql = """
                SELECT DISTINCT t.* FROM transactions t
                INNER JOIN transaction_category_mappings m ON t.id = m.transactionId
                WHERE m.ledgerId = ? AND m.categoryId IN (\(placeholders))
                AND \(baseQuery.whereClause)
            """
            baseQuery.parameters.insert(ledgerId.uuidString, at: 0)
            baseQuery.parameters.insert(contentsOf: includeCategories.map { $0.uuidString }, at: 1)
        }
        
        // Similar for excludeCategories, includeTags, excludeTags
        
        return baseQuery
    }
}
```

---

## 6. Implementation Plan

### Phase 1: Foundation (Week 1)
- [ ] Create `SQLiteDatabase` wrapper/manager
- [ ] Design and create database schema (all tables)
- [ ] Implement schema versioning and migration system
- [ ] Create `SQLiteTransactionRepository` with basic CRUD
- [ ] Create `SQLiteAccountRepository` with basic CRUD
- [ ] Create `SQLiteLedgerRepository` with basic CRUD
- [ ] Add unit tests for repositories

### Phase 2: Query Implementation (Week 1-2)
- [ ] Implement `LedgerQueryBuilder` for rule-based queries
- [ ] Implement `findTransactions(ledgerRules:)` with query builder
- [ ] Implement `monthlyStatistics()` with SQL aggregation
- [ ] Implement `calculateBalance()` queries
- [ ] Add indexes for optimal query performance
- [ ] Add query performance tests

### Phase 3: Category Integration (Week 2)
- [ ] Implement category template tables and repository
- [ ] Implement transaction category mapping tables
- [ ] Extend query builder for category/tag filtering
- [ ] Update ledger queries to support category rules
- [ ] Test category-based ledger filtering

### Phase 4: Raw Transaction Migration (Week 2-3)
- [ ] Design unified raw transaction table schema
- [ ] Implement `SQLiteRawTransactionRepository`
- [ ] Migrate account-specific raw transaction repositories
- [ ] Update import batch tracking
- [ ] Test raw transaction queries

### Phase 5: Integration (Week 3)
- [ ] Update `LedgerService` to use repository queries
- [ ] Replace in-memory filtering with database queries
- [ ] Update `LedgerMonthlyStatisticsService` to use repository
- [ ] Update `LedgerBalanceService` to use repository
- [ ] Remove in-memory transaction array from `AppState`
- [ ] Update all transaction access to use repository

### Phase 6: Migration (Week 3-4)
- [ ] Create migration utility (JSON → SQLite)
- [ ] Add migration on first launch after update
- [ ] Implement dual-write phase (write to both JSON and SQLite)
- [ ] Verify data integrity after migration
- [ ] Add rollback mechanism if migration fails
- [ ] Test migration with large datasets

### Phase 7: Cleanup (Week 4)
- [ ] Remove JSON file persistence code
- [ ] Remove in-memory arrays (keep only for caching if needed)
- [ ] Update documentation
- [ ] Performance benchmarking and optimization
- [ ] Final testing and validation

---

## 7. Migration Strategy

### 7.1 Dual-Write Approach (Safest)

**Phase 1: Dual-Write (1-2 weeks)**
- Write to both JSON and SQLite
- Read from SQLite, fallback to JSON if needed
- Verify data consistency

**Phase 2: SQLite Primary (1 week)**
- Read from SQLite only
- Keep JSON as backup
- Monitor for issues

**Phase 3: Migration (1 week)**
- Migrate existing JSON data to SQLite
- Verify data integrity
- Keep JSON as read-only backup

**Phase 4: Cleanup (1 week)**
- Remove JSON writes
- Optionally remove JSON files (or keep as archive)

### 7.2 Migration Utility

```swift
actor DatabaseMigrationService {
    func migrateJSONToSQLite() async throws {
        // 1. Create database schema
        try await createSchema()
        
        // 2. Migrate accounts
        let accounts = try await loadAccountsFromJSON()
        try await accountRepository.save(accounts)
        
        // 3. Migrate transactions (in batches)
        let transactions = try await loadTransactionsFromJSON()
        try await transactionRepository.save(transactions)  // Batch insert
        
        // 4. Migrate ledgers
        let ledgers = try await loadLedgersFromJSON()
        try await ledgerRepository.save(ledgers)
        
        // 5. Migrate raw transactions (per account type)
        // ... migrate each account-specific raw data
        
        // 6. Migrate category data
        // ... migrate category templates and mappings
        
        // 7. Verify data integrity
        try await verifyMigration()
        
        // 8. Mark migration as complete
        UserDefaults.standard.set(true, forKey: "databaseMigrationComplete")
    }
    
    func verifyMigration() async throws -> Bool {
        // Compare counts
        let jsonTxCount = try await countTransactionsInJSON()
        let dbTxCount = try await transactionRepository.count()
        guard jsonTxCount == dbTxCount else {
            throw MigrationError.countMismatch
        }
        
        // Sample verification
        let sampleTransactions = try await loadSampleTransactionsFromJSON()
        for tx in sampleTransactions {
            guard let dbTx = try await transactionRepository.findById(tx.id) else {
                throw MigrationError.missingTransaction(tx.id)
            }
            guard tx == dbTx else {
                throw MigrationError.dataMismatch(tx.id)
            }
        }
        
        return true
    }
}
```

### 7.3 Rollback Plan

- Keep JSON files as backup during migration
- Migration flag in UserDefaults
- If SQLite fails, fallback to JSON repository
- User can manually trigger re-migration
- Database backup before migration

---

## 8. Performance Improvements

### 8.1 Expected Performance Gains

#### Before (JSON + In-Memory)
```
Load all transactions: ~500ms (10K transactions)
Filter by ledger rules: ~100ms (scan all)
Filter by month: ~50ms (scan filtered)
Calculate monthly stats: ~200ms (group all)
Total: ~850ms
```

#### After (SQLite)
```
Query transactions (ledger + month): ~15ms (indexed)
Calculate monthly stats: ~5ms (SQL aggregation)
Total: ~20ms
```

**Expected Improvement: 40-50x faster for ledger month queries**

#### Memory Usage

**Before**:
- All transactions in memory: ~10MB (10K transactions)
- All accounts in memory: ~100KB
- Total: ~10.1MB

**After**:
- No transactions in memory (lazy loading)
- Cached transactions (current view only): ~500KB
- Total: ~600KB

**Memory Reduction: ~94%**

### 8.2 Query Performance Examples

#### Example 1: Ledger Month View
```swift
// Before: Load all → filter in memory
let all = transactionsProvider()  // 10K transactions
let filtered = filterByLedger(all, rules)  // 5K transactions
let month = filterByMonth(filtered, year: 2024, month: 12)  // 200 transactions
// Time: ~750ms

// After: Direct SQL query
let month = try await repository.findTransactions(
    ledgerRules: rules,
    year: 2024,
    month: 12
)
// Time: ~15ms (50x faster)
```

#### Example 2: Monthly Statistics
```swift
// Before: Load all → group in memory
let all = transactionsProvider()  // 10K transactions
let filtered = filterByLedger(all, rules)  // 5K transactions
let stats = calculateMonthlyStats(filtered)  // Process all 5K
// Time: ~800ms

// After: SQL aggregation
let stats = try await repository.monthlyStatistics(
    ledgerRules: rules,
    year: 2024
)
// Time: ~10ms (80x faster)
```

---

## 9. Code Changes Summary

### Files to Create

**Core Database**:
- `MoneyFlow/Core/Data/Database/SQLiteDatabase.swift` - Database wrapper
- `MoneyFlow/Core/Data/Database/SchemaManager.swift` - Schema versioning
- `MoneyFlow/Core/Data/Database/MigrationService.swift` - Migration utility

**Repositories**:
- `MoneyFlow/Core/Data/Repositories/SQLiteTransactionRepository.swift`
- `MoneyFlow/Core/Data/Repositories/SQLiteAccountRepository.swift`
- `MoneyFlow/Ledgers/Data/Repositories/SQLiteLedgerRepository.swift`
- `MoneyFlow/CategoryTemplates/Data/Repositories/SQLiteCategoryTemplateRepository.swift`
- `MoneyFlow/DataSources/Shared/SQLiteRawTransactionRepository.swift`

**Query Builders**:
- `MoneyFlow/Ledgers/Domain/Services/LedgerQueryBuilder.swift`
- `MoneyFlow/Ledgers/Domain/Services/LedgerViewManager.swift` (optional)

### Files to Modify

**Core Services**:
- `MoneyFlow/Core/State/AppState.swift` - Remove in-memory arrays, use repositories
- `MoneyFlow/Core/Data/Dependencies/AppDependencies.swift` - Add SQLite repositories

**Ledger Services**:
- `MoneyFlow/Ledgers/Services/LedgerService.swift` - Use repository queries
- `MoneyFlow/Ledgers/Domain/Services/LedgerMonthlyStatisticsService.swift` - Use repository
- `MoneyFlow/Ledgers/Domain/Services/LedgerBalanceService.swift` - Use repository
- `MoneyFlow/Ledgers/Domain/Services/LedgerFilterService.swift` - Deprecate (use query builder)

**Repositories**:
- Update all repository protocols to add new query methods

### Files to Deprecate (Eventually)

- `MoneyFlow/Core/Data/Persistence/FilePersistence.swift` - Keep for migration period
- `MoneyFlow/Core/Data/Repositories/FileTransactionRepository.swift` - Keep for migration
- `MoneyFlow/Ledgers/Data/Repositories/FileLedgerRepository.swift` - Keep for migration

---

## 10. Risks and Mitigation

### Risk 1: Data Loss During Migration
**Mitigation**: 
- Dual-write during transition
- Keep JSON backup
- Verify data integrity after migration
- Rollback mechanism
- Database backups

### Risk 2: Performance Regression
**Mitigation**:
- Comprehensive performance tests
- Benchmark before/after
- Query optimization with indexes
- Monitor query execution times
- Profile slow queries

### Risk 3: Breaking Changes
**Mitigation**:
- Maintain repository protocol compatibility
- Gradual migration (dual-write phase)
- Feature flags for new vs old code paths
- Extensive testing
- Keep JSON fallback during transition

### Risk 4: SQLite Complexity
**Mitigation**:
- ✅ **SQLite.swift library**: Using established, type-safe SQLite wrapper (chosen)
- ✅ **Wrapper class**: `SQLiteDatabase` actor wrapper for all database operations
- ✅ **Well-documented SQL queries**: All queries documented with comments explaining purpose and performance considerations
- ✅ **Error handling and logging**: Comprehensive error types and logging throughout database layer
- ✅ **Comprehensive error messages**: Detailed error messages with context for debugging

### Risk 5: Category Filtering Performance
**Mitigation**:
- Proper indexes on category mapping tables
- Consider materialized views for complex category queries
- Cache frequently accessed category filters
- Profile and optimize JOIN queries

### Risk 6: Migration Time for Large Datasets
**Mitigation**:
- Batch inserts (1000 transactions per batch)
- Progress indicators for user
- Background migration
- Allow app usage during migration (read from JSON, write to SQLite)

---

## 11. Testing Strategy

### Unit Tests
- Repository CRUD operations
- Query builder SQL generation
- Query methods (ledger filtering, monthly stats, balance calculation)
- Migration utility
- Data integrity verification
- Schema versioning

### Integration Tests
- End-to-end query flow
- Ledger view computation
- Migration from JSON to SQLite
- Performance benchmarks
- Concurrent access (actor isolation)
- Category filtering with joins

### Performance Tests
- Query performance with 1K, 10K, 100K transactions
- Memory usage comparison
- App launch time
- Ledger view computation time
- Monthly statistics calculation time

### Manual Testing
- Large dataset (10K+ transactions)
- Migration on existing data
- Rollback scenarios
- Performance comparison
- Multiple ledgers with complex rules
- Category-based filtering

---

## 12. Success Metrics

### Performance
- Monthly ledger query: < 20ms (vs ~750ms currently)
- Monthly statistics: < 10ms (vs ~200ms currently)
- App launch: No regression (lazy loading)
- Memory usage: Reduced by 60-80% (no in-memory array)

### Reliability
- Zero data loss during migration
- 100% test coverage for repositories
- All existing features work identically
- Query accuracy matches in-memory filtering

### Developer Experience
- Cleaner code (no in-memory filtering)
- More flexible query capabilities
- Easier to add new query types
- Better separation of concerns

### User Experience
- Faster ledger view loading
- Smoother scrolling in transaction lists
- No noticeable delays when switching ledgers
- Better performance as dataset grows

---

## 13. Timeline Estimate

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Foundation | 4-5 days | SQLite repositories with basic CRUD |
| Query Implementation | 4-5 days | Ledger queries and aggregations |
| Category Integration | 3-4 days | Category filtering support |
| Raw Transaction Migration | 3-4 days | Unified raw transaction storage |
| Integration | 3-4 days | Update services to use repositories |
| Migration | 4-5 days | Migration utility and testing |
| Cleanup | 2-3 days | Remove old code, documentation |
| **Total** | **23-29 days** | **Full migration complete** |

**Note**: Timeline assumes single developer. Can be parallelized with multiple developers.

---

## 14. Dependencies

### External Libraries
- **SQLite.swift** (✅ **CHOSEN**): Type-safe SQLite wrapper
  - Package: `https://github.com/stephencelis/SQLite.swift`
  - Provides type-safe Swift API, compile-time query checking, and better error handling
  - Version: Latest stable (0.15.x or newer)
  - Integration: Add via Swift Package Manager

### System Requirements
- iOS 13+ (SQLite3 available)
- No additional frameworks needed
- SQLite is built into iOS

---

## 15. Future Enhancements

### 15.1 Full-Text Search
- FTS5 virtual tables for merchant/notes search
- Fast search across all transactions
- Search within ledger views

### 15.2 Advanced Aggregations
- Year-over-year comparisons
- Trend analysis
- Predictive analytics

### 15.3 Materialized Views
- Pre-computed views for frequently accessed ledgers
- Automatic refresh on transaction changes
- Further performance improvements

### 15.4 Database Replication
- Cloud sync (if needed in future)
- Multi-device support
- Backup and restore

---

## 16. Next Steps

1. ✅ **Review and Approve** this proposal - **APPROVED**
2. ✅ **Choose SQLite library** - **SQLite.swift CHOSEN**
3. **Create detailed technical design** for schema and queries - **IN PROGRESS**
4. **Set up development branch** for migration work
5. **Begin Phase 1** implementation
6. **Set up performance benchmarking** infrastructure
7. **Create migration test data** (large dataset)

---

## Appendix: Example SQL Queries

### Ledger Transaction Query
```sql
SELECT * FROM transactions
WHERE accountId IN (?, ?, ?)
  AND accountId NOT IN (?, ?)
  AND date >= ? AND date <= ?
  AND ABS(CAST(amount AS REAL)) >= ?
  AND ABS(CAST(amount AS REAL)) <= ?
  AND (
    (flow = 'flowIn' AND ? = 1) OR
    (flow = 'flowOut' AND ? = 1) OR
    (flow = 'neutral' AND ? = 1)
  )
  AND year = ? AND month = ?
ORDER BY date DESC, sequenceNumber ASC;
```

### Monthly Statistics Query
```sql
SELECT 
    year,
    month,
    SUM(CASE WHEN CAST(amount AS REAL) > 0 THEN CAST(amount AS REAL) ELSE 0 END) as income,
    SUM(CASE WHEN CAST(amount AS REAL) < 0 THEN ABS(CAST(amount AS REAL)) ELSE 0 END) as expenses,
    COUNT(*) as transactionCount
FROM transactions
WHERE accountId IN (?, ?, ?)
  AND date >= ? AND date <= ?
  AND (
    (flow = 'flowIn' AND ? = 1) OR
    (flow = 'flowOut' AND ? = 1)
  )
GROUP BY year, month
ORDER BY year DESC, month DESC;
```

### Balance Calculation Query
```sql
SELECT 
    accountId,
    SUM(CAST(amount AS REAL)) as balanceChange
FROM transactions
WHERE accountId IN (?, ?, ?)
  AND date <= ?
GROUP BY accountId;
```

### Category-Filtered Query
```sql
SELECT DISTINCT t.*
FROM transactions t
INNER JOIN transaction_category_mappings m ON t.id = m.transactionId
WHERE m.ledgerId = ?
  AND m.categoryId IN (?, ?, ?)
  AND t.accountId IN (?, ?, ?)
  AND t.date >= ? AND t.date <= ?
ORDER BY t.date DESC, t.sequenceNumber ASC;
```

---

**Document Version**: 1.0  
**Date**: 2025-01-XX  
**Author**: Architecture Team  
**Status**: Proposal - Pending Approval

