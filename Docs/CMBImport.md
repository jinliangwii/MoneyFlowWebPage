# CMB (China Merchants Bank) Import Feature

## Overview

This feature allows users to import transaction statements from China Merchants Bank (招商银行) by uploading encrypted ZIP files containing PDF transaction statements.

## Feature Flow

1. User clicks "导入招商银行流水" button in account detail view
2. User selects encrypted ZIP file
3. User enters ZIP password (not saved)
4. System extracts ZIP → PDF
5. System parses PDF → RawCMBTransaction objects
6. System converts RawCMBTransaction → Transaction objects
7. System detects duplicates
8. System associates transactions with current account
9. Transactions imported to system

## Architecture

### Data Models

**RawCMBTransaction** - Stores original PDF parsed data (CMB-specific format):
- Raw fields from PDF (date, currency, amount, balance, transaction type, counterparty)
- Parsed fields (Date, Decimal amounts)
- Metadata (source file, page number, import batch ID)

**ImportCMBTransactionBatch** - Tracks each import batch:
- Source file name
- Import date/time
- Account ID
- Statistics (total records, successful imports, duplicates)

### Key Components

- `CMBImportView` - UI for file selection and password input
- `CMBImportService` - Orchestrates import process
- `CMBPDFParser` - Parses PDF files (uses PDFKit with OCR fallback)
- `ZipFileProcessingService` - Handles ZIP extraction
- `CMBRepository` - Stores CMB-specific data (raw transactions, import batches)

## Technical Details

### ZIP Extraction
- Uses `Zip` library (https://github.com/marmelroy/Zip)
- Supports password-protected ZIP files
- Pure Swift, SPM compatible

### PDF Parsing
- **Primary**: PDFKit for text extraction (text-based PDFs)
- **Fallback**: Vision framework OCR (for scanned PDFs)
- Handles multi-page PDFs with page markers
- See `PDFPageNumberMatching.md` for page numbering logic

### Duplicate Detection
- Based on: date, amount, transaction type, counterparty, account ID
- Compares against existing RawCMBTransaction and Transaction objects
- Skips exact duplicates automatically

### Page Number Matching
- Pages marked with `--- Page X ---` markers during extraction
- First page (no marker) is always Page 1
- Subsequent pages extract number from marker
- See `PDFPageNumberMatching.md` for detailed logic

## File Structure

```
Features/CMB/
├── Data/
│   └── CMBRepository.swift
├── Domain/
│   ├── RawCMBTransaction.swift
│   └── ImportCMBTransactionBatch.swift
├── Import/
│   └── CMBImportView.swift
├── Provider/
│   ├── ChinaMerchantsBankProvider.swift
│   └── CMBAccountFeatureProvider.swift
└── Services/
    ├── CMBImportService.swift
    ├── CMBPDFParser.swift
    └── CMBTransactionPDFKitExtractor.swift
```

## Related Documents

- `PDFPageNumberMatching.md` - PDF page numbering implementation details
- `AccountFeatureProviderArchitecture.md` - Provider pattern for account-specific features
- `DecouplingPlan_CMBFromCore.md` - Plan to decouple CMB from core layer


