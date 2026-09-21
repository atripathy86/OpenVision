# OpenVision TestFlight pipeline

This project can build and upload a signed OpenVision release to TestFlight
from GitHub Actions. The workflow uses XcodeGen, fastlane `match`, and an App
Store Connect API key; no Apple ID password or local Mac is required for CI.

## How the pipeline works

1. GitHub Actions creates the ignored `Config.xcconfig` and `Config.swift`
   files from repository secrets.
2. XcodeGen generates `OpenVision.xcodeproj` from `project.yml`.
3. fastlane `match` fetches or creates the encrypted App Store distribution
   certificate and provisioning profile on the `match-certs` branch.
4. fastlane archives the `OpenVision` scheme with manual Release signing.
5. The resulting IPA is uploaded to TestFlight using the App Store Connect API.

## One-time Apple setup

1. Confirm an active Apple Developer Program membership and note the Team ID.
2. Register at least one test device at [Apple Developer devices](https://developer.apple.com/account/resources/devices/list).
3. Create an App Store Connect API key with App Manager access. Download the
   `.p8` file immediately and record its Key ID and Issuer ID.
4. In the Apple Developer portal (Identifiers), register the bundle ID and enable the
   **Increased Memory Limit** capability. `OpenVision.entitlements` requests it, and `match`
   does not enable capabilities, so a profile without it fails at archive time.
5. Create the OpenVision app in App Store Connect (My Apps → + → New App) using the exact
   bundle ID stored in `PRODUCT_BUNDLE_IDENTIFIER`. This must be done by hand: fastlane's
   `produce` only authenticates with an Apple ID, not an App Store Connect API key, so it
   cannot create the app from CI.

## GitHub repository secrets

Add these under **Settings → Secrets and variables → Actions**:

| Secret | Value |
| --- | --- |
| `DEVELOPMENT_TEAM` | Apple Team ID |
| `PRODUCT_BUNDLE_IDENTIFIER` | The registered OpenVision bundle ID |
| `META_APP_ID` | Meta Developer App ID |
| `CLIENT_TOKEN` | Full Meta client token (`AR|...`) |
| `APP_LINK_URL_SCHEME` | Meta callback URL scheme |
| `APP_STORE_CONNECT_API_KEY_ID` | App Store Connect API Key ID |
| `APP_STORE_CONNECT_API_ISSUER_ID` | App Store Connect Issuer ID |
| `APP_STORE_CONNECT_API_KEY_CONTENT` | Base64-encoded `.p8` file |
| `MATCH_PASSWORD` | Strong password used to encrypt signing assets |

Encode the API key locally with:

```bash
base64 -i AuthKey_XXXXXXXXXX.p8 | tr -d '\n'
```

Treat the API key, Meta client token, and match password as credentials. Do
not commit `Config.xcconfig`, `Config.swift`, or the `.p8` file.

## Running a build

1. Enable GitHub Actions for the repository.
2. Open **Actions → TestFlight → Run workflow**.
3. Make sure the app already exists in App Store Connect (see the one-time setup).

The workflow assigns the GitHub run number as the build number and uses the
marketing version in `project.yml`. TestFlight processing may take additional
time after the workflow succeeds.

## Signing storage

`match` stores encrypted signing assets in the private `match-certs` branch of
the repository the workflow runs in (`GITHUB_REPOSITORY`, so your fork). The workflow has `contents: write` permission because the
first signing run may need to create or update that branch. Keep the branch
protected as appropriate for the repository and never expose its decryption
password.

## Local release troubleshooting

For local development, continue using the existing `Config.xcconfig` and
automatic signing flow. For CI-specific build failures, inspect the uploaded
fastlane/gym logs. OpenVision’s MLX packages require the workflow’s
`-skipMacroValidation` and `-skipPackagePluginValidation` flags.
