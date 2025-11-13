# SQLite.swift Setup Guide

## Adding SQLite.swift Dependency

### Step 1: Add Package in Xcode

1. Open `MoneyFlow.xcodeproj` in Xcode
2. Go to **File → Add Package Dependencies...** (or **File → Add Package...**)
3. Enter the URL: `https://github.com/stephencelis/SQLite.swift.git`
4. Select version: **Up to Next Major Version** with `0.15.0` (or latest stable)
5. Ensure **MoneyFlow** target is selected
6. Click **Add Package**

### Step 2: Verify Installation

1. In Xcode, go to **Project Navigator** (left sidebar)
2. Expand **Package Dependencies**
3. You should see `SQLite.swift` listed
4. If not visible, go to **File → Packages → Reset Package Caches** and try again

### Step 3: Build Project

1. Clean build folder: **⌘⇧K** (Cmd+Shift+K)
2. Build project: **⌘B** (Cmd+B)
3. Verify no errors

## Verification

After adding the package, you should be able to:
- ✅ Import SQLite in `SQLiteDatabase.swift`
- ✅ Import SQLite in `SchemaManager.swift`
- ✅ Build without "Unable to find module dependency" errors

## Package Details

- **Repository**: https://github.com/stephencelis/SQLite.swift
- **License**: MIT (commercial-friendly)
- **Version**: 0.15.x or newer (recommended)
- **Platform**: iOS 13+, macOS 10.15+

## Troubleshooting

### If package doesn't appear:
1. Check internet connection
2. Go to **File → Packages → Reset Package Caches**
3. Try **File → Packages → Update to Latest Package Versions**

### If build fails:
1. Clean build folder: **⌘⇧K**
2. Delete derived data: **Xcode → Settings → Locations → Derived Data → Delete**
3. Rebuild: **⌘B**

### If import still fails:
1. Verify package is added to **MoneyFlow** target (not just project)
2. Check **Build Phases → Link Binary With Libraries** includes SQLite
3. Restart Xcode

---

**Note**: This is a required dependency for Phase 1 & 2 implementation. All SQLite code depends on this package.


