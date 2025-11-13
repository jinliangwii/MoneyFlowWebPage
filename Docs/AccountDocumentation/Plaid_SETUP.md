# Plaid SDK Setup Instructions

## Adding Plaid SDK Dependency

The Plaid iOS SDK needs to be added to the Xcode project. There are two options:

### Option 1: Swift Package Manager (Recommended)

1. Open Xcode project
2. Go to **File → Add Package Dependencies...**
3. Enter the URL: `https://github.com/plaid/plaid-link-ios`
4. Select the version (latest stable release)
5. Add to the **MoneyFlow** target

### Option 2: CocoaPods

1. Add to `Podfile`:
   ```ruby
   pod 'Plaid', '~> 3.0'
   ```
2. Run `pod install`
3. Open workspace (not project)

## Configuration

After adding the SDK, configure Plaid credentials:

### 1. Add to Info.plist

Add your Plaid Client ID:
```xml
<key>PlaidClientId</key>
<string>YOUR_CLIENT_ID</string>
```

### 2. Set Environment Variable

For development, set the Plaid secret as an environment variable:
```bash
export PLAID_SECRET=your_secret_here
```

**⚠️ Important**: Never commit secrets to version control!

### 3. Production Setup

For production, store the secret securely:
- Use Keychain for local storage
- Or use a secure configuration service
- Never hardcode in source code

## Testing

Use Plaid's sandbox environment for testing:
- Environment is automatically set to `.sandbox` in DEBUG builds
- Use test credentials from Plaid Dashboard
- Test accounts: `user_good`, `user_login_required`, etc.

## Next Steps

Once SDK is added:
1. Import Plaid SDK in files that need it
2. Continue with Phase 2 (Link Integration)
3. Test with sandbox environment

