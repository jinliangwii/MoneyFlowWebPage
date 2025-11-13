# Import Service Size Analysis

**Date:** 2025-01-XX  
**Status:** Analysis Complete

---

## Executive Summary

Despite implementing `BaseImportService` with the Template Method pattern, import services remain large (198-777 lines). This document explains why and proposes solutions.

---

## Current Service Sizes

| Service | Lines | Status |
|---------|-------|--------|
| CMBCreditCardImportService | 777 | 🔴 Very Large |
| CMBPersonalCheckingImportService | 411 | 🟡 Large |
| CMBExpressLoanImportService | 259 | 🟢 Moderate |
| Alipay3rdPartyTxImportService | 219 | 🟢 Moderate |
| WeChatImportService | 198 | 🟢 Moderate |

**Total:** ~1,864 lines across 5 services

---

## Why Services Are Still Large

### 1. **Duplicate Import Flows** 🔴 (CMBCreditCard: ~220 lines)

**Problem:** `CMBCreditCardImportService` has TWO complete import implementations:

- `executeImport()` override (lines 311-531, ~220 lines)
- `importFromDataSource()` (lines 81-305, ~224 lines)

Both implement nearly identical logic:
- Parse/extract transactions
- Verify metadata
- Filter transactions
- Load existing data
- Build duplicate structures
- Build sequence counters
- Process transactions (duplicate detection, sequence assignment, conversion)
- Save data
- Build result

**Root Cause:** Credit cards need custom balance calculation that requires sequence numbers BEFORE calculating balances (work backwards from statement balance). This breaks the normal flow where balances are calculated first, then sequence numbers assigned.

**Impact:** ~220 lines of duplicated logic

---

### 2. **Large Duplicated Validation Methods** 🔴 (~100 lines each)

**Problem:** `validateOverlappingData()` is nearly identical across CMB services:

- `CMBCreditCardImportService`: lines 620-716 (~96 lines)
- `CMBPersonalCheckingImportService`: lines 201-307 (~106 lines)

Both implement the same logic:
- Find date ranges
- Check for overlap
- Extract overlapping transactions
- Compare transaction counts
- Compare transaction amounts by day
- Generate detailed error messages

**Root Cause:** This validation is CMB-specific (PDFs with overlapping date ranges must match exactly), but it's implemented in each service rather than shared.

**Impact:** ~200 lines of duplicated validation logic

---

### 3. **Complex Balance Calculation Embedded in Service** 🟡 (CMBCreditCard: ~150 lines)

**Problem:** Credit card balance calculation is embedded directly in `executeImport()`:

- Lines 392-488: Process transactions, assign sequence numbers, collect metadata
- Lines 448-470: Calculate balances working backwards from statement balance
- Lines 472-488: Convert to transactions with calculated balances

**Root Cause:** Credit cards need sequence numbers BEFORE calculating balances (unlike other accounts), so the normal `calculateBalances()` hook can't be used.

**Impact:** ~150 lines of complex logic in the service

---

### 4. **Account-Specific Helpers Not Extracted** 🟡 (~30-50 lines each)

**Problem:** Each service has private helper methods that could be extracted:

- `CMBCreditCardImportService`:
  - `calculateTransactionAmountForBalance()` (lines 757-768)
  - `normalizeCurrency()` (lines 770-772)
  - `currencyDisplayName()` (lines 774-776)

- `CMBPersonalCheckingImportService`:
  - `validateBalanceConsistency()` (lines 369-402, ~33 lines)
  - `normalizeCurrency()` (lines 404-406)
  - `currencyDisplayName()` (lines 408-410)

**Root Cause:** These are account-specific utilities that could be moved to separate utility classes.

**Impact:** ~30-50 lines per service

---

### 5. **Currency Filtering Logic in Service** 🟡 (~20-30 lines each)

**Problem:** Currency filtering logic is embedded in `filterTransactions()`:

- `CMBCreditCardImportService`: lines 580-617
- `CMBPersonalCheckingImportService`: lines 168-199

Both implement similar currency normalization and filtering.

**Root Cause:** Currency handling is account-specific but follows similar patterns.

**Impact:** ~20-30 lines per service

---

## Detailed Breakdown

### CMBCreditCardImportService (777 lines)

```
Lines 1-76:     Configuration, initialization, factory methods
Lines 78-305:   importFromDataSource() - Complete import flow (~227 lines)
Lines 307-531:  executeImport() override - Complete import flow (~224 lines)
Lines 533-559:  parseSource() - PDF parsing (~26 lines)
Lines 561-578:  verifyAndSaveMetadata() - Account number verification (~17 lines)
Lines 580-618:  filterTransactions() - Currency filtering (~38 lines)
Lines 620-716:  validateOverlappingData() - Date range validation (~96 lines)
Lines 718-725:  calculateBalances() - Not used (credit card specific)
Lines 727-737:  convertRawToTransaction() - Conversion (~10 lines)
Lines 739-753:  createImportBatch() - Batch creation (~14 lines)
Lines 755-777:  Private helpers (~22 lines)
```

**Key Issues:**
- Two complete import flows (lines 78-305 and 307-531) = ~450 lines
- Large validation method (lines 620-716) = ~96 lines
- Complex balance calculation embedded in import flow = ~150 lines

**Total Account-Specific Logic:** ~696 lines  
**Total Duplicated/Extractable:** ~470 lines

---

### CMBPersonalCheckingImportService (411 lines)

```
Lines 1-78:     Configuration, initialization, factory methods
Lines 80-112:   importFromDataSource() - Delegates to base (~32 lines)
Lines 114-147:  parseSource() - ZIP extraction + PDF parsing (~33 lines)
Lines 149-166:  verifyAndSaveMetadata() - Account number verification (~17 lines)
Lines 168-199:  filterTransactions() - Currency filtering + balance validation (~31 lines)
Lines 201-307:  validateOverlappingData() - Date range validation (~106 lines)
Lines 309-333:  calculateBalances() - Balance map from PDF (~24 lines)
Lines 335-349:  convertRawToTransaction() - Conversion (~14 lines)
Lines 351-365:  createImportBatch() - Batch creation (~14 lines)
Lines 367-411:  Private helpers (~44 lines)
```

**Key Issues:**
- Large validation method (lines 201-307) = ~106 lines
- Balance validation helper (lines 369-402) = ~33 lines
- Currency filtering logic = ~31 lines

**Total Account-Specific Logic:** ~411 lines  
**Total Duplicated/Extractable:** ~170 lines

---

## What BaseImportService Already Handles ✅

The `BaseImportService` (561 lines) already extracts:

1. ✅ **Common Import Flow** (~200 lines)
   - Loading existing data
   - Building duplicate detection structures
   - Building sequence number counters
   - Processing transactions (duplicate detection, sequence assignment)
   - Saving data
   - Building results

2. ✅ **Data Source Support** (~150 lines)
   - `executeImportFromDataSource()` method
   - Type conversion from protocol types
   - Shared continuation flow

3. ✅ **Helper Methods** (via `BaseImportServiceHelper`)
   - Duplicate detection
   - Sequence number management
   - Existing data loading

**Estimated Code Saved:** ~400 lines per service × 5 services = ~2,000 lines

---

## Proposed Solutions

### Solution 1: Extract CMB Overlap Validation 🔴 High Priority

**Create:** `CMBOverlapValidator` utility class

```swift
class CMBOverlapValidator {
    static func validate(
        filteredTransactions: [any RawTransactionProtocol],
        existingTransactions: [Transaction],
        getDate: (any RawTransactionProtocol) -> Date?,
        getAmount: (any RawTransactionProtocol) -> Decimal?
    ) throws {
        // Shared validation logic
    }
}
```

**Impact:** 
- Remove ~100 lines from each CMB service
- Total savings: ~300 lines

---

### Solution 2: Extract Credit Card Balance Calculator 🔴 High Priority

**Create:** `CreditCardBalanceCalculator` utility class

```swift
class CreditCardBalanceCalculator {
    static func calculateBalances(
        transactions: [RawCMBCreditCardTransaction],
        statementBalance: Decimal,
        sequenceCounters: inout SequenceCounters
    ) -> [String: Decimal] {
        // Complex balance calculation logic
    }
}
```

**Impact:**
- Remove ~150 lines from `CMBCreditCardImportService`
- Make balance calculation reusable and testable

---

### Solution 3: Unify Import Flows 🟡 Medium Priority

**Problem:** Credit card has two import flows because balance calculation needs sequence numbers first.

**Solution:** Create a specialized balance calculation hook that runs after sequence assignment:

```swift
// In BaseImportService
func calculateBalancesAfterSequence(
    transactionsWithSequence: [(raw: Config.RawTransactionType, sequence: Int, date: Date)],
    accountId: UUID
) async throws -> [String: Decimal] {
    // Default: use normal calculateBalances
    return [:]
}
```

**Impact:**
- Remove duplicate `importFromDataSource()` flow from credit card service
- Reduce service size by ~220 lines

---

### Solution 4: Extract Currency Utilities 🟢 Low Priority

**Create:** Shared currency utilities or protocol-based currency handlers

**Impact:**
- Remove ~20-30 lines per service
- Total savings: ~100 lines

---

### Solution 5: Extract Balance Validation 🟢 Low Priority

**Create:** `BalanceConsistencyValidator` utility

**Impact:**
- Remove ~33 lines from `CMBPersonalCheckingImportService`

---

## Expected Results After Refactoring

| Service | Current | After | Reduction |
|---------|---------|-------|-----------|
| CMBCreditCardImportService | 777 | ~350 | **-55%** |
| CMBPersonalCheckingImportService | 411 | ~240 | **-42%** |
| CMBExpressLoanImportService | 259 | ~200 | **-23%** |
| Alipay3rdPartyTxImportService | 219 | ~180 | **-18%** |
| WeChatImportService | 198 | ~170 | **-14%** |

**Total:** ~1,864 → ~1,140 lines (**-39% reduction**)

---

## Implementation Priority

1. **🔴 High Priority:**
   - Extract CMB overlap validation (~300 lines saved)
   - Extract credit card balance calculator (~150 lines saved)

2. **🟡 Medium Priority:**
   - Unify credit card import flows (~220 lines saved)

3. **🟢 Low Priority:**
   - Extract currency utilities (~100 lines saved)
   - Extract balance validation (~33 lines saved)

**Total Potential Savings:** ~803 lines (**43% reduction**)

---

## Conclusion

The `BaseImportService` successfully extracted ~2,000 lines of common logic, but services remain large due to:

1. **Duplicate flows** in credit card service (~220 lines)
2. **Duplicated validation** across CMB services (~300 lines)
3. **Complex embedded logic** that could be extracted (~280 lines)

**Recommended Next Steps:**
1. Extract CMB overlap validation (quick win, ~300 lines)
2. Extract credit card balance calculator (~150 lines)
3. Refactor credit card to use unified flow (~220 lines)

This would reduce total service code by **~670 lines (36%)**, making services more maintainable and testable.

