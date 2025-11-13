# HSplitView vs NavigationSplitView Comparison

## Key Differences

### **HSplitView**
- **Purpose**: Simple horizontal split with resizable divider
- **Control**: Full control over spacing, no system dividers
- **Layout**: Manual width management with `.frame()`
- **Divider**: User-draggable divider (optional)
- **Navigation**: No built-in navigation features
- **Use Case**: Custom layouts where you need precise control

### **NavigationSplitView**
- **Purpose**: Navigation-based multi-column interface
- **Control**: System-managed column widths (uses `.navigationSplitViewColumnWidth()`)
- **Layout**: Automatic layout management, adaptive to screen size
- **Divider**: System divider (always present, ~1-2pt)
- **Navigation**: Built-in navigation features, integrates with NavigationStack
- **Use Case**: Navigation-based apps (mail, file browsers, settings)

## Code Examples

### HSplitView
```swift
HSplitView {
    LeftPane()
        .frame(minWidth: 280, idealWidth: 300, maxWidth: 350)
    RightPane()
}
// Full control over spacing, no system divider
```

### NavigationSplitView
```swift
NavigationSplitView {
    LeftPane()
        .navigationSplitViewColumnWidth(min: 280, ideal: 300, max: 350)
} detail: {
    RightPane()
}
// System divider always present, automatic layout management
```

## For Your Use Case (AccountBookView)

### Current Issue
- NavigationSplitView adds a system divider (~1-2pt gap)
- Column width constraints via `.navigationSplitViewColumnWidth()`
- Less control over exact spacing

### If You Switch to HSplitView
- ✅ **No system divider** - complete control over spacing
- ✅ **Direct width control** - use `.frame()` modifiers
- ✅ **Customizable divider** - can hide or style it
- ❌ **No navigation features** - but you don't need them here
- ❌ **Manual layout management** - need to handle resizing yourself

## Recommendation

**For AccountBookView, HSplitView might be better** because:
1. You don't need navigation features
2. You want precise control over spacing (no gap)
3. You want direct width control
4. Your layout is simple (two panes side-by-side)

**Keep NavigationSplitView if:**
- You need adaptive layouts (iPad/iPhone)
- You want system navigation features
- You prefer automatic layout management

## Example: Converting to HSplitView

```swift
// Current (NavigationSplitView)
NavigationSplitView {
    LedgerListPane(...)
        .navigationSplitViewColumnWidth(min: 280, ideal: 300, max: 350)
} detail: {
    LedgerDetailPaneView(...)
}

// Alternative (HSplitView)
HSplitView {
    LedgerListPane(...)
        .frame(minWidth: 280, idealWidth: 300, maxWidth: 350)
    LedgerDetailPaneView(...)
}
// No system divider, full control!
```

## In Your Codebase

Looking at `PropertiesView.swift`, you're already using a custom `HStack` approach for similar needs:
```swift
HStack(alignment: .top, spacing: spacing) {
    AccountsLeftPane(...)
        .frame(width: leftWidth)
    AccountDetailPane(...)
        .frame(width: rightWidth)
}
```

This gives you the same benefits as HSplitView but with even more control (you can hide the divider completely).

