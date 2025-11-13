# Why We Have Both Zip and ZIPFoundation Dependencies

## Overview

Our project includes two ZIP-related libraries:
- **Zip** (marmelroy/Zip) - Direct dependency
- **ZIPFoundation** (weichsel/ZIPFoundation) - Transitive dependency (via CoreXLSX)

This document explains why both exist and their respective roles.

## Dependency Structure

```
MoneyFlow
├── Zip (direct dependency)
│   └── Used for: Password-protected ZIP extraction
│
└── CoreXLSX (direct dependency)
    └── ZIPFoundation (transitive dependency)
        └── Used for: xlsx file parsing (internal to CoreXLSX)
```

## Why Both Libraries?

### 1. Zip Library (marmelroy/Zip) - Direct Dependency

**Purpose:** Extract password-protected ZIP files

**Used in:**
- `ZipFileProcessingService.swift` - Core ZIP extraction service
- CMB import - Extract password-protected PDF files from ZIP
- Alipay import - Extract password-protected CSV files from ZIP

**Why we need it:**
- ✅ **Password protection support** - Essential for CMB and Alipay imports
- ✅ **Simple API** - `Zip.unzipFile()` with password parameter
- ✅ **Proven reliability** - Works well for our use case

**Code location:**
```swift
// MoneyFlow/Core/Services/ZipFileProcessingService.swift
import Zip

try Zip.unzipFile(
    zipURL,
    destination: tempDir,
    overwrite: true,
    password: password.isEmpty ? nil : password,
    progress: nil
)
```

### 2. ZIPFoundation - Transitive Dependency (via CoreXLSX)

**Purpose:** Parse xlsx files (used internally by CoreXLSX)

**Used in:**
- `WeChatExcelParser.swift` - Uses CoreXLSX to parse WeChat Excel exports
- CoreXLSX library internally uses ZIPFoundation to extract xlsx files

**Why it exists:**
- ✅ **Not directly used** - We don't import or use ZIPFoundation in our code
- ✅ **Transitive dependency** - Automatically included because CoreXLSX depends on it
- ✅ **CoreXLSX needs it** - CoreXLSX uses ZIPFoundation to extract xlsx files (which are ZIP archives)

**Code location:**
```swift
// MoneyFlow/Features/WeChat/Services/WeChatExcelParser.swift
import CoreXLSX  // This brings in ZIPFoundation as a transitive dependency

let file = XLSXFile(filepath: excelURL.path)
// CoreXLSX internally uses ZIPFoundation to extract the xlsx file
```

## Why Not Use ZIPFoundation for Everything?

We considered using ZIPFoundation for password-protected ZIP extraction, but:

1. **Password support complexity** - ZIPFoundation's API for password-protected ZIPs is more complex
2. **Different use cases** - 
   - Zip library: Simple password-protected ZIP extraction (CMB, Alipay)
   - ZIPFoundation: xlsx parsing (via CoreXLSX, for WeChat)
3. **Stability** - Zip library works reliably for our password-protected ZIP needs
4. **No direct control** - ZIPFoundation comes as a transitive dependency, we don't control its version

## Current Status

### Direct Dependencies
- ⚠️ **Zip** (marmelroy/Zip) - **Needs to be added in Xcode** for password-protected ZIP extraction
  - Currently used in code but may need to be added as a package dependency
  - URL: `https://github.com/marmelroy/Zip.git`
  - Version: 2.1.0 (recommended)

### Transitive Dependencies (via CoreXLSX)
- ✅ **ZIPFoundation** (v0.9.20) - Automatically included by CoreXLSX
- ✅ **XMLCoder** (v0.14.0) - Also included by CoreXLSX

### Adding Zip Library

If you see compilation errors about missing `Zip` module:

1. Open `MoneyFlow.xcodeproj` in Xcode
2. Go to **File → Add Package Dependencies...**
3. Enter URL: `https://github.com/marmelroy/Zip.git`
4. Select version: `2.1.0` (or latest)
5. Add to **MoneyFlow** target
6. Build project

## Summary

| Library | Type | Purpose | Used For |
|---------|------|---------|----------|
| **Zip** | Direct | Password-protected ZIP extraction | CMB, Alipay imports |
| **ZIPFoundation** | Transitive | xlsx file parsing | WeChat Excel parsing (via CoreXLSX) |

**Key Points:**
- We **directly use** Zip library for password-protected ZIP files
- We **don't directly use** ZIPFoundation - it's only here because CoreXLSX needs it
- Both libraries serve different purposes and are not redundant
- This is a reasonable dependency structure given our requirements

## Future Considerations

If we ever want to consolidate:
1. **Option 1:** Keep current structure (recommended)
   - Simple and works well
   - Clear separation of concerns

2. **Option 2:** Use ZIPFoundation for everything
   - Would require implementing password-protected ZIP extraction ourselves
   - More complex code
   - Not recommended unless ZIPFoundation adds better password support

3. **Option 3:** Remove CoreXLSX and use Zip for xlsx
   - Would require manual XML parsing (we tried this, it's complex)
   - Not recommended - CoreXLSX is much better for xlsx parsing

**Recommendation:** Keep the current structure. It's clean, works well, and each library serves its purpose.

