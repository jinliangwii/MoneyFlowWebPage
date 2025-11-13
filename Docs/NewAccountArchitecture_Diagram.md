# New Account Architecture - Visual Diagrams

**Date:** 2025-01-XX

---

## 1. Overall Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application Layer                        │
│  (UI, ViewModels, Account Creation Flow)                         │
└────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Account Factory Layer                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              AccountFactory (Singleton)                   │  │
│  │  - register(accountType, dataSourceType, factory)         │  │
│  │  - createAccount(type, dataSource, draft) → BaseAccount   │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
┌──────────────────────────┐   ┌──────────────────────────┐
│   Account Type Layer     │   │   Data Source Layer      │
│   (Inheritance)          │   │   (Protocol)              │
└──────────────────────────┘   └──────────────────────────┘
```

---

## 2. Account Type Hierarchy

```
                    ┌─────────────────────┐
                    │   BaseAccount       │
                    │  (Abstract Class)   │
                    │                     │
                    │ + id: UUID          │
                    │ + name: String      │
                    │ + institution       │
                    │ + currencyCode      │
                    │ + balance: Decimal  │
                    │                     │
                    │ + calculateBalance()│
                    │ + validateBalance() │
                    │ + toAccountStruct() │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│CreditCard     │    │LoanAccount    │    │CheckingAccount│
│Account        │    │               │    │               │
│               │    │+ loanAmount   │    │               │
│+ creditLimit  │    │+ interestRate │    │               │
│+ available    │    │+ loanTerm     │    │               │
│  Credit       │    │               │    │               │
└───────┬───────┘    └───────┬───────┘    └───────┬───────┘
        │                    │                      │
        │                    │                      │
┌───────┴───────┐    ┌───────┴───────┐    ┌───────┴───────┐
│CMBCreditCard  │    │CMBExpressLoan  │    │CMBPersonal    │
│Account        │    │Account         │    │CheckingAccount│
│               │    │                │    │               │
│+ accountNumber│    │+ loanNumber    │    │+ accountNumber│
└───────────────┘    │+ customerNumber│    └───────────────┘
                     └────────────────┘
        │
        ▼
┌───────────────┐
│PlaidCreditCard│
│Account        │
│               │
│+ plaidItemId  │
│+ plaidAccountId│
└───────────────┘

        ┌───────────────┐
        │PaymentGateway │
        │Account        │
        │               │
        │+ email/nickname│
        └───────┬───────┘
                │
        ┌───────┴───────┐
        │               │
┌───────┴───────┐ ┌────┴──────────┐
│AlipayAccount  │ │WeChatAccount  │
│               │ │               │
│+ email        │ │+ nickname     │
└───────────────┘ └───────────────┘
```

---

## 3. Data Source Abstraction

```
                    ┌─────────────────────┐
                    │DataSourceProtocol    │
                    │  (Protocol)          │
                    │                     │
                    │ + extractMetadata() │
                    │ + parseTransactions()│
                    │ + getSourceInfo()   │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│PDFDataSource  │    │PlaidDataSource│   │CSVDataSource│
│(Base Class)   │    │               │   │(Base Class)  │
│               │    │+ plaidService │   │              │
│+ pdfParser    │    │+ itemId       │   │              │
└───────┬───────┘    └───────────────┘   └───────┬───────┘
        │                                        │
        │                                        │
┌───────┴───────┐                    ┌─────────┴─────────┐
│               │                    │                     │
▼               ▼                    ▼                     ▼
┌───────────────┐  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│CMBCreditCard  │  │CMBPersonal    │ │AlipayCSV      │ │WeChatExcel    │
│PDFDataSource  │  │CheckingPDF    │ │DataSource     │ │DataSource     │
│               │  │DataSource     │ │               │ │               │
│→ CMBCreditCard│  │               │ │→ AlipayCSV    │ │→ WeChatExcel  │
│  PDFParser    │  │→ CMBPersonal  │ │  Parser       │ │  Parser       │
└───────────────┘  │  CheckingPDF  │ └───────────────┘ └───────────────┘
                   │  Parser       │
                   └───────────────┘
```

---

## 4. Account Factory Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    AccountFactory                            │
│                                                              │
│  factories: [AccountType: [DataSourceType: Factory]]         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Registration Map:                                    │   │
│  │                                                      │   │
│  │ .creditCard →                                        │   │
│  │   .pdf → CMBCreditCardAccountFactory                │   │
│  │   .plaid → PlaidCreditCardAccountFactory            │   │
│  │                                                      │   │
│  │ .deposit →                                           │   │
│  │   .pdf → CMBPersonalCheckingAccountFactory         │   │
│  │   .plaid → PlaidCheckingAccountFactory             │   │
│  │                                                      │   │
│  │ .loan →                                              │   │
│  │   .pdf → CMBExpressLoanAccountFactory              │   │
│  │                                                      │   │
│  │ .paymentGate →                                       │   │
│  │   .csv → AlipayAccountFactory                       │   │
│  │   .excel → WeChatAccountFactory                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  createAccount(type, dataSource, draft) → BaseAccount       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ AccountCreation  │
                    │ Factory          │
                    │ (Protocol)       │
                    │                  │
                    │ + createAccount()│
                    └─────────┬────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│CMBCreditCard  │   │CMBExpressLoan  │   │AlipayAccount  │
│AccountFactory │   │AccountFactory  │   │Factory        │
│               │   │                │   │               │
│1. Extract     │   │1. Extract      │   │1. Extract     │
│   metadata    │   │   metadata     │   │   metadata    │
│2. Create      │   │2. Create       │   │2. Create      │
│   account     │   │   account      │   │   account     │
└───────────────┘   └───────────────┘   └───────────────┘
```

---

## 5. Complete Flow: Creating an Account

```
User Action: "Add CMB Credit Card Account"
        │
        ▼
┌─────────────────────────────────────┐
│  AccountProvider                     │
│  (CMBCreditCardAccountProvider)      │
│                                      │
│  - makeConfigurationView()         │
│  - validate()                        │
│  - createAccount()                   │
└──────────────┬───────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  AccountFactory.createAccount()      │
│                                      │
│  1. Infer dataSourceType (.pdf)     │
│  2. Lookup factory                   │
│  3. Call factory.createAccount()     │
└──────────────┬───────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  CMBCreditCardAccountFactory        │
│                                      │
│  1. Get PDFDataSource               │
│  2. Extract metadata from PDF        │
│     └─> CMBCreditCardPDFParser      │
│  3. Create CMBCreditCardAccount      │
│     └─> new CMBCreditCardAccount()   │
└──────────────┬───────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  CMBCreditCardAccount                │
│  (extends CreditCardAccount)         │
│                                      │
│  - accountNumber (from metadata)     │
│  - creditLimit                       │
│  - balance (from PDF)                │
└─────────────────────────────────────┘
```

---

## 6. Data Flow: Importing Transactions

```
User Action: "Import Transactions"
        │
        ▼
┌─────────────────────────────────────┐
│  AccountFeatureProvider              │
│  (CMBCreditCardAccountFeatureProvider)│
│                                      │
│  performImport()                     │
└──────────────┬───────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  CMBCreditCardImportService          │
│  (BaseImportService)                 │
│                                      │
│  1. Get DataSource                   │
│     └─> CMBCreditCardPDFDataSource  │
│  2. Parse transactions               │
│     └─> CMBCreditCardPDFParser       │
│  3. Process transactions             │
│     └─> BaseImportService.execute() │
│  4. Convert to Transactions          │
│     └─> CMBCreditCardTransaction  │
│         Converter                     │
└──────────────┬───────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Transaction[]                       │
│  (Saved to Repository)               │
└─────────────────────────────────────┘
```

---

## 7. Component Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                    Account (Domain)                          │
│                                                              │
│  BaseAccount ──────┐                                         │
│       │            │                                         │
│       │            │ inherits                                │
│       │            │                                         │
│       ▼            ▼                                         │
│  CreditCardAccount, LoanAccount, CheckingAccount, etc.      │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Data Source (Data Layer)                    │
│                                                              │
│  DataSourceProtocol ──────┐                                 │
│       │                    │                                 │
│       │                    │ implements                     │
│       │                    │                                 │
│       ▼                    ▼                                 │
│  PDFDataSource, PlaidDataSource, CSVDataSource, etc.       │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Import Service (Service Layer)               │
│                                                              │
│  BaseImportService ──────┐                                   │
│       │                  │                                   │
│       │                  │ inherits                         │
│       │                  │                                   │
│       ▼                  ▼                                   │
│  CMBCreditCardImportService, AlipayImportService, etc.       │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Repository (Data Layer)                      │
│                                                              │
│  RawTransactionRepository ──────┐                           │
│       │                         │                            │
│       │                         │ implements                │
│       │                         │                            │
│       ▼                         ▼                            │
│  CMBCreditCardRepository, AlipayRepository, etc.            │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. Adding a New Account Type (Example: ICBC Credit Card)

```
Step 1: Create Account Class
┌─────────────────────────────────────┐
│  ICBCCreditCardAccount               │
│  extends CreditCardAccount           │
│                                      │
│  + ICBC-specific fields              │
└─────────────────────────────────────┘

Step 2: Create Data Source
┌─────────────────────────────────────┐
│  ICBCCreditCardPDFDataSource         │
│  extends PDFDataSource               │
│                                      │
│  + ICBC PDF parsing logic            │
└─────────────────────────────────────┘

Step 3: Create Factory
┌─────────────────────────────────────┐
│  ICBCCreditCardAccountFactory         │
│  implements AccountCreationFactory   │
│                                      │
│  + createAccount() logic             │
└─────────────────────────────────────┘

Step 4: Register
┌─────────────────────────────────────┐
│  AccountFactory.shared.register(     │
│    .creditCard,                     │
│    .pdf,                            │
│    ICBCCreditCardAccountFactory()    │
│  )                                   │
└─────────────────────────────────────┘

Done! No core code changes needed.
```

---

## 9. Separation of Concerns

```
┌─────────────────────────────────────────────────────────────┐
│                    Account Creation                          │
│                    (Business Logic)                          │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │Account Types │  │Account       │  │Account       │     │
│  │(Inheritance) │  │Factory       │  │Providers      │     │
│  │              │  │              │  │              │     │
│  │- Behavior    │  │- Creation    │  │- UI          │     │
│  │- Validation  │  │- Registration│  │- Validation  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ uses
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Source                               │
│                    (Data Access)                             │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │PDF           │  │Plaid         │  │CSV/Excel     │     │
│  │DataSource    │  │DataSource    │  │DataSource    │     │
│  │              │  │              │  │              │     │
│  │- Parse PDF   │  │- API calls   │  │- Parse files │     │
│  │- Extract     │  │- Extract     │  │- Extract     │     │
│  │  metadata    │  │  metadata    │  │  metadata    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. Migration Path

```
Current State                    Phase 1              Phase 2
┌─────────────┐              ┌─────────────┐      ┌─────────────┐
│Account      │              │Account      │      │Account      │
│(struct)     │              │(struct)     │      │(struct)     │
│             │              │             │      │(deprecated) │
│             │              │+ BaseAccount│      │             │
│             │              │(new)        │      │BaseAccount  │
│             │              │             │      │(primary)    │
└─────────────┘              └─────────────┘      └─────────────┘
     │                              │                    │
     │                              │                    │
     └──────────────────────────────┴────────────────────┘
                    Conversion methods
                    (toAccountStruct())

Phase 3              Phase 4              Phase 5
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│Account      │      │Account       │      │Account      │
│(removed)    │      │(removed)     │      │(removed)    │
│             │      │              │      │             │
│BaseAccount  │      │BaseAccount   │      │BaseAccount  │
│+ DataSource │      │+ DataSource  │      │+ DataSource │
│             │      │+ Factory     │      │+ Factory    │
│             │      │              │      │(complete)   │
└─────────────┘      └─────────────┘      └─────────────┘
```

---

## Legend

- **Solid lines (──)**: Inheritance/Implementation
- **Dashed lines (- -)**: Uses/Depends on
- **Arrows (→)**: Data flow
- **Boxes**: Classes/Protocols/Components
- **Rounded boxes**: Instances/Objects

