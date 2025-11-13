Agent Query: must review the PDF included in /Users/jinliangwei/Developer/MoneyFlow/MoneyFlowTests/TestData/招商银行个人贷款还款账单20251108204310_110016.zip. Zip archive's password is 110016.
Use pythin3 to read the PDF structure. If you failed to read, do not continue. I extract the PDF for you, /Users/jinliangwei/Developer/MoneyFlow/MoneyFlowTests/TestData/CPEC0490A2AF00370000_03.pdf
Add a CMB Express Loan 闪电贷 account account type, debug pane. 
Please do not simply copy existing logic, analyze the data from PDF and then design. 
Inlucde all funtionality we have for other account type, include it here. Plus, this is a new account type, debt account, things may be different. Think throughfully, give me a quick proposal. do not add code yet.
Account meta data shoud include all info mentioned in header section beside customer info.

# Proposal: CMB Express Loan (闪电贷) Account Type

## 1. PDF Structure Analysis

### Header Section (贷款关键信息):
- **贷款编号** (Loan Number): 6113431666900001
- **客户号** (Customer Number): 1134316669
- **客户名称** (Customer Name): 魏金亮
- **证件类型** (ID Type): P01-居民身份证
- **证件号码** (ID Number): 520114199504110016
- **放款日** (Disbursement Date): 20250504
- **业务品种名称** (Product Name): 闪电贷
- **贷款金额** (Loan Amount): 180,000
- **贷款余额** (Outstanding Balance): 163,285.79
- **贷款期限** (Loan Term): 60 months
- **贷款还款方式** (Repayment Method): SAM-按月结息，按月还本，等额本息
- **执行利率** (Interest Rate): 3.25%
- **贷款状态** (Loan Status): USED-正常
- **第一扣款账号** (Deduction Account): 6214850310760811

### Transaction Section (已出账单列表):
Each payment record includes:
- **应扣日期** (Due Date)
- **期数** (Period Number)
- **账单状态** (Statement Status)
- **扣款状态** (Payment Status)
- **应还本金/实还本金** (Principal Due/Paid)
- **应还利息/实还利息** (Interest Due/Paid)
- **应还罚息/实还罚息** (Penalty Due/Paid)
- **应还复息/实还复息** (Compound Interest Due/Paid)
- **应还合计/实还合计** (Total Due/Paid)
- **欠款合计** (Outstanding Amount)
- **首次扣款日/实扣日期** (First Deduction Date/Actual Deduction Date)

## 2. Account Type Considerations

### Debt Account Specifics:
1. **Balance**: Outstanding loan amount (negative, represents debt)
2. **Transactions**: Payments that reduce the balance (negative amounts from user perspective)
3. **Transaction Structure**: Track principal, interest, penalties separately
4. **Account Metadata**: Store all header fields (excluding customer personal info)

## 3. Proposed Architecture

### 3.1 Domain Models

#### `RawCMBExpressLoanTransaction.swift`:
- **Raw fields** (from transaction table only):
  - 应扣日期 (due date)
  - 期数 (period number)
  - 账单状态 (statement status)
  - 扣款状态 (payment status)
  - 应还本金/实还本金 (principal due/paid)
  - 应还利息/实还利息 (interest due/paid)
  - 应还罚息/实还罚息 (penalty due/paid)
  - 应还复息/实还复息 (compound interest due/paid)
  - 应还合计/实还合计 (total due/paid)
  - 欠款合计 (outstanding amount for this period)
  - 首次扣款日/实扣日期 (first deduction date/actual deduction date)
- **Parsed fields**: date, total payment amount, principal, interest, penalty
- **Note**: Balance (贷款余额) is in header section, not in transaction records

#### `ImportCMBExpressLoanTransactionBatch.swift`:
- Tracks import batches with metadata (source file, dates, counts)

#### `CMBExpressLoanAccountMetadata.swift`:
- **贷款编号** (loanNumber)
- **客户号** (customerNumber) - optional, for matching
- **放款日** (disbursementDate)
- **业务品种名称** (productName): "闪电贷"
- **贷款金额** (loanAmount)
- **贷款期限** (loanTerm) - in months
- **贷款还款方式** (repaymentMethod)
- **贷款状态** (loanStatus)
- **第一扣款账号** (deductionAccount) - last 4 digits
- **Note**: Interest rate (执行利率) is stored in `Account.interestRate` field, not in metadata

### 3.2 Data Layer

#### `CMBExpressLoanRepository.swift`:
- Protocol: `CMBExpressLoanRepositoryProtocol`
- Implementation: `CMBExpressLoanRepository` (actor)
- Methods:
  - Save/load raw transactions
  - Save/load import batches
  - Save/load account metadata
  - Find metadata by loan number

### 3.3 Services Layer

#### `CMBExpressLoanPDFParser.swift`:
- Extract header information (loan metadata)
- Parse payment records from transaction table
- Handle PDF text extraction (PDFKit-based)
- Return structured data: `CMBExpressLoanPDFInfo` and `RawCMBExpressLoanTransaction[]`

#### `CMBExpressLoanImportService.swift`:
- Orchestrate import process
- **ZIP File Handling**: Extract PDF from password-protected ZIP file using `ZipFileProcessingService`
- Parse PDF → Filter duplicates → Convert to Transactions → Save
- Handle debt account logic:
  - Payments reduce balance (negative amounts)
  - Balance = outstanding loan amount
  - Track principal vs interest separately in transaction notes
- **Duplicate Detection**: Use `CMBExpressLoanDuplicateDetectionStrategy`
  - Match by: actual paid date (实扣日期) + total amount (实还合计)
  - Implement `DuplicateDetectionStrategy` protocol

### 3.4 Provider Layer

#### `CMBExpressLoanAccountProvider.swift`:
- Account creation UI (ZIP file picker with password input)
- Validation (require loan number, balance from PDF)
- Account creation with metadata
- Currency: Always set to CNY (no user selection)
- Interest Rate: Extract from PDF header (执行利率) and store in Account.interestRate as Decimal

#### `CMBExpressLoanAccountFeatureProvider.swift`:
- Import button/view
- Custom actions for loan accounts

### 3.5 UI Layer

#### `CMBExpressLoanAccountImportConfigurationView.swift`:
- ZIP file picker with password input
- Extract PDF from ZIP file (password-protected)
- Display extracted loan information
- Account name generation: "闪电贷(贷款编号后4位) - 人民币"
- Currency: Always CNY (no currency selection needed)

#### `CMBExpressLoanImportView.swift`:
- Import UI with ZIP file picker and password input
- Progress tracking for ZIP extraction and PDF parsing

#### `CMBExpressLoanAutoImportView.swift` (Debug Pane):
- Auto-detect ZIP files from test data
- Filter files: "招商银行" + "个人贷款还款账单" or "闪电贷"
- Extract password from filename (last 6 digits before extension)
- Auto-create accounts and import transactions

## 4. Import Process

### ZIP File Processing:
1. **ZIP Extraction**: Extract PDF from password-protected ZIP file
   - Password typically extracted from filename (last 6 digits before extension)
   - Use `ZipFileProcessingService` to handle extraction
2. **PDF Parsing**: Parse extracted PDF for header and transaction data
3. **Transaction Import**: Process transactions as described below

### From PDF Payment Record to Transaction:
- **Date**: 实扣日期 (actual payment date)
- **Amount**: -实还合计 (negative, as it's a payment reducing debt)
- **Merchant**: "招商银行 - 贷款还款"
- **Category**: `TransactionCategory.other`
- **Notes**: "期数: X | 本金: ¥X,XXX | 利息: ¥XXX | 罚息: ¥X"
- **Balance**: Calculated based on loan amount and principal payments (not from transaction record)
- **Sequence number**: Based on period number

### Duplicate Detection:
- **Strategy**: `CMBExpressLoanDuplicateDetectionStrategy`
- **Fingerprint**: actual paid date (实扣日期) + total amount (实还合计) + accountId
- **Match Criteria**: Same date + same amount (both negative for payments)
- **Currency**: Not needed in matching (always CNY)
- Transactions with same payment date and same total paid amount are considered duplicates

### Balance Calculation:
- Start with loan amount (贷款金额) from header (stored as negative: -贷款金额)
- Each payment reduces balance by principal amount (实还本金) - balance becomes less negative
- Final balance = outstanding balance (贷款余额) from header section (stored as negative: -贷款余额)
- Balance is calculated during conversion, not stored in raw transaction

## 5. Account Metadata Storage

### Store in `Account.metadata` dictionary:
- `loanNumber`: Full loan number
- `loanAmount`: Original loan amount
- `loanTerm`: Term in months
- `repaymentMethod`: Repayment method code/description
- `loanStatus`: Current status
- `deductionAccountLast4`: Last 4 digits of deduction account
- `disbursementDate`: Disbursement date

### Store in `Account.interestRate`:
- **Interest Rate** (执行利率): Stored as Decimal (e.g., 3.25 for 3.25%)

### Store in `CMBExpressLoanAccountMetadata`:
- All metadata fields in structured format (excluding interestRate)
- Used for duplicate detection and account matching
- Interest rate is stored separately in `Account.interestRate` as Decimal

## 6. Differences from Credit Card Accounts

1. **Balance Semantics**: Both represent debt and should be negative (outstanding loan amount as negative, credit card balance as negative)
2. **Transaction Direction**: Payments reduce balance (negative amounts)
3. **Transaction Details**: Track principal, interest, penalties separately
4. **Account Metadata**: More fields (loan term, repayment method, etc.)
5. **Account Type**: Use `.loan` instead of `.creditCard`
6. **Account Name**: "闪电贷(XXXX) - 人民币" format

## 7. Implementation Checklist

- [ ] Domain models (RawTransaction, ImportBatch, Metadata)
- [ ] Data layer (Repository with metadata support)
- [ ] Services (PDF Parser, Import Service)
- [ ] Duplicate Detection Strategy (`CMBExpressLoanDuplicateDetectionStrategy`)
  - Implement `DuplicateDetectionStrategy` protocol
  - `generateFingerprint`: date (实扣日期) + amount (实还合计) + accountId
  - `isDuplicate`: Match by date + amount (negative for payments)
- [ ] Provider (AccountProvider, AccountFeatureProvider)
- [ ] UI (Configuration view, Import view, Debug pane)
- [ ] Registration (ProviderRegistry, AccountFeatureRegistry)
- [ ] Testing (PDF parsing, transaction conversion, duplicate detection)

## 8. Questions to Consider

1. ~~**Transaction Category**: Use existing `other` or add `loanPayment`?~~ ✅ **Decided**: Use `TransactionCategory.other`
2. ~~**Balance Display**: Show as positive (outstanding) or negative (debt)?~~ ✅ **Decided**: Store as negative (represents debt)
3. ~~**Duplicate Detection**: Match by loan number + currency?~~ ✅ **Decided**: Match by actual paid date (实扣日期) + total amount (实还合计)
4. ~~**Multi-Currency**: Support multiple currencies (likely CNY only)?~~ ✅ **Decided**: CNY only
5. ~~**Interest Rate**: Store as Decimal in Account.interestRate field?~~ ✅ **Decided**: Yes, store as Decimal in Account.interestRate field

