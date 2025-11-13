# New Account Architecture - Implementation Roadmap

**Date:** 2025-01-XX  
**Current Status:** Foundation Complete ✅  
**Next Milestone:** Integration Phase

---

## 🗺️ Roadmap Overview

```
Foundation (✅ Complete)
    ↓
Integration (⏳ Next)
    ↓
UI Updates (⏳ Pending)
    ↓
Testing (⏳ Pending)
    ↓
Production (🎯 Goal)
```

---

## ✅ Completed (Phases 1-6)

- [x] Account Traits Protocol
- [x] Data Source Protocol
- [x] All 6 Data Source Implementations
- [x] AccountCreator
- [x] DataSourceFactory
- [x] Integration Helpers
- [x] Documentation

---

## ⏳ Next: Integration Phase (Phases 7-10)

### Phase 7: Import Service Integration (20-30 hours)

**Priority: High**

1. **Update BaseImportService** (4-6 hours)
   - Add data source support method
   - Bridge old and new architectures
   - Maintain backward compatibility

2. **Update Import Services** (12-18 hours)
   - CMBCreditCardImportService
   - CMBPersonalCheckingImportService
   - CMBExpressLoanImportService
   - Alipay3rdPartyTxImportService
   - WeChatImportService
   - PlaidTransactionSyncService

3. **Update AppState** (4-6 hours)
   - Support data source in importAccountTransactions
   - Handle account extraction
   - Support account selection

**Deliverable:** Import services can use new data sources

---

### Phase 8: UI Updates (32-42 hours)

**Priority: High**

1. **Account Selection Component** (6-8 hours)
   - Reusable AccountSelectionView
   - Multi-select support
   - Account metadata display

2. **Update Import Views** (12-16 hours)
   - All account-specific import views
   - Add account extraction step
   - Use AccountCreator

3. **Update AddAccountView** (8-10 hours)
   - Support file-based account extraction
   - Account selection UI
   - Integration with AccountCreator

4. **Update Plaid UI** (6-8 hours)
   - Use PlaidAPIDataSource
   - Enhance account selection
   - Use AccountCreator

**Deliverable:** UI supports new architecture

---

### Phase 9: Testing (18-24 hours)

**Priority: High**

1. **Integration Tests** (8-10 hours)
   - Data source tests
   - AccountCreator tests
   - End-to-end import tests

2. **End-to-End Tests** (6-8 hours)
   - Full import flow
   - Multi-account scenarios
   - Error handling

3. **UI Tests** (4-6 hours)
   - Account selection
   - Import flow
   - Error states

**Deliverable:** Comprehensive test coverage

---

### Phase 10: Cleanup (10-16 hours)

**Priority: Medium**

1. **Remove Old Code** (4-6 hours)
   - Old parseSource implementations
   - Duplicate logic
   - Unused imports

2. **Optimization** (4-6 hours)
   - Performance profiling
   - Caching strategies
   - Memory optimization

3. **Documentation** (2-4 hours)
   - Update Architecture.md
   - Update feature docs
   - Update READMEs

**Deliverable:** Clean, optimized codebase

---

## 🎯 Milestones

### Milestone 1: Foundation ✅
**Status:** Complete  
**Date:** 2025-01-XX

### Milestone 2: First Integration
**Status:** Not Started  
**Target:** Complete Phase 7, Task 7.1-7.2  
**Success Criteria:** One import service using new architecture

### Milestone 3: UI Integration
**Status:** Not Started  
**Target:** Complete Phase 8  
**Success Criteria:** UI supports account extraction and selection

### Milestone 4: Production Ready
**Status:** Not Started  
**Target:** Complete Phases 9-10  
**Success Criteria:** All tests pass, old code removed

---

## 📊 Progress Tracking

| Phase | Status | Progress | Estimated Remaining |
|-------|--------|----------|---------------------|
| Phase 1-6 | ✅ Complete | 100% | 0 hours |
| Phase 7 | ⏳ Not Started | 0% | 20-30 hours |
| Phase 8 | ⏳ Not Started | 0% | 32-42 hours |
| Phase 9 | ⏳ Not Started | 0% | 18-24 hours |
| Phase 10 | ⏳ Not Started | 0% | 10-16 hours |
| **Total** | | **~33%** | **80-112 hours** |

---

## 🚀 Quick Start Guide

### To Start Integration:

1. **Pick a Simple Service**
   - Recommendation: WeChat or Alipay (simpler than PDF)
   - Less complex, easier to test

2. **Update BaseImportService**
   - Add `importFromDataSource()` method
   - Bridge to existing flow

3. **Update One Import Service**
   - Add data source support
   - Test thoroughly

4. **Update UI**
   - Add account extraction step
   - Test with real files

5. **Iterate**
   - Move to next service
   - Refine approach
   - Document learnings

---

## 📝 Next Immediate Actions

1. **Update BaseImportService** to support data sources
2. **Create AccountSelectionView** component
3. **Update one import service** (WeChat recommended)
4. **Write integration tests** for the updated service
5. **Test with real data** files

---

## 🔗 Key Files to Modify

### Import Services
- `MoneyFlow/Accounts/Shared/BaseImportService.swift`
- `MoneyFlow/Accounts/CMBCreditCard/Services/CMBCreditCardImportService.swift`
- `MoneyFlow/Accounts/CMBPersonalChecking/Services/CMBPersonalCheckingImportService.swift`
- `MoneyFlow/Accounts/CMBExpressLoan/Services/CMBExpressLoanImportService.swift`
- `MoneyFlow/Accounts/Alipay3rdPartyTx/Services/Alipay3rdPartyTxImportService.swift`
- `MoneyFlow/Accounts/WeChat/Services/WeChatImportService.swift`

### UI Views
- `MoneyFlow/UI/Properties/AddAccountView.swift`
- `MoneyFlow/Accounts/CMBCreditCard/UI/CMBCreditCardAccountImportConfigurationView.swift`
- `MoneyFlow/Accounts/CMBPersonalChecking/UI/CMBPersonalCheckingAccountImportConfigurationView.swift`
- `MoneyFlow/Accounts/Plaid/UI/PlaidAccountConfigurationView.swift`

### App State
- `MoneyFlow/Core/State/AppState.swift` (importAccountTransactions method)

---

## 💡 Tips for Success

1. **Start Small**: One service at a time
2. **Test Early**: Write tests as you go
3. **Keep Old Code**: Don't remove until proven
4. **Document Changes**: Update docs as you integrate
5. **Get Feedback**: Test with real users early

---

**Ready to start? Begin with Phase 7, Task 7.1!**

