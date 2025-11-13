# CoreXLSX Setup Guide

## Adding CoreXLSX Dependency

### Step 1: Add Package in Xcode

1. Open `MoneyFlow.xcodeproj` in Xcode
2. Go to **File → Add Package Dependencies...**
3. Enter the URL: `https://github.com/CoreOffice/CoreXLSX.git`
4. Select the latest version (or a specific version)
5. Ensure **MoneyFlow** target is selected
6. Click **Add Package**

### Step 2: Verify Installation

1. In Xcode, go to **Project Navigator** (left sidebar)
2. Expand **Package Dependencies**
3. You should see `CoreXLSX` listed
4. If not visible, go to **File → Packages → Reset Package Caches** and try again

### Step 3: Build Project

1. Clean build folder: **⌘⇧K**
2. Build project: **⌘B**
3. Verify no errors

## What Changed

- `WeChatExcelParser.swift` now uses CoreXLSX instead of Zip library
- Removed all Zip-related code for xlsx parsing
- CoreXLSX provides proper xlsx parsing without manual XML handling

## Benefits

- ✅ Proper xlsx format support
- ✅ No manual XML parsing needed
- ✅ Handles shared strings correctly
- ✅ Better error handling
- ✅ More reliable parsing

