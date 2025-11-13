# Plaid Setup Guide - Step by Step Instructions

This guide will walk you through setting up Plaid SDK and configuring credentials for the MoneyFlow app.

## Prerequisites

1. Xcode installed (version 14.0 or later)
2. A Plaid account (sign up at https://dashboard.plaid.com/signup)
3. Access to Plaid Dashboard to get your API keys

---

## Step 1: Get Plaid API Credentials

### 1.1 Create Plaid Account
1. Go to https://dashboard.plaid.com/signup
2. Sign up for a free account
3. Complete the onboarding process

### 1.2 Get Your API Keys
1. Log in to Plaid Dashboard: https://dashboard.plaid.com/
2. Navigate to **Team Settings** → **Keys**
3. You'll find:
   - **Client ID** (starts with `client_`)
   - **Sandbox Secret** (for testing)
   - **Development Secret** (for development)
   - **Production Secret** (for production - only after going live)

### 1.3 Note Your Keys
Write down:
- Client ID: `client_xxxxxxxxxx`
- Sandbox Secret: `secret_sandbox_xxxxxxxxxx`

**⚠️ Important**: Keep these keys secure! Never commit them to version control.

---

## Step 2: Add Plaid SDK to Xcode Project

### 2.1 Open Xcode Project
1. Open `MoneyFlow.xcodeproj` in Xcode
2. Wait for Xcode to finish indexing

### 2.2 Add Package Dependency
1. In Xcode, go to **File** → **Add Package Dependencies...** (or **File** → **Add Packages...** in older Xcode)
2. In the search field, enter: `https://github.com/plaid/plaid-link-ios`
3. Click **Add Package**
4. Select the latest version (or a specific version like `3.0.0`)
5. Ensure **MoneyFlow** target is selected
6. Click **Add Package**

### 2.3 Verify SDK Installation
1. In Xcode, go to **Project Navigator** (left sidebar)
2. Expand **Package Dependencies**
3. You should see `plaid-link-ios` listed
4. If not visible, go to **File** → **Packages** → **Reset Package Caches** and try again

---

## Step 3: Configure Plaid Credentials

### 3.1 Check Info.plist Configuration

Your project uses `GENERATE_INFOPLIST_FILE = YES`, which means Info.plist is auto-generated. We have two options:

**Option A: Add to Build Settings (Recommended)**
1. In Xcode, select the **MoneyFlow** project in Project Navigator
2. Select the **MoneyFlow** target
3. Go to **Build Settings** tab
4. Search for "Info.plist"
5. Find **Info.plist Values** or **INFOPLIST_KEY_**
6. Click **+** to add a new key
7. Key: `PlaidClientId`
8. Value: Your Client ID (e.g., `client_xxxxxxxxxx`)

**Option B: Create Custom Info.plist**
1. In Xcode, right-click on **MoneyFlow** folder → **New File...**
2. Select **Property List** (under iOS → Resource)
3. Name it `Info.plist`
4. Click **Create**
5. In Build Settings, set **Info.plist File** path to `MoneyFlow/Info.plist`
6. Add this entry:
   - Right-click in editor → **Add Row**
   - Key: `PlaidClientId`
   - Type: `String`
   - Value: Your Client ID

**Option C: Add via Build Settings (INFOPLIST_KEY)**
1. In Xcode, select **MoneyFlow** target → **Build Settings**
2. Search for "INFOPLIST_KEY"
3. Add new entry: `INFOPLIST_KEY_PlaidClientId` = `client_xxxxxxxxxx`
4. Or add to existing Info.plist Values section

### 3.2 Verify Configuration
After adding, verify in Xcode:
- The key appears in Build Settings under Info.plist Values
- Or the key exists in your custom Info.plist file

### 3.3 Configure Secret (Environment Variable)

**For Development/Testing:**
1. In Xcode, go to **Product** → **Scheme** → **Edit Scheme...**
2. Select **Run** → **Arguments** tab
3. Under **Environment Variables**, click **+**
4. Name: `PLAID_SECRET`
5. Value: Your Sandbox Secret (e.g., `secret_sandbox_xxxxxxxxxx`)
6. Click **Close**

**For Production:**
- Store the secret in Keychain or secure configuration service
- Never hardcode in source code

### 3.4 Alternative: Hardcode for Testing (Not Recommended for Production)

If you need a quick test setup, you can temporarily modify `PlaidConfig` in `PlaidService.swift`:

```swift
static var secret: String {
    #if DEBUG
    return "secret_sandbox_xxxxxxxxxx"  // Replace with your sandbox secret
    #else
    return ProcessInfo.processInfo.environment["PLAID_SECRET"] ?? ""
    #endif
}
```

**⚠️ Warning**: Remove hardcoded secrets before committing to version control!

---

## Step 4: Verify Configuration

### 4.1 Check PlaidConfig
1. Open `MoneyFlow/Features/Plaid/Services/PlaidService.swift`
2. Verify `PlaidConfig` reads from Info.plist:
   ```swift
   static var clientId: String {
       Bundle.main.object(forInfoDictionaryKey: "PlaidClientId") as? String ?? ""
   }
   ```

### 4.2 Test Configuration (Optional)
Add a temporary test in your app to verify:
```swift
print("Plaid Client ID: \(PlaidConfig.clientId.isEmpty ? "NOT SET" : "SET")")
print("Plaid Environment: \(PlaidConfig.environment)")
```

---

## Step 5: Update .gitignore

Ensure secrets are not committed:

1. Check if `.gitignore` exists in project root
2. Add these entries:
```
# Plaid Secrets
PLAID_SECRET
*.plist.bak
Info.plist.backup
```

3. If you accidentally committed secrets, remove them:
   ```bash
   git filter-branch --force --index-filter \
     "git rm --cached --ignore-unmatch Info.plist" \
     --prune-empty --tag-name-filter cat -- --all
   ```

---

## Step 6: Test Sandbox Environment

### 6.1 Plaid Sandbox Test Credentials
Plaid provides test credentials for sandbox:
- Username: `user_good`
- Password: `pass_good`
- Institution: Use any test institution (e.g., "First Platypus Bank")

### 6.2 Verify Environment
In `PlaidService.swift`, verify:
```swift
static var environment: PlaidEnvironment {
    #if DEBUG
    return .sandbox  // Should be sandbox for testing
    #else
    return .production
    #endif
}
```

---

## Step 7: Build and Test

### 7.1 Build Project
1. In Xcode, select a simulator or device
2. Press **⌘B** (or **Product** → **Build**)
3. Verify no build errors related to Plaid SDK

### 7.2 Common Issues

**Issue: "Module 'Plaid' not found"**
- Solution: Clean build folder (**⌘⇧K**), then rebuild
- Check Package Dependencies is properly linked to target

**Issue: "PlaidClientId not found"**
- Solution: Verify Info.plist has the key
- Check key name matches exactly: `PlaidClientId`

**Issue: "Secret is empty"**
- Solution: Set environment variable in Scheme settings
- Or temporarily hardcode for testing (remove before commit)

---

## Step 8: Ready for Phase 2

Once you've completed these steps:
- ✅ Plaid SDK is added to project
- ✅ Client ID is in Info.plist
- ✅ Secret is configured (via environment variable)
- ✅ Project builds successfully

You're ready to proceed with **Phase 2: Link Integration**!

---

## Quick Reference

### Files Modified
- `Info.plist` - Added `PlaidClientId`
- Xcode Scheme - Added `PLAID_SECRET` environment variable
- `.gitignore` - Added secret exclusions

### Files to Check
- `MoneyFlow/Features/Plaid/Services/PlaidService.swift` - `PlaidConfig`
- Xcode → Package Dependencies - Verify `plaid-link-ios`

### Environment Setup
- **Development**: Uses sandbox environment
- **Production**: Uses production environment (requires production keys)

---

## Next Steps

After completing setup, we'll implement:
1. **PlaidLinkService** - Link SDK integration
2. **PlaidLinkView** - SwiftUI component for Link
3. **PlaidAccountProvider** - Account creation via Link
4. **Update AddAccountView** - Support Plaid provider

Ready when you are! 🚀

