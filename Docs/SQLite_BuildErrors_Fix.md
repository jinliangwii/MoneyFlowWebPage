# SQLite.swift Build Errors - Fix Guide

## Common Build Errors and Solutions

### Error 1: "Unable to find module dependency: 'SQLite'"

**Solution**: Add SQLite.swift package dependency
1. Open Xcode
2. File → Add Package Dependencies...
3. URL: `https://github.com/stephencelis/SQLite.swift.git`
4. Version: Up to Next Major from `0.15.0`
5. Add to **MoneyFlow** target
6. Clean build: ⌘⇧K, then ⌘B

### Error 2: Type 'Statement' has no member 'columnNames'

**Fix**: Use different approach to get column names

```swift
// Instead of: statement.columnNames
// Use: Access columns by index directly
for index in 0..<statement.columnCount {
    let columnName = statement.columnNames[index]
    // ...
}
```

### Error 3: Cannot convert value of type 'X' to expected argument type 'Binding'

**Fix**: Ensure types match SQLite.Binding protocol
- String ✅
- Int64 ✅ (not Int)
- Double ✅
- Data ✅
- nil ✅

Convert Int to Int64, Decimal to String, etc.

### Error 4: 'Binding' is ambiguous

**Fix**: Use fully qualified name
```swift
// Instead of: Binding?
// Use: SQLite.Binding?
```

### Error 5: Statement.bind() returns different type

**Fix**: SQLite.swift's bind() returns a new Statement, chain them:
```swift
var stmt = statement
stmt = try stmt.bind(param1)
stmt = try stmt.bind(param2)
```

### Error 6: Row subscript access issues

**Fix**: Access row values correctly:
```swift
// Row is a Row type, access by index:
let value = row[0]  // First column
// Or by column name if available
```

## Quick Fix Checklist

- [ ] SQLite.swift package added via SPM
- [ ] Package added to MoneyFlow target (not just project)
- [ ] Clean build folder (⌘⇧K)
- [ ] Rebuild (⌘B)
- [ ] Check deployment target matches (iOS 13+)
- [ ] Verify import SQLite works

## Alternative: If SQLite.swift continues to have issues

If SQLite.swift API is too complex, we can:
1. Use Foundation's SQLite3 C API directly (more verbose but stable)
2. Create a simpler wrapper
3. Use a different SQLite Swift library

## Next Steps

1. **Share specific error messages** - I can provide targeted fixes
2. **Check Xcode build log** - Look for the first error (others might be cascading)
3. **Verify package installation** - Check Package Dependencies in Xcode

---

**If you can share the specific error messages**, I can provide exact fixes for each one.


