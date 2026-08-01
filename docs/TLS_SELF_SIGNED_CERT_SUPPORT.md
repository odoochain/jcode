# Named Provider Profile: TLS Self-Signed Cert Support & Auth-Test Fixes

## Date: 2026-08-02

## Overview

Implemented per-provider `accept_invalid_certs` config field for named provider profiles, allowing jcode to connect to endpoints using self-signed certificates (e.g., internal gateways, development servers). Also fixed two pre-existing bugs in `auth-test` for named provider profiles.

## Bugs Fixed

### Bug 1: credential_probe reports `not_configured` for named profiles

**Root Cause:** `apply_login_provider_profile_env()` in `src/cli/provider_init.rs:1219` unconditionally overwrites named profile env vars with built-in OpenAI-compatible defaults when `--provider-profile` is active.

**Fix:** Added early return guard:
```rust
// src/cli/provider_init.rs ~line 1219
if std::env::var_os("JCODE_NAMED_PROVIDER_PROFILE").is_some() {
    return;
}
```

### Bug 2: provider_smoke hits wrong base_url (`https://api.openai.com/v1/models`)

**Root Cause:** `discover_openai_compatible_validation_model()` in `src/cli/auth_test/choice.rs:149` used the built-in profile's hardcoded `api_base` instead of the user-configured `base_url`.

**Fix:** Read `JCODE_OPENROUTER_API_BASE` env var (set by dispatch.rs for named profiles) as override:
```rust
let api_base = std::env::var("JCODE_OPENROUTER_API_BASE")
    .unwrap_or_else(|_| profile.api_base.clone());
```

## Feature: accept_invalid_certs Config Field

### Problem

Workbuddy endpoint (`https://myapi.zenx.tech:8001`) uses a self-signed certificate where the CA cert is used as the leaf certificate, causing rustls error: `CaUsedAsEndEntity`. Standard TLS verification rejects this.

### Solution Architecture

Three-layer approach:

1. **Config layer** — New optional boolean field on `NamedProviderConfig`
2. **Runtime layer** — Conditional HTTP client selection in `OpenRouterProvider::new_named_openai_compatible()`
3. **Auth-test layer** — Direct HTTP probe also respects the flag via env var

### Files Modified

| File | Line(s) | Change |
|------|---------|--------|
| `crates/jcode-config-types/src/lib.rs` | 473 | Added `accept_invalid_certs: Option<bool>` field with serde attributes |
| `crates/jcode-provider-core/src/lib.rs` | 674-692 | Added `insecure_http_client()` function returning shared reqwest client with `.danger_accept_invalid_certs(true)` and `.danger_accept_invalid_hostnames(true)` |
| `crates/jcode-base/src/provider/mod.rs` | 64 | Re-exported `insecure_http_client` from `jcode_provider_core` |
| `crates/jcode-provider-openrouter-runtime/src/lib.rs` | 1460-1464 | Conditional client selection: uses `insecure_http_client()` when `accept_invalid_certs == Some(true)`, otherwise `shared_http_client()` |
| `crates/jcode-base/src/provider_catalog.rs` | 701-705 | Sets/removes `JCODE_ACCEPT_INVALID_CERTS` env var based on profile config during `apply_named_provider_profile_env_from_config()` |
| `src/cli/auth_test/choice.rs` | 152-157 | Auth-test smoke probe checks `JCODE_ACCEPT_INVALID_CERTS` env var to select insecure client |
| `src/cli/commands/provider_setup.rs` | 185 | Added `accept_invalid_certs: None` to struct literal init |

### User Configuration

Added to `~/.jcode/config.toml` under `[providers.workbuddy]`:

```toml
accept_invalid_certs = true
```

## Verification Results

All 4 auth-test checks pass for workbuddy profile:

```
✓ credential_probe — API key available
✓ refresh_probe — Skipped (not applicable)
✓ provider_smoke — AUTH_TEST_OK
✓ tool_smoke — AUTH_TEST_OK
result: PASS
```

Normal model calls also verified working:

```
$ jcode --provider-profile workbuddy --model glm-5.2 run 'say hi'
Hi! 👋
[Tokens] upload: 10976 download: 5
```

## Technical Notes

- **OnceLock pattern**: Both `shared_http_client()` and `insecure_http_client()` use `OnceLock` for process-wide singleton clients. Cannot have per-request TLS settings — must choose at construction time.
- **Env var bridge**: `JCODE_ACCEPT_INVALID_CERTS` serves as the bridge between config loading (provider_catalog) and HTTP client selection (auth_test choice.rs direct probe), since the auth-test smoke probe doesn't go through the runtime's `new_named_openai_compatible()`.
- **Re-export chain**: `jcode_provider_core::insecure_http_client()` → re-exported by `jcode_base::provider` → accessible as `crate::provider::insecure_http_client()` in CLI code.
- **Scoop shim caveat**: User has jcode installed via Scoop at `/d/programs/Scoop/shims/jcode`. After `scripts/install_release.sh`, must manually copy release-lto binary to Scoop shim location for changes to take effect in shell.

## Pre-existing Test Failures (NOT caused by these changes)

- `cli::acp::tests::cwd_must_be_absolute`
- `cli::selfdev::selfdev_tests::test_launcher_dir_ignores_blank_overrides_and_uses_home_default`
- 3x `auth_test_choice_plan_*` tests (failing before any choice.rs edits)
