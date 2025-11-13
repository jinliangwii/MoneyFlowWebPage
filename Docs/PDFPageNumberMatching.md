# PDF Page Number Matching Logic

## Overview

This document explains how page numbers are correctly assigned to transactions extracted from multi-page PDF documents for CMB transaction statements.

## Problem

When extracting text from a multi-page PDF, we need to:
1. Track which page each transaction came from
2. Ensure page numbers are sequential (1, 2, 3, 4, 5) not incorrect (2025, 1, 2, 3, 4)

## Solution

### Step 1: Text Extraction with Page Markers

During PDF text extraction, markers are placed **AFTER** each page's content:

```swift
for pageIndex in 0..<pageCount {
    // Extract page text
    fullText += pageText
    
    // Add page separator AFTER each page (except the last)
    // Marker format: "--- Page X ---" where X indicates the NEXT page
    if pageIndex < pageCount - 1 {
        let nextPageNumber = pageIndex + 2  // pageIndex (0-based) + 1 (1-based) + 1 (next)
        fullText += "\n\n--- Page \(nextPageNumber) ---\n\n"
    }
}
```

**Example Output** (5 pages):
```
[Page 0 content]
--- Page 2 ---

[Page 1 content]
--- Page 3 ---

[Page 2 content]
--- Page 4 ---

[Page 3 content]
--- Page 5 ---

[Page 4 content]
```

### Step 2: Parsing Transactions with Page Numbers

When parsing, the text is split by markers:

```swift
// Split text by "--- Page " delimiter
let pageParts = text.components(separatedBy: "--- Page ")

for (index, part) in pageParts.enumerated() {
    if index == 0 {
        // First part: No marker before it → Always Page 1
        pageNumber = 1
    } else {
        // Extract page number from marker (e.g., "2 ---")
        let markerPattern = #"^(\d+)\s*---.*?\n"#
        // Extract number, assign to transactions
        pageNumber = extractedPageNumber
    }
    
    // Parse transactions from this page with assigned page number
    transactions.append(parsePageTransactions(part, pageNumber: pageNumber))
}
```

### Key Points

1. **First page (index 0)**: Always Page 1, no digit matching needed (avoids matching years like "2025")
2. **Marker placement**: Markers placed **AFTER** page content, marker number = next page number
3. **Marker extraction**: Uses regex `^(\d+)\s*---` to extract page number from marker
4. **Sequential numbering**: Results in pages 1, 2, 3, 4, 5 correctly assigned

## Implementation Location

- **Extraction**: `CMBTransactionPDFKitExtractor.extractText(from:)`
- **Parsing**: `CMBTransactionPDFKitExtractor.parseTransactions(from:pdfInfo:...)`
- **Model**: `RawCMBTransaction` has `pdfPageNumber: Int` field


