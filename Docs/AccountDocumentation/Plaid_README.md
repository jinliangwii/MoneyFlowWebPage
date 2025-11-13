# Plaid Feature Module

This module provides integration with Plaid for importing bank transactions from U.S. banks.

**📱 Multi-Platform Support**: Fully supports both iOS and macOS with platform-specific Link implementations.

## Architecture

The Plaid integration follows the same AccountFeatureProvider pattern as CMB:
- **Provider**: `PlaidAccountProvider` and `PlaidAccountFeatureProvider`
- **Services**: `PlaidService`, `PlaidLinkService`, `PlaidTransactionSyncService`
- **Data**: `PlaidRepository`, `PlaidModels`
- **Domain**: `PlaidImportParameters`

## Key Components

### PlaidService
Core API service for Plaid API interactions:
- `exchangePublicToken()` - Exchange public token for access token
- `fetchAccounts()` - Get accounts for an access token
- `fetchTransactions()` - Fetch transactions for date range
- `refreshAccessToken()` - Refresh expired tokens

### PlaidTokenManager
Secure Keychain storage for access tokens:
- Stores tokens securely in iOS Keychain
- One token per Plaid item (can have multiple accounts)
- Automatic token encryption

### PlaidModels
Data models for Plaid API responses:
- `PlaidAccessToken` - Access token with item info
- `PlaidAccount` - Account information
- `PlaidTransaction` - Transaction data
- `PlaidBalances` - Account balances

## Configuration

Plaid configuration is handled via `PlaidConfig`:
- `clientId`: From Info.plist or environment
- `secret`: From secure storage (Keychain) or environment variable
- `environment`: Sandbox (debug) or Production (release)

## Multi-Account Support

Each Plaid account becomes an individual `Account` record:
- Accounts appear in existing account list (no special grouping)
- Accounts respect existing account type groups
- When syncing, all accounts from same Plaid item sync together (efficient)

## Platform Support

### iOS & macOS (Both Use WebView)
- ✅ Uses Plaid Link in WebView (web-based Link flow)
- ✅ Same implementation for both platforms
- ✅ Consistent user experience
- ✅ Single codebase to maintain
- ✅ No SDK dependency needed

**Key Insight**: We use WebView for both platforms for simplicity and consistency. All core API calls, token management, and transaction sync are platform-agnostic and shared.

## Setup

1. ~~Add Plaid SDK dependency~~ ✅ **NOT NEEDED** - Using WebView for both platforms
2. Configure `PlaidConfig` with your client ID and secret
3. Set up Plaid account and get API keys
4. Register `PlaidAccountFeatureProvider` in `AccountFeatureRegistry`
5. Implement WebView-based Link in Phase 2 (works on both iOS and macOS)

## Status

**Phase 1**: Foundation ✅ (Complete - Platform-Agnostic)
- ✅ Folder structure
- ✅ Account model metadata extension
- ✅ PlaidModels (platform-agnostic)
- ✅ PlaidTokenManager (platform-agnostic)
- ✅ PlaidService (platform-agnostic)
- ✅ PlaidError enum (platform-agnostic)
- ✅ No SDK needed (using WebView approach)

**Phase 2**: Link Integration (Pending - WebView for Both)
- ⏳ PlaidLinkServiceProtocol
- ⏳ PlaidLinkService (WebView-based, unified)
- ⏳ PlaidLinkView (WebView with platform-specific wrappers)
- ⏳ PostMessage event handling

**Phase 3**: Transaction Sync (Pending - Platform-Agnostic)
**Phase 4**: Polish & Testing (Pending - Both Platforms)

## Documentation

- **Full Multi-Platform Architecture**: `Docs/PlaidIntegrationArchitecture_MultiPlatform.md`
- **Quick Summary**: `Docs/PlaidMultiPlatformSummary.md`
- **Setup Guide**: `Docs/PlaidQuickSetup.md`

