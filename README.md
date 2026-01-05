# Eranin CI

Eranin CI is a centralized repository of **enterprise‑grade, reusable GitHub Actions workflows** for all Eranin projects. It standardizes build, test, security, and release automation using `workflow_call` so teams can adopt consistent CI/CD with minimal per‑repo maintenance.

> **Recommended:** Pin workflows to a **version tag** (e.g., `@v1`) rather than `@main` for stability.

---

## Table of Contents
- [Key Principles](#key-principles)
- [Repository Structure](#repository-structure)
- [Quickstart](#quickstart)
- [Reusable Workflows](#reusable-workflows)
  - [1) Docker Build & Push](#1-docker-build--push)
  - [2) Flutter Mobile Build (Android/iOS)](#2-flutter-mobile-build-androidios)
  - [3) Code Quality (Lint/Test/Coverage)](#3-code-quality-linttestcoverage)
  - [4) Security Scan (SAST/Secrets/IaC)](#4-security-scan-sastsecretsiac)
  - [5) Dependency Audit](#5-dependency-audit)
  - [6) SBOM Generation (CycloneDX)](#6-sbom-generation-cyclonedx)
  - [7) License Scan](#7-license-scan)
  - [8) Release Gate (Policy Checks)](#8-release-gate-policy-checks)
- [Enterprise Standards](#enterprise-standards)
- [Troubleshooting](#troubleshooting)
- [Support](#support)

---

## Key Principles
- **Standardization:** One set of workflows across repos and teams.
- **Security by default:** Minimum permissions, auditable logs, optional OIDC and signing.
- **Deterministic versioning:** Predictable tags and artifacts to simplify rollout and rollback.
- **Composable modules:** Pipelines can be used independently or chained in parent workflows.
- **Governance friendly:** Supports protected environments, approvals, and policy gates.

---

## Repository Structure
```text
.
├── .github/
│   └── workflows/
│       ├── docker_build_push.yml         # Build & push Docker images
│       ├── mobile_flutter_build.yml      # Build Android/iOS artifacts
│       ├── code_quality.yml              # Lint/test/coverage
│       ├── security_scan.yml             # SAST/secrets/IaC scanning
│       ├── dependency_audit.yml          # Dependency vulnerability audit
│       ├── sbom_generate.yml             # SBOM generation (CycloneDX)
│       ├── license_scan.yml              # License compliance scanning
│       └── release_gate.yml              # Enterprise policy gate (branch/env rules)
└── README.md
```

> Some workflows may be added progressively. Versioned tags will reflect the available modules.

---

## Quickstart

### 1) Add organization/repository secrets
Store secrets at **org** level when possible to reduce duplication.

Common secrets:
- `REGISTRY_URL`
- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`
- `ENV_PROD_BASE64` (optional - base64 encoded .env file)
- `ANDROID_GOOGLE_SERVICES_JSON_BASE64` (optional)
- `ANDROID_KEYSTORE_BASE64` (optional)
- `ANDROID_KEY_PROPERTIES_BASE64` (optional)
- `IOS_GOOGLESERVICE_INFO_PLIST_BASE64` (optional)
- `IOS_GOOGLEOAUTH_PLIST_BASE64` (optional)
- `IOS_P12_BASE64` (optional)
- `IOS_P12_PASSWORD` (optional)
- `IOS_MOBILEPROVISION_BASE64` (optional)
- `IOS_KEYCHAIN_PASSWORD` (optional)
- `IOS_EXPORT_OPTIONS_PLIST_BASE64` (optional)

### 2) Use a workflow from Eranin CI
Create a workflow in your project repository and call a reusable workflow via `uses:`.

> Prefer `@v1` (or a semver tag) instead of `@main`:
```yaml
jobs:
  build:
    uses: eranin/eranin-ci/.github/workflows/docker_build_push.yml@v1
```

### 3) Recommended permissions (least privilege)
In the calling workflow:
```yaml
permissions:
  contents: read
  packages: write
  id-token: write   # only if using OIDC signing / cloud auth
```

---

## Reusable Workflows

### Workflow Catalog
| Workflow | File | Primary Use |
|---|---|---|
| Docker Build & Push | `docker_build_push.yml` | Build and push container images |
| Flutter Mobile Build | `mobile_flutter_build.yml` | Build Android APK/AAB and iOS IPA |
| Code Quality | `code_quality.yml` | Lint, tests, coverage, report artifacts |
| Security Scan | `security_scan.yml` | SAST, secret scanning, IaC scanning |
| Dependency Audit | `dependency_audit.yml` | Vulnerable dependency checks |
| SBOM Generation | `sbom_generate.yml` | CycloneDX SBOM artifacts |
| License Scan | `license_scan.yml` | License policy and compliance reporting |
| Release Gate | `release_gate.yml` | Policy enforcement for releases/deployments |

---

## 1) Docker Build & Push

Builds and pushes Docker images to an internal (optionally insecure HTTP) registry using Buildx. Generates a deterministic tag:
- **`YYYYMMDD_<commit_count>`** (e.g., `20251229_128`)

### Usage (calling workflow example)
```yaml
name: Build Backend Image

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  docker:
    uses: eranin/eranin-ci/.github/workflows/docker_build_push.yml@v1
    with:
      image_name: "emoney-be"
      dockerfile: "Dockerfile"
      context: "."
      platforms: "linux/amd64"
    secrets:
      REGISTRY_URL: ${{ secrets.REGISTRY_URL }}
      REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
      REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `image_name` | ✅ | string | — | Docker image name (without registry prefix) |
| `dockerfile` | ❌ | string | `Dockerfile` | Path to Dockerfile |
| `context` | ❌ | string | `.` | Build context |
| `platforms` | ❌ | string | `linux/amd64` | Build target platforms |
| `push_latest` | ❌ | boolean | `true` | Also push the `:latest` tag |
| `allow_insecure_registry` | ❌ | boolean | `true` | Configure Docker/BuildKit for HTTP registry |

### Secrets
| Name | Required | Description |
|---|---:|---|
| `REGISTRY_URL` | ✅ | Registry host (and port if needed), e.g., `registry.internal:5000` |
| `REGISTRY_USERNAME` | ✅ | Registry username |
| `REGISTRY_PASSWORD` | ✅ | Registry password/token |

### Outputs
| Name | Description |
|---|---|
| `image_tag` | Generated tag (e.g., `YYYYMMDD_<commit_count>`) |
| `full_image` | Fully qualified image (e.g., `registry/internal/emoney-be:TAG`) |

### Notes (enterprise guidance)
- Pin workflow version: `@v1` (or semver tag).
- Enable environment protection in deployment workflows that consume the image.
- Consider adding image signing + provenance (SLSA) for production pipelines.

---

## 2) Flutter Mobile Build (Android/iOS)

Builds Flutter mobile artifacts with full support for environment injection, Firebase configuration, and code signing:
- Android: **APK** or **AAB** (with optional keystore signing)
- iOS: **IPA** (with optional P12 certificate + provisioning profile signing)

### Features
- **Deterministic versioning:** `YYYYMMDD.<commit_count>` format
- **Environment injection:** Base64-encoded `.env` file support
- **Firebase integration:** Automatic injection of `google-services.json` and `GoogleService-Info.plist`
- **Google OAuth:** Optional `GoogleOAuth.plist` injection for iOS
- **Android signing:** Keystore + key.properties injection for release builds
- **iOS signing:** P12 certificate + provisioning profile + ExportOptions.plist support
- **Flexible export:** Supports `app-store`, `ad-hoc`, and `development` export methods

### Usage (Android only - minimal)
```yaml
name: Build Mobile (Android)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  mobile:
    uses: eranin/eranin-ci/.github/workflows/mobile_flutter_build.yml@v1
    with:
      platform: "android"
      flutter_version: "3.19.6"
      build_type: "release"
      android_output: "apk"
      working_directory: "."
```

### Usage (Android with signing + Firebase)
```yaml
name: Build Mobile (Android - Signed)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  mobile:
    uses: eranin/eranin-ci/.github/workflows/mobile_flutter_build.yml@v1
    with:
      platform: "android"
      flutter_version: "3.19.6"
      build_type: "release"
      android_output: "appbundle"
      working_directory: "."
      dotenv_target_path: ".env"
    secrets:
      ENV_PROD_BASE64: ${{ secrets.ENV_PROD_BASE64 }}
      ANDROID_GOOGLE_SERVICES_JSON_BASE64: ${{ secrets.ANDROID_GOOGLE_SERVICES_JSON_BASE64 }}
      ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
      ANDROID_KEY_PROPERTIES_BASE64: ${{ secrets.ANDROID_KEY_PROPERTIES_BASE64 }}
```

### Usage (iOS only - no codesign)
```yaml
name: Build Mobile (iOS - No Sign)

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  mobile:
    uses: eranin/eranin-ci/.github/workflows/mobile_flutter_build.yml@v1
    with:
      platform: "ios"
      flutter_version: "3.19.6"
      build_type: "release"
      ios_codesign: false
      working_directory: "."
```

### Usage (iOS with full signing)
```yaml
name: Build Mobile (iOS - Signed)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  mobile:
    uses: eranin/eranin-ci/.github/workflows/mobile_flutter_build.yml@v1
    with:
      platform: "ios"
      flutter_version: "3.19.6"
      build_type: "release"
      ios_codesign: true
      ios_bundle_id: "com.eranin.myapp"
      ios_export_method: "app-store"
      working_directory: "."
      dotenv_target_path: ".env"
    secrets:
      ENV_PROD_BASE64: ${{ secrets.ENV_PROD_BASE64 }}
      IOS_GOOGLESERVICE_INFO_PLIST_BASE64: ${{ secrets.IOS_GOOGLESERVICE_INFO_PLIST_BASE64 }}
      IOS_GOOGLEOAUTH_PLIST_BASE64: ${{ secrets.IOS_GOOGLEOAUTH_PLIST_BASE64 }}
      IOS_P12_BASE64: ${{ secrets.IOS_P12_BASE64 }}
      IOS_P12_PASSWORD: ${{ secrets.IOS_P12_PASSWORD }}
      IOS_MOBILEPROVISION_BASE64: ${{ secrets.IOS_MOBILEPROVISION_BASE64 }}
      IOS_KEYCHAIN_PASSWORD: ${{ secrets.IOS_KEYCHAIN_PASSWORD }}
      IOS_EXPORT_OPTIONS_PLIST_BASE64: ${{ secrets.IOS_EXPORT_OPTIONS_PLIST_BASE64 }}
```

### Usage (Android + iOS)
```yaml
name: Build Mobile (Both Platforms)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  mobile:
    uses: eranin/eranin-ci/.github/workflows/mobile_flutter_build.yml@v1
    with:
      platform: "both"
      flutter_version: "3.19.6"
      build_type: "release"
      android_output: "appbundle"
      ios_codesign: true
      ios_bundle_id: "com.eranin.myapp"
      ios_export_method: "app-store"
      working_directory: "."
      dotenv_target_path: ".env"
    secrets:
      ENV_PROD_BASE64: ${{ secrets.ENV_PROD_BASE64 }}
      ANDROID_GOOGLE_SERVICES_JSON_BASE64: ${{ secrets.ANDROID_GOOGLE_SERVICES_JSON_BASE64 }}
      ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
      ANDROID_KEY_PROPERTIES_BASE64: ${{ secrets.ANDROID_KEY_PROPERTIES_BASE64 }}
      IOS_GOOGLESERVICE_INFO_PLIST_BASE64: ${{ secrets.IOS_GOOGLESERVICE_INFO_PLIST_BASE64 }}
      IOS_GOOGLEOAUTH_PLIST_BASE64: ${{ secrets.IOS_GOOGLEOAUTH_PLIST_BASE64 }}
      IOS_P12_BASE64: ${{ secrets.IOS_P12_BASE64 }}
      IOS_P12_PASSWORD: ${{ secrets.IOS_P12_PASSWORD }}
      IOS_MOBILEPROVISION_BASE64: ${{ secrets.IOS_MOBILEPROVISION_BASE64 }}
      IOS_KEYCHAIN_PASSWORD: ${{ secrets.IOS_KEYCHAIN_PASSWORD }}
      IOS_EXPORT_OPTIONS_PLIST_BASE64: ${{ secrets.IOS_EXPORT_OPTIONS_PLIST_BASE64 }}
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `flutter_version` | ❌ | string | `3.19.6` | Flutter SDK version |
| `platform` | ❌ | string | `android` | Target platform: `android` \| `ios` \| `both` |
| `build_type` | ❌ | string | `release` | Build type: `debug` \| `release` |
| `android_output` | ❌ | string | `apk` | Android output format: `apk` \| `appbundle` |
| `working_directory` | ❌ | string | `.` | App directory (monorepo support) |
| `dotenv_target_path` | ❌ | string | `.env` | Target path for decoded .env file |
| `ios_codesign` | ❌ | boolean | `false` | Enable iOS code signing |
| `ios_bundle_id` | ❌ | string | `com.eranin.etask` | iOS bundle identifier (for ExportOptions.plist generation) |
| `ios_export_method` | ❌ | string | `app-store` | iOS export method: `app-store` \| `ad-hoc` \| `development` |

### Secrets
| Name | Required | Description |
|---|---:|---|
| `ENV_PROD_BASE64` | ❌ | Base64-encoded production `.env` file |
| `ANDROID_GOOGLE_SERVICES_JSON_BASE64` | ❌ | Base64-encoded `google-services.json` for Firebase Android |
| `IOS_GOOGLESERVICE_INFO_PLIST_BASE64` | ❌ | Base64-encoded `GoogleService-Info.plist` for Firebase iOS |
| `IOS_GOOGLEOAUTH_PLIST_BASE64` | ❌ | Base64-encoded `GoogleOAuth.plist` for Google Sign-In iOS |
| `ANDROID_KEYSTORE_BASE64` | ❌ | Base64-encoded Android keystore (`.jks` file) |
| `ANDROID_KEY_PROPERTIES_BASE64` | ❌ | Base64-encoded `key.properties` file |
| `IOS_P12_BASE64` | ❌ | Base64-encoded iOS signing certificate (`.p12` file) |
| `IOS_P12_PASSWORD` | ❌ | Password for the P12 certificate |
| `IOS_MOBILEPROVISION_BASE64` | ❌ | Base64-encoded iOS provisioning profile |
| `IOS_KEYCHAIN_PASSWORD` | ❌ | Password for the temporary build keychain |
| `IOS_EXPORT_OPTIONS_PLIST_BASE64` | ❌ | Base64-encoded custom `ExportOptions.plist` (optional, auto-generated if not provided) |

### Outputs
| Name | Description |
|---|---|
| `build_version` | Deterministic build version (e.g., `20251229.128`) |

### Artifacts
Artifacts are uploaded with names including the build version:
- **Android:** `android-<version>` containing `**/build/app/outputs/**/*.apk` and/or `**/build/app/outputs/**/*.aab`
- **iOS:** `ios-<version>` containing `**/build/ios/ipa/*.ipa`

### How Signing Works

#### Android Signing (Release Only)
When `build_type: release` and both `ANDROID_KEYSTORE_BASE64` and `ANDROID_KEY_PROPERTIES_BASE64` are provided:
1. `key.properties` is written to `android/key.properties`
2. Keystore is written to both `android/app/key.jks` and `android/key.jks` (for compatibility)
3. Your `android/app/build.gradle` should reference these files for signing

#### iOS Signing (Release Only)
When `ios_codesign: true`, `build_type: release`, and all required iOS secrets are provided:
1. A temporary keychain (`build.keychain-db`) is created
2. P12 certificate is imported into the keychain
3. Provisioning profile is installed to `~/Library/MobileDevice/Provisioning Profiles/`
4. If `IOS_EXPORT_OPTIONS_PLIST_BASE64` is provided, it's used directly
5. Otherwise, a minimal `ExportOptions.plist` is auto-generated using `ios_bundle_id` and `ios_export_method`
6. Build runs with `flutter build ipa --release --export-options-plist=ios/ExportOptions.plist`

If iOS signing is not configured, the build runs with `--no-codesign` flag.

### Preparing Secrets

#### Encode files to Base64
```bash
# .env file
base64 -i .env.prod -o env_prod_base64.txt

# Android
base64 -i google-services.json -o google_services_base64.txt
base64 -i key.jks -o keystore_base64.txt
base64 -i key.properties -o key_properties_base64.txt

# iOS
base64 -i GoogleService-Info.plist -o google_service_info_base64.txt
base64 -i GoogleOAuth.plist -o google_oauth_base64.txt
base64 -i certificate.p12 -o p12_base64.txt
base64 -i profile.mobileprovision -o mobileprovision_base64.txt
base64 -i ExportOptions.plist -o export_options_base64.txt
```

#### Example `key.properties` (Android)
```properties
storePassword=your_store_password
keyPassword=your_key_password
keyAlias=your_key_alias
storeFile=key.jks
```

---

## 3) Code Quality (Lint/Test/Coverage)

Runs language‑specific linting and tests, produces coverage, and uploads artifacts. Designed for mono‑repo support via `working_directory`.

### Usage (Node.js example)
```yaml
name: Code Quality

on:
  pull_request:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  quality:
    uses: eranin/eranin-ci/.github/workflows/code_quality.yml@v1
    with:
      runtime: "node"
      node_version: "20"
      working_directory: "."
      run_lint: true
      run_tests: true
      run_coverage: true
```

### Usage (Python example)
```yaml
jobs:
  quality:
    uses: eranin/eranin-ci/.github/workflows/code_quality.yml@v1
    with:
      runtime: "python"
      python_version: "3.11"
      working_directory: "."
      run_lint: true
      run_tests: true
      run_coverage: true
```

### Inputs (common)
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `runtime` | ✅ | string | — | `node` \| `python` \| `go` \| `java` \| `dotnet` |
| `working_directory` | ❌ | string | `.` | Project subfolder |
| `run_lint` | ❌ | boolean | `true` | Run lint |
| `run_tests` | ❌ | boolean | `true` | Run tests |
| `run_coverage` | ❌ | boolean | `false` | Generate coverage reports |

### Outputs
| Name | Description |
|---|---|
| `quality_status` | `pass` or `fail` |
| `coverage_percent` | Coverage value when enabled |

---

## 4) Security Scan (SAST/Secrets/IaC)

Runs a standardized security suite that can include:
- SAST (CodeQL or Semgrep)
- Secret scanning (gitleaks)
- IaC scanning (tfsec/checkov)

### Usage
```yaml
name: Security Scan

on:
  pull_request:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  security-events: write

jobs:
  security:
    uses: eranin/eranin-ci/.github/workflows/security_scan.yml@v1
    with:
      enable_codeql: true
      enable_gitleaks: true
      enable_iac_scan: true
      working_directory: "."
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `enable_codeql` | ❌ | boolean | `true` | Enable CodeQL scanning |
| `enable_gitleaks` | ❌ | boolean | `true` | Enable secret scanning |
| `enable_iac_scan` | ❌ | boolean | `true` | Enable IaC scanning |
| `working_directory` | ❌ | string | `.` | Repo directory |

### Outputs
| Name | Description |
|---|---|
| `security_status` | `pass` / `fail` |
| `findings_summary` | High‑level summary |

---

## 5) Dependency Audit

Audits dependencies for known vulnerabilities (e.g., npm audit, pip-audit, osv-scanner).

### Usage
```yaml
name: Dependency Audit

on:
  pull_request:

permissions:
  contents: read

jobs:
  deps:
    uses: eranin/eranin-ci/.github/workflows/dependency_audit.yml@v1
    with:
      ecosystem: "node"         # node | python | go | java
      working_directory: "."
      fail_on_severity: "high"  # low | medium | high | critical
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `ecosystem` | ✅ | string | — | `node` \| `python` \| `go` \| `java` |
| `fail_on_severity` | ❌ | string | `high` | Fail threshold |
| `working_directory` | ❌ | string | `.` | Repo directory |

### Outputs
| Name | Description |
|---|---|
| `audit_status` | `pass` / `fail` |
| `vuln_count` | Vulnerability count |

---

## 6) SBOM Generation (CycloneDX)

Generates and uploads SBOM artifacts, enabling supply‑chain traceability.

### Usage
```yaml
name: SBOM

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read

jobs:
  sbom:
    uses: eranin/eranin-ci/.github/workflows/sbom_generate.yml@v1
    with:
      ecosystem: "node"
      working_directory: "."
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `ecosystem` | ✅ | string | — | `node` \| `python` \| `go` \| `java` \| `docker` |
| `working_directory` | ❌ | string | `.` | Repo directory |

### Outputs
| Name | Description |
|---|---|
| `sbom_path` | Path to generated SBOM artifact |

---

## 7) License Scan

Scans dependencies for license compliance against allowed/denied policies.

### Usage
```yaml
name: License Scan

on:
  pull_request:

permissions:
  contents: read

jobs:
  license:
    uses: eranin/eranin-ci/.github/workflows/license_scan.yml@v1
    with:
      ecosystem: "node"
      policy: "enterprise-default"   # customizable policy profile
      working_directory: "."
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `ecosystem` | ✅ | string | — | `node` \| `python` \| `go` \| `java` |
| `policy` | ❌ | string | `enterprise-default` | License policy profile |
| `working_directory` | ❌ | string | `.` | Repo directory |

### Outputs
| Name | Description |
|---|---|
| `license_status` | `pass` / `fail` |
| `violations` | Number of policy violations |

---

## 8) Release Gate (Policy Checks)

Applies enterprise release policies before deployment:
- Branch protection alignment
- Required checks passed
- Tag naming conventions
- Environment approvals (optional)
- Change ticket reference (optional)

### Usage (pre-deploy gate)
```yaml
name: Release Gate

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  gate:
    uses: eranin/eranin-ci/.github/workflows/release_gate.yml@v1
    with:
      require_main_branch: true
      require_tag: true
      tag_pattern: "^v\\d+\\.\\d+\\.\\d+$"
      require_change_ticket: false
```

### Inputs
| Name | Required | Type | Default | Description |
|---|---:|---|---|---|
| `require_main_branch` | ❌ | boolean | `true` | Only allow from main |
| `require_tag` | ❌ | boolean | `true` | Require a version tag |
| `tag_pattern` | ❌ | string | `^v\d+\.\d+\.\d+$` | Release tag regex |
| `require_change_ticket` | ❌ | boolean | `false` | Require change ticket in metadata |

### Outputs
| Name | Description |
|---|---|
| `gate_status` | `pass` / `fail` |

---

## Enterprise Standards

### Versioning & Stability
- Use `@v1` tags (or semver) for workflow references.
- Maintain changelog and release notes per workflow version.

### Least‑Privilege Permissions
- Default to `contents: read`.
- Only enable `packages: write`, `security-events: write`, or `id-token: write` when required.

### Environment Protection
- Use GitHub Environments for prod deployments:
  - Approval rules
  - Required reviewers
  - Wait timers
  - Secrets scoping
  - Deployment history

### Supply Chain Security
Recommended (production):
- SBOM generation for all releases.
- Image signing (Cosign) and provenance (SLSA) for container builds.
- Dependency audits with strict fail thresholds.

### Auditing & Compliance
- Store artifacts (SBOM, scan reports, coverage reports) for audit retention.
- Enforce policy gates for regulated workloads.

---

## Troubleshooting

### Docker push: "Access Denied"
- Verify:
  - `REGISTRY_URL` (host/port correct)
  - HTTP vs HTTPS mismatch
  - Credentials validity and permissions

### Buildx fails to start
- Ensure BuildKit is available on runner; try removing existing builder:
  - `docker buildx rm builder || true`
- Confirm registry settings when using HTTP/insecure registry.

### Shallow clone prevents deterministic tags
- Ensure `actions/checkout` uses `fetch-depth: 0`.

### iOS build fails
- iOS requires `macos-latest` runner.
- Ensure `flutter build ipa` is supported by your Flutter version.
- Verify all iOS signing secrets are properly base64-encoded.
- Check that `ios_bundle_id` matches your provisioning profile.
- If using a custom `ExportOptions.plist`, ensure the provisioning profile UUID is correct.

### Android signing fails
- Ensure `key.properties` references the correct keystore path (`key.jks`).
- Verify `storeFile` in `key.properties` matches the keystore filename.
- Check that your `android/app/build.gradle` is configured to read signing config from `key.properties`.

### .env file not found
- Verify `ENV_PROD_BASE64` secret is set and properly base64-encoded.
- Check `dotenv_target_path` matches where your app expects the `.env` file.

### Firebase configuration missing
- Ensure the appropriate `*_BASE64` secrets are set.
- For Android: `ANDROID_GOOGLE_SERVICES_JSON_BASE64`
- For iOS: `IOS_GOOGLESERVICE_INFO_PLIST_BASE64` and optionally `IOS_GOOGLEOAUTH_PLIST_BASE64`

---

## Support
For support, contact the **Eranin DevOps Team**.
