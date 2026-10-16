# Building Mac Mouse Fix

This document explains how to build Mac Mouse Fix for personal use.

## Prerequisites

- Xcode installed
- macOS 13.0 or later

## Option 1: Unsigned Build (Quick Test)

Build without code signing for quick testing:

```bash
xcodebuild -project "Mouse Fix.xcodeproj" \
  -scheme "App - Release" \
  -configuration Release \
  -derivedDataPath build_output \
  clean build \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO
```

**Note**: Unsigned builds will **NOT** receive accessibility permissions on modern macOS. The helper will crash in a loop. Use Option 2 for a working build.

## Option 2: Self-Signed Build (Recommended for Personal Use)

For a fully working build that can receive accessibility permissions, you need to sign the app with a self-signed certificate.

### Step 1: Create Self-Signed Certificate (One-Time Setup)

1. Open **Keychain Access** (Applications → Utilities → Keychain Access)
2. Menu: **Keychain Access** → **Certificate Assistant** → **Create a Certificate...**
3. Fill in the form:
   - **Name**: `Mac Mouse Fix Developer`
   - **Identity Type**: `Self-Signed Root`
   - **Certificate Type**: `Code Signing`
   - **✓ Check**: "Let me override defaults"
4. Click **Continue**
5. **Validity Period**: 3650 days (10 years), click **Continue** through remaining screens
6. **Keychain**: Select `login`, click **Create**
7. **Trust the certificate**:
   - Find your new certificate in Keychain Access
   - Double-click it
   - Expand **Trust** section
   - Set **Code Signing** to: `Always Trust`
   - Close (enter password when prompted)

### Step 2: Build Unsigned First

```bash
xcodebuild -project "Mouse Fix.xcodeproj" \
  -scheme "App - Release" \
  -configuration Release \
  -derivedDataPath build_output \
  build \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO
```

### Step 3: Sign All Components

Sign the app components in the correct order (nested frameworks first):

```bash
# Remove any cached signatures
find "build_output/Build/Products/Release/Mac Mouse Fix.app" \
  -name "_CodeSignature" -type d -exec rm -rf {} + 2>/dev/null

# Sign Sparkle framework (auto-update framework)
codesign --force --sign "Mac Mouse Fix Developer" \
  "build_output/Build/Products/Release/Mac Mouse Fix.app/Contents/Frameworks/Sparkle.framework/Versions/A"

# Sign Helper app
codesign --force --sign "Mac Mouse Fix Developer" \
  "build_output/Build/Products/Release/Mac Mouse Fix.app/Contents/Library/LoginItems/Mac Mouse Fix Helper.app"

# Sign main app
codesign --force --sign "Mac Mouse Fix Developer" \
  "build_output/Build/Products/Release/Mac Mouse Fix.app"
```

### Step 4: Verify Signatures

```bash
codesign --verify --deep --strict --verbose=2 \
  "build_output/Build/Products/Release/Mac Mouse Fix.app"
```

You should see: `valid on disk` and `satisfies its Designated Requirement`

### Step 5: Package (Optional)

```bash
ditto -c -k --sequesterRsrc --keepParent \
  "build_output/Build/Products/Release/Mac Mouse Fix.app" \
  "Mac Mouse Fix-Signed.zip"
```

## Running the Signed App

1. Open the app from `build_output/Build/Products/Release/Mac Mouse Fix.app`
2. Click "Enable Mac Mouse Fix"
3. Go to **System Settings** → **Privacy & Security** → **Accessibility**
4. Add and enable **Mac Mouse Fix Helper**
5. The helper should now stay enabled! ✅

## Why Self-Signing is Required

Modern macOS **refuses to grant accessibility permissions** to unsigned applications. Without proper code signing:

- The helper will fail accessibility checks
- The helper will crash and restart in a loop
- The "Enable Mac Mouse Fix" checkbox will auto-disable

Self-signed certificates are **trusted on your Mac only** and allow the app to receive accessibility permissions for personal use.

## Troubleshooting

### App crashes on launch with "Library missing" error

This means the Sparkle framework wasn't signed properly. Re-run Step 3, making sure to sign Sparkle framework first.

### Helper keeps auto-disabling

This means accessibility permissions aren't being granted. Verify:

1. The app is properly signed (run Step 4 verification)
2. You've added the helper to Accessibility in System Settings
3. The certificate is trusted for code signing in Keychain Access

### "Code signature invalid" error

Remove all signatures and re-sign:

```bash
find "build_output/Build/Products/Release/Mac Mouse Fix.app" \
  -name "_CodeSignature" -type d -exec rm -rf {} +
```

Then repeat Step 3.
