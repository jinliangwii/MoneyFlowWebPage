# UPSERT Refactoring Summary

## Overview

Refactored all database repositories to use `INSERT ... ON CONFLICT DO UPDATE` (UPSERT) instead of `INSERT OR REPLACE`. This is a better approach for several critical reasons.

## Why UPSERT is Better Than INSERT OR REPLACE

### Problems with INSERT OR REPLACE

1. **Triggers DELETE First**: `INSERT OR REPLACE` deletes the existing row first, then inserts a new one
2. **CASCADE DELETE Issues**: The DELETE operation triggers `ON DELETE CASCADE`, which can unintentionally delete child rows
3. **Rowid Changes**: If using INTEGER PRIMARY KEY, the rowid changes, breaking references
4. **Performance**: DELETE + INSERT is less efficient than an in-place UPDATE

### Benefits of UPSERT (INSERT ... ON CONFLICT DO UPDATE)

1. **In-Place Updates**: Updates the existing row without deleting it
2. **No CASCADE DELETE**: Doesn't trigger DELETE operations, so CASCADE DELETE doesn't fire
3. **Preserves Rowid**: Keeps the same rowid, maintaining referential integrity
4. **Better Performance**: Single atomic operation that's more efficient
5. **Preserves Foreign Keys**: Maintains all foreign key relationships

## Files Updated

### 1. SQLiteTransactionRepository.swift
- **Changed**: `INSERT OR REPLACE` → `INSERT ... ON CONFLICT(id) DO UPDATE`
- **Impact**: Prevents accidental deletion of related records (e.g., category mappings)
- **Note**: Transactions have foreign key to accounts with `ON DELETE CASCADE`, but transactions are children, so this mainly prevents issues with child tables of transactions

### 2. SQLiteRawTransactionRepository.swift
- **Changed**: `INSERT OR REPLACE` → `INSERT ... ON CONFLICT(id) DO UPDATE`
- **Impact**: Prevents issues when updating raw transaction data
- **Note**: Raw transactions have foreign key to accounts with `ON DELETE CASCADE`

### 3. SQLiteImportBatchRepository.swift
- **Changed**: `INSERT OR REPLACE` → `INSERT ... ON CONFLICT(id) DO UPDATE`
- **Impact**: Prevents issues when updating import batch metadata
- **Note**: Import batches have foreign key to accounts with `ON DELETE CASCADE`

### 4. SQLiteCategoryTemplateRepository.swift
- **Changed**: `INSERT OR REPLACE` → `INSERT ... ON CONFLICT(id) DO UPDATE`
- **Impact**: **CRITICAL** - Prevents CASCADE DELETE of child categories and rules when updating a template
- **Note**: Category templates have children (categories) with `ON DELETE CASCADE`, and categories have children (rules) with `ON DELETE CASCADE`. Using `INSERT OR REPLACE` would delete all categories and rules when updating a template!

### 5. SQLiteTransactionCategoryMappingRepository.swift
- **Changed**: `DELETE + INSERT` → `DELETE (for uniqueness) + INSERT ... ON CONFLICT(id) DO UPDATE`
- **Impact**: More efficient updates while maintaining uniqueness constraint
- **Note**: Still uses DELETE to enforce application-level uniqueness on `(transactionId, ledgerId)` since there's no unique constraint in the schema. The UPSERT handles updates to existing mappings by `id`.

## Files Already Using Correct Pattern

### SQLiteAccountRepository.swift
- **Status**: ✅ Already uses `UPDATE + INSERT` pattern
- **Reason**: Explicitly avoids `INSERT OR REPLACE` to prevent CASCADE DELETE of transactions
- **Comment**: "INSERT OR REPLACE would delete the old row first, triggering ON DELETE CASCADE and deleting all transactions for that account."

## SQLite Version Compatibility

UPSERT syntax (`INSERT ... ON CONFLICT DO UPDATE`) was introduced in SQLite 3.24.0 (2018). Modern iOS/macOS systems include SQLite 3.24.0+, so this is safe to use.

## Example: Before vs After

### Before (INSERT OR REPLACE)
```sql
INSERT OR REPLACE INTO transactions (
    id, accountId, date, amount, merchant, notes,
    balance, sequenceNumber, flow
) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
```

**What happens:**
1. DELETE existing row (if exists)
2. Triggers ON DELETE CASCADE (if any)
3. INSERT new row
4. New rowid (if using INTEGER PRIMARY KEY)

### After (UPSERT)
```sql
INSERT INTO transactions (
    id, accountId, date, amount, merchant, notes,
    balance, sequenceNumber, flow
) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
ON CONFLICT(id) DO UPDATE SET
    accountId = excluded.accountId,
    date = excluded.date,
    amount = excluded.amount,
    merchant = excluded.merchant,
    notes = excluded.notes,
    balance = excluded.balance,
    sequenceNumber = excluded.sequenceNumber,
    flow = excluded.flow
```

**What happens:**
1. Try to INSERT
2. If conflict on `id`, UPDATE existing row in-place
3. No DELETE operation
4. No CASCADE DELETE
5. Same rowid preserved

## Testing Recommendations

1. **Test Template Updates**: Verify that updating a category template doesn't delete its categories and rules
2. **Test Transaction Updates**: Verify that updating transactions doesn't affect related records
3. **Test Foreign Key Integrity**: Verify that all foreign key relationships are maintained
4. **Test Performance**: UPSERT should be slightly faster than INSERT OR REPLACE

## Migration Notes

- **No Schema Changes Required**: This is a code-only change
- **Backward Compatible**: Existing data works with both approaches
- **No Migration Script Needed**: The database schema remains unchanged

## Future Improvements

Consider adding a unique constraint to `transaction_category_mappings` on `(transactionId, ledgerId)` to enforce uniqueness at the database level, which would allow removing the DELETE operation before INSERT in `SQLiteTransactionCategoryMappingRepository`.

---

**Date**: 2024
**Status**: ✅ Complete
**Impact**: Improved data integrity and performance

