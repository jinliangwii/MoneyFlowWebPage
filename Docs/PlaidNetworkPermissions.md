# Plaid Network Permissions Fix

## Problem
"无法创建连接令牌" (Failed to create link token) - This is a network permissions issue.

## Solution: Enable Outgoing Network Connections

### For macOS

Your app has **App Sandbox** enabled with **Outgoing Network Connections = NO**. This blocks Plaid API calls.

**Fix:**

1. Open Xcode
2. Select **MoneyFlow** project (top of Project Navigator)
3. Select **MoneyFlow** target
4. Go to **Signing & Capabilities** tab
5. Find **App Sandbox** section
6. Check **Outgoing Connections (Client)** ✅
   - This allows your app to make HTTPS requests to Plaid API

### Alternative: Via Build Settings (if Signing & Capabilities doesn't show)

If you don't see the Signing & Capabilities tab or App Sandbox section:

1. In **Build Settings**, search for: `ENABLE_OUTGOING_NETWORK_CONNECTIONS`
2. Set it to **YES** for both Debug and Release configurations
3. Or manually edit `project.pbxproj`:
   ```
   ENABLE_OUTGOING_NETWORK_CONNECTIONS = YES;
   ```

### For iOS

iOS apps typically don't need special network permissions, but if you encounter issues:

1. In Xcode, select **MoneyFlow** target
2. Go to **Info** tab
3. Add to **Custom iOS Target Properties**:
   - Key: `NSAppTransportSecurity`
   - Type: `Dictionary`
   - Add sub-key: `NSAllowsArbitraryLoads`
   - Value: `YES` (for development only - remove in production)

**Note**: For production, use domain-specific exceptions instead of `NSAllowsArbitraryLoads`.

### Verify

After making changes:
1. Clean build: **⌘⇧K**
2. Rebuild: **⌘B**
3. Try connecting to Plaid again

---

## Quick Check

To verify network permissions are enabled:

1. Open Xcode → MoneyFlow target → Signing & Capabilities
2. Look for **App Sandbox**
3. Verify **Outgoing Connections (Client)** is checked ✅

If it's not there:
- Click **+ Capability** → Add **App Sandbox** (if not already added)
- Then check **Outgoing Connections (Client)**

---

## For Production

In production:
- ✅ Use HTTPS only (Plaid API uses HTTPS)
- ✅ Don't use `NSAllowsArbitraryLoads` on iOS
- ✅ App Sandbox with Outgoing Connections is fine for macOS

