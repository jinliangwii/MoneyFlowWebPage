# SQLite Migration Proposal: Database Query for Monthly Grouped Transactions

## Executive Summary

This proposal outlines migrating transaction storage from JSON files to SQLite database to enable efficient database queries for monthly grouped transactions. The migration will improve performance, scalability, and enable complex query capabilities while maintaining backward compatibility.

---

## 1. Current State Analysis

### Current Architecture
- **Storage**: JSON files (`transactions.json`)
- **Loading**: All transactions loaded into memory on app launch
- **Querying**: In-memory filtering after loading all data
- **Monthly Grouping**: `LedgerMonthlyStatisticsService` groups all transactions in memory
- **Performance**: O(n) for all queries, regardless of data needed

### Performance Issues
- **Inefficient**: Loading 10K+ transactions when only one month is needed
- **Memory**: All transactions kept in memory (`AppState.transactions`)
- **Query Pattern**: `findByAccountAndMonth()` loads all → filters in memory
- **Monthly Stats**: Must process all transactions even for single month view

### Current Query Pattern
```swift
// Current: Loads ALL transactions, then filters
func findByAccountAndMonth(_ accountId: UUID, year: Int, month: Int) async throws -> [Transaction] {
    let all = try await findAll()  // Loads entire JSON file
    return all.filter { /* month/year check */ }  // In-memory filter
}
```

---

## 2. Proposed Solution: SQLite Database

### Overview
Migrate from JSON file storage to SQLite database with:
- **Indexed queries** for fast date/account lookups
- **SQL aggregation** (`GROUP BY`, `SUM`) for monthly statistics
- **Query-level filtering** instead of in-memory filtering
- **Repository pattern** maintained (swap implementation)

### Key Benefits
1. **Performance**: Indexed queries are O(log n) vs O(n) in-memory
2. **Scalability**: Handles large datasets (100K+ transactions) efficiently
3. **Query Flexibility**: SQL supports complex aggregations and filters
4. **Memory Efficiency**: Only load data needed for current view
5. **Standard Technology**: SQLite is battle-tested and reliable

---

## 3. Database Schema Design

### Transactions Table
```sql
CREATE TABLE transactions (
    id TEXT PRIMARY KEY,
    accountId TEXT NOT NULL,
    date TEXT NOT NULL,  -- ISO8601 format
    amount TEXT NOT NULL,  -- Decimal as string for precision
    merchant TEXT,
    notes TEXT,
    balance TEXT,
    sequenceNumber INTEGER,
    
    -- Computed columns for fast queries
    year INTEGER GENERATED ALWAYS AS (CAST(strftime('%Y', date) AS INTEGER)) STORED,
    month INTEGER GENERATED ALWAYS AS (CAST(strftime('%m', date) AS INTEGER)) STORED
);

-- Indexes for common query patterns
CREATE INDEX idx_transactions_date ON transactions(date);
CREATE INDEX idx_transactions_account ON transactions(accountId);
CREATE INDEX idx_transactions_year_month ON transactions(year, month);
CREATE INDEX idx_transactions_account_date ON transactions(accountId, date);
CREATE INDEX idx_transactions_account_year_month ON transactions(accountId, year, month);
```

### Schema Migration Support
- Version tracking for future schema changes
- Migration scripts for schema evolution
- Backward compatibility during transition

---

## 4. Repository Interface Enhancement

### Enhanced Protocol
```swift
protocol TransactionRepositoryProtocol {
    // Existing methods (maintained for compatibility)
    func findAll() async throws -> [Transaction]
    func findByAccount(_ accountId: UUID) async throws -> [Transaction]
    func findByDateRange(_ start: Date, _ end: Date) async throws -> [Transaction]
    func findByAccountAndMonth(_ accountId: UUID, year: Int, month: Int) async throws -> [Transaction]
    func findById(_ id: UUID) async throws -> Transaction?
    func save(_ transactions: [Transaction]) async throws
    func delete(_ id: UUID) async throws
    func deleteByAccount(_ accountId: UUID) async throws
    
    // NEW: Direct monthly statistics query
    func monthlyStatistics(
        accountIds: [UUID]?,
        excludeAccountIds: [UUID]?,
        year: Int?,
        startDate: Date?,
        endDate: Date?
    ) async throws -> [LedgerMonthlyStatistics]
    
    // NEW: Query with ledger rules
    func findTransactions(
        accountIds: [UUID]?,
        excludeAccountIds: [UUID]?,
        startDate: Date?,
        endDate: Date?,
        minAmount: Decimal?,
        maxAmount: Decimal?,
        includeIncome: Bool,
        includeExpense: Bool
    ) async throws -> [Transaction]
}
```

### SQLite Implementation
```swift
actor SQLiteTransactionRepository: TransactionRepositoryProtocol {
    private let db: SQLiteDatabase
    
    // Monthly statistics query (SQL aggregation)
    func monthlyStatistics(
        accountIds: [UUID]?,
        excludeAccountIds: [UUID]?,
        year: Int?,
        startDate: Date?,
        endDate: Date?
    ) async throws -> [LedgerMonthlyStatistics] {
        // SQL: SELECT year, month, SUM(income), SUM(expenses)
        //      FROM transactions
        //      WHERE accountId IN (...) AND date BETWEEN ... AND ...
        //      GROUP BY year, month
        //      ORDER BY year DESC, month DESC
    }
    
    // Efficient transaction query
    func findByAccountAndMonth(_ accountId: UUID, year: Int, month: Int) async throws -> [Transaction] {
        // SQL: SELECT * FROM transactions
        //      WHERE accountId = ? AND year = ? AND month = ?
        //      ORDER BY date DESC
        // Uses index: idx_transactions_account_year_month
    }
}
```

---

## 5. Implementation Plan

### Phase 1: Foundation (Week 1)
- [ ] Create `SQLiteDatabase` wrapper/manager
- [ ] Design and create database schema
- [ ] Implement `SQLiteTransactionRepository` with basic CRUD
- [ ] Add unit tests for repository

### Phase 2: Query Implementation (Week 1-2)
- [ ] Implement `monthlyStatistics()` query with SQL aggregation
- [ ] Implement `findTransactions()` with ledger rules
- [ ] Optimize queries with proper indexes
- [ ] Add query performance tests

### Phase 3: Integration (Week 2)
- [ ] Update `LedgerService` to use repository queries
- [ ] Replace in-memory filtering with database queries
- [ ] Update `LedgerMonthlyStatisticsService` to use repository
- [ ] Maintain backward compatibility with existing code

### Phase 4: Migration (Week 2-3)
- [ ] Create migration utility (JSON → SQLite)
- [ ] Add migration on first launch after update
- [ ] Verify data integrity after migration
- [ ] Add rollback mechanism if migration fails

### Phase 5: Cleanup (Week 3)
- [ ] Remove in-memory transaction array from `AppState`
- [ ] Update all transaction access to use repository
- [ ] Remove JSON file persistence
- [ ] Update documentation

---

## 6. Migration Strategy

### Dual-Write Approach (Safest)
1. **Phase 1**: Write to both JSON and SQLite
2. **Phase 2**: Read from SQLite, fallback to JSON if needed
3. **Phase 3**: Migrate existing JSON data to SQLite
4. **Phase 4**: Remove JSON writes, keep as backup
5. **Phase 5**: Remove JSON entirely

### Migration Utility
```swift
actor TransactionMigrationService {
    func migrateJSONToSQLite(
        jsonPath: URL,
        sqlitePath: URL
    ) async throws {
        // 1. Load transactions from JSON
        // 2. Create SQLite database
        // 3. Insert transactions in batches
        // 4. Verify data integrity
        // 5. Mark migration as complete
    }
    
    func verifyMigration() async throws -> Bool {
        // Compare counts, sample records
    }
}
```

### Rollback Plan
- Keep JSON file as backup during migration
- Migration flag in UserDefaults
- If SQLite fails, fallback to JSON repository
- User can manually trigger re-migration

---

## 7. Performance Improvements

### Before (JSON + In-Memory)
```
Load all transactions: ~500ms (10K transactions)
Filter by month: ~50ms (scan all)
Calculate monthly stats: ~200ms (group all)
Total: ~750ms
```

### After (SQLite)
```
Query transactions for month: ~10ms (indexed)
Calculate monthly stats: ~5ms (SQL aggregation)
Total: ~15ms
```

**Expected Improvement: 50x faster for monthly queries**

---

## 8. Code Changes Summary

### Files to Create
- `MoneyFlow/Core/Data/Database/SQLiteDatabase.swift`
- `MoneyFlow/Core/Data/Repositories/SQLiteTransactionRepository.swift`
- `MoneyFlow/Core/Data/Migration/TransactionMigrationService.swift`

### Files to Modify
- `MoneyFlow/Ledgers/Services/LedgerService.swift` - Use repository queries
- `MoneyFlow/Ledgers/Domain/Services/LedgerMonthlyStatisticsService.swift` - Use repository
- `MoneyFlow/Core/State/AppState.swift` - Remove in-memory array, use repository
- `MoneyFlow/Core/Data/Repositories/TransactionRepositoryProtocol.swift` - Add new methods

### Files to Deprecate (Eventually)
- `MoneyFlow/Core/Data/Repositories/FileTransactionRepository.swift` - Keep for migration period

---

## 9. Risks and Mitigation

### Risk 1: Data Loss During Migration
**Mitigation**: 
- Dual-write during transition
- Keep JSON backup
- Verify data integrity after migration
- Rollback mechanism

### Risk 2: Performance Regression
**Mitigation**:
- Comprehensive performance tests
- Benchmark before/after
- Query optimization with indexes
- Monitor query execution times

### Risk 3: Breaking Changes
**Mitigation**:
- Maintain repository protocol compatibility
- Gradual migration (dual-write phase)
- Feature flags for new vs old code paths
- Extensive testing

### Risk 4: SQLite Complexity
**Mitigation**:
- Well-documented SQL queries
- Wrapper class for database operations
- Error handling and logging
- Use established SQLite Swift libraries (e.g., SQLite.swift)

---

## 10. Testing Strategy

### Unit Tests
- Repository CRUD operations
- Query methods (monthly stats, filtering)
- Migration utility
- Data integrity verification

### Integration Tests
- End-to-end query flow
- Migration from JSON to SQLite
- Performance benchmarks
- Concurrent access (actor isolation)

### Manual Testing
- Large dataset (10K+ transactions)
- Migration on existing data
- Rollback scenarios
- Performance comparison

---

## 11. Success Metrics

### Performance
- Monthly query: < 20ms (vs ~750ms currently)
- App launch: No regression (lazy loading)
- Memory usage: Reduced by 50%+ (no in-memory array)

### Reliability
- Zero data loss during migration
- 100% test coverage for repository
- All existing features work identically

### Developer Experience
- Cleaner code (no in-memory filtering)
- More flexible query capabilities
- Easier to add new query types

---

## 12. Timeline Estimate

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Foundation | 3-4 days | SQLite repository with basic CRUD |
| Query Implementation | 3-4 days | Monthly stats and filtering queries |
| Integration | 2-3 days | Update LedgerService and related code |
| Migration | 2-3 days | Migration utility and testing |
| Cleanup | 1-2 days | Remove old code, documentation |
| **Total** | **11-16 days** | **Full migration complete** |

---

## 13. Dependencies

### External Libraries (Optional)
- **SQLite.swift** (recommended): Type-safe SQLite wrapper
  - Alternative: Use Foundation's SQLite3 C API directly

### System Requirements
- iOS 13+ (SQLite3 available)
- No additional frameworks needed

---

## 14. Next Steps

1. **Review and Approve** this proposal
2. **Choose SQLite library** (SQLite.swift vs Foundation API)
3. **Create detailed technical design** for schema and queries
4. **Set up development branch** for migration work
5. **Begin Phase 1** implementation

---

## Appendix: Example SQL Queries

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
  AND (year = ? OR ? IS NULL)
  AND date >= ? AND date <= ?
GROUP BY year, month
ORDER BY year DESC, month DESC;
```

### Transaction Query with Ledger Rules
```sql
SELECT *
FROM transactions
WHERE accountId IN (?, ?, ?)
  AND accountId NOT IN (?, ?)
  AND date >= ? AND date <= ?
  AND ABS(CAST(amount AS REAL)) >= ?
  AND ABS(CAST(amount AS REAL)) <= ?
  AND (
    (CAST(amount AS REAL) > 0 AND ? = 1) OR  -- includeIncome
    (CAST(amount AS REAL) < 0 AND ? = 1)      -- includeExpense
  )
ORDER BY date DESC, sequenceNumber ASC;
```

---

**Document Version**: 1.0  
**Date**: 2025-01-XX  
**Author**: Architecture Team  
**Status**: Proposal - Pending Approval

