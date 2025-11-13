# Plaid Quick Setup - Step by Step

> **📱 Multi-Platform Support**: This setup works for both iOS and macOS. The Plaid integration supports both platforms with platform-specific Link implementations.

## 🚀 Quick Start (5 Steps)

### Step 1: Get Plaid API Keys
1. Go to https://dashboard.plaid.com/signup
2. Sign up and log in
3. Go to **Team Settings** → **Keys**
4. Copy your **Client ID** and **Sandbox Secret**

---

### Step 2: No SDK Needed! ✅
**Decision**: We're using **WebView for both iOS and macOS**, so no Plaid SDK dependency is needed.

- ✅ Simpler architecture
- ✅ No external dependencies
- ✅ Smaller app size
- ✅ Works on both platforms with same code
- ✅ Automatic updates from Plaid's web Link

**Skip this step** - we'll implement Plaid Link using WebView in Phase 2.

---

### Step 3: Add Client ID to Build Settings
Since your project uses auto-generated Info.plist:

**Method 1: Using Info Tab (Easiest)**
1. In Xcode, select **MoneyFlow** project (top of Project Navigator)
2. Select **MoneyFlow** target
3. Go to **Info** tab
4. Under **Custom iOS Target Properties**, click **+**
5. Key: `PlaidClientId`
6. Type: `String`
7. Value: `client_xxxxxxxxxx` (your actual Client ID)

**Method 2: Using Build Settings**
1. In Xcode, select **MoneyFlow** target
2. Go to **Build Settings** tab
3. Search for: `INFOPLIST_KEY`
4. Find **Info.plist Values** section
5. Click **+** to add new key
6. Key: `INFOPLIST_KEY_PlaidClientId` (note: underscore and prefix)
7. Value: `client_xxxxxxxxxx`
8. The key will be accessible as `PlaidClientId` in code

---

### Step 4: Add Secret as Environment Variable
1. In Xcode: **Product** → **Scheme** → **Edit Scheme...**
2. Select **Run** → **Arguments** tab
3. Under **Environment Variables**, click **+**
4. Name: `PLAID_SECRET`
5. Value: `secret_sandbox_xxxxxxxxxx` (your Sandbox Secret)
6. Click **Close**

---

### Step 5: Build and Test
1. Press **⌘B** to build
2. If you see errors, clean build folder: **⌘⇧K**, then **⌘B** again
3. Verify no "Module 'Plaid' not found" errors

---

## ✅ Verification Checklist

- [ ] `INFOPLIST_KEY_PlaidClientId` is set in Build Settings
- [ ] `PLAID_SECRET` environment variable is set in Scheme
- [ ] Project builds without errors
- [ ] Can see `PlaidConfig.clientId` is not empty (add debug print if needed)
- [x] No SDK needed (using WebView approach)

---

## 🔍 Troubleshooting

**"PlaidClientId not found"**
- Verify key name: `INFOPLIST_KEY_PlaidClientId` (with underscore!)
- Or use `PlaidClientId` in Info tab
- Check target is selected correctly

**"Secret is empty"**
- Check Scheme → Run → Arguments → Environment Variables
- Verify `PLAID_SECRET` is set
- Restart Xcode if needed

---

## 📝 Next Steps

Once setup is complete:
- ✅ SDK is installed
- ✅ Credentials are configured
- ✅ Project builds successfully

**You're ready for Phase 2: Link Integration!** 🎉

---

## 📚 Full Documentation

For detailed explanations, see `PlaidSetupGuide.md`

