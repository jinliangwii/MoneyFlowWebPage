# CMB (China Merchants Bank) Feature

Account-specific implementation for China Merchants Bank accounts.

## Overview

This feature provides CMB-specific functionality:
- Account creation with ZIP file upload
- Balance extraction from PDF statements
- Transaction import from encrypted ZIP/PDF files
- Balance verification

## Structure

```
CMB/
├── Provider/
│   ├── CMBAccountProvider.swift   # Account creation provider
│   └── CMBAccountFeatureProvider.swift    # Feature provider (import, UI)
├── Import/
│   └── CMBImportView.swift                # Import UI
├── Services/
│   ├── CMBImportService.swift             # Import logic
│   ├── CMBPDFParser.swift                 # PDF parsing
│   └── CMBTransactionPDFKitExtractor.swift # Text extraction
└── Domain/
    ├── RawCMBTransaction.swift            # Raw transaction model
    └── ImportCMBTransactionBatch.swift    # Import batch tracking
```

## Provider

### CMBAccountProvider
Handles CMB account creation:
- ZIP file selection
- Password input
- Balance extraction
- Account configuration

### CMBAccountFeatureProvider
Provides CMB-specific features:
- Import button in account detail view
- Import view
- Import service integration

## Import Flow

1. User selects encrypted ZIP file
2. User enters password
3. Extract PDF from ZIP
4. Parse PDF to extract transactions
5. Verify balance calculation
6. Detect duplicates
7. Convert to Transaction models
8. Update account balance

## Services

### CMBImportService
Orchestrates the import process:
- ZIP extraction
- PDF parsing coordination
- Balance verification
- Duplicate detection
- Transaction conversion

### CMBPDFParser
Parses CMB PDF statements:
- Extracts account information
- Parses transaction data
- Handles multiple pages

### CMBTransactionPDFKitExtractor
Extracts text from PDF pages using PDFKit.

## Domain Models

### RawCMBTransaction
Preserves original PDF data before conversion.

### ImportCMBTransactionBatch
Tracks import batches for audit trail.

## Adding New Account Types

To add support for another bank:
1. Create new folder under `Features/[BankName]/`
2. Follow the same structure (Provider, Import, Services, Domain)
3. Implement `AccountFeatureProvider` and `AccountImportProvider`
4. Register in `AccountFeatureRegistry`

