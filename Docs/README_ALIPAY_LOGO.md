# Account Logo Setup Guide

This guide explains how to add custom logos for account institutions (Alipay, CMB, Plaid, etc.).

## General Logo Conversion Script

We have a generalized script `convert_logo.py` that can convert any logo image to iOS asset sizes.

### Usage

```bash
cd /Users/jinliangwei/Developer/MoneyFlow
python3 convert_logo.py <logo_name> <input_image> [base_size]
```

**Parameters:**
- `logo_name`: Name of the logo (e.g., "AlipayLogo", "CMBLogo", "PlaidLogo")
- `input_image`: Path to your logo image file
- `base_size`: Base size in points (default: 44)

**Examples:**
```bash
# Alipay logo
python3 convert_logo.py AlipayLogo ~/Downloads/alipay_logo.png 44

# CMB logo
python3 convert_logo.py CMBLogo ~/Downloads/cmb_logo.png 44

# Plaid logo (future)
python3 convert_logo.py PlaidLogo ~/Downloads/plaid_logo.png 44
```

### What the Script Does

1. **Creates the imageset directory** if it doesn't exist
2. **Generates Contents.json** with proper iOS asset catalog structure
3. **Converts the image** to three sizes:
   - `{LogoName}.png` (1x - 44x44 pixels by default)
   - `{LogoName}@2x.png` (2x - 88x88 pixels)
   - `{LogoName}@3x.png` (3x - 132x132 pixels)
4. **Saves all files** to `MoneyFlow/Assets.xcassets/{LogoName}.imageset/`

### After Running the Script

1. **Add the logo name to `InstitutionIcons.swift`:**
   - Add `"{LogoName}"` to the `customLogoIdentifiers` array
   - Add detection logic in `symbol()` function
   - Add color in `color()` function (optional)

2. **Rebuild the app** in Xcode

## Alipay Logo Setup

### Current Status
- ✅ Code is configured to use `AlipayLogo` 
- ✅ Asset catalog structure created (`AlipayLogo.imageset`)
- ✅ Logo image files created

### Option 2: Manual Addition

1. **Create the three image sizes:**
   - 1x: 44x44 pixels → Save as `AlipayLogo.png`
   - 2x: 88x88 pixels → Save as `AlipayLogo@2x.png`
   - 3x: 132x132 pixels → Save as `AlipayLogo@3x.png`

2. **Add them to Xcode:**
   - Open `MoneyFlow.xcodeproj` in Xcode
   - Navigate to `MoneyFlow/Assets.xcassets/AlipayLogo.imageset`
   - Drag and drop the three PNG files into the imageset

## Fallback Behavior

If the logo images are not found, the app will automatically fall back to using the SF Symbol `creditcard.triangle` with Alipay blue color. The app will still function correctly, but won't show the custom logo.

## Verification

After adding the logo, rebuild the app. The Alipay logo should appear:
- In the account list (left sidebar)
- In account detail headers
- In the account provider selection screen

