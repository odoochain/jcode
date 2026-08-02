# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [v0.65.1] - 2026-08-03

### Added
- **TLS Self-Signed Certificate Support**: New `accept_invalid_certs` config field on `NamedProviderConfig` for per-provider self-signed certificate acceptance (`jcode-config-types`, `jcode-provider-core`, `jcode-base`, `jcode-provider-openrouter-runtime`)
- **`insecure_http_client()`**: Shared reqwest client with TLS verification disabled, used as fallback when `accept_invalid_certs = true` (`jcode-provider-core`)
- **TUI Model Picker Tiering**: Configured/authenticated providers (OAuth, API key, OpenAI-compatible, Copilot, Cursor, Bedrock, ...) now rank above OpenRouter's auto-discovered catalog in the model selection UI (`jcode-tui`)
- **Documentation**: `docs/TLS_SELF_SIGNED_CERT_SUPPORT.md` with full architecture details and verification results

### Fixed
- **credential_probe false negative**: `apply_login_provider_profile_env()` no longer overwrites named profile env vars (`--provider-profile`) with built-in defaults — added early return guard on `JCODE_NAMED_PROVIDER_PROFILE` (`src/cli/provider_init.rs`)
- **provider_smoke wrong URL**: `discover_openai_compatible_validation_model()` now reads `JCODE_OPENROUTER_API_BASE` env var as override instead of using hardcoded profile `api_base` (`src/cli/auth_test/choice.rs`)
- **panic ratchet regression**: Replaced `.expect("failed to build insecure HTTP client")` with `.unwrap_or_else(|_| shared_http_client())` graceful fallback in `insecure_http_client()` (`jcode-provider-core`)

### Changed
- `.gitignore`: Added `/nul` to ignore Windows device file artifact

### Configuration Example
```toml
[providers.my-gateway]
type = "openai-compatible"
base_url = "https://internal-gateway.company.local/v1"
api_key_env = "MY_GATEWAY_API_KEY"
model = "my-model"
accept_invalid_certs = true  # Allow self-signed certs
```

### Notes
- Based on upstream [v0.65.0](https://github.com/1jehuang/jcode/releases/tag/v0.65.0)
- Release binary: `jcode-x86_64-pc-windows-msvc.exe` (104MB, LTO optimized)
- All changes pass `cargo fmt` and clippy (zero new warnings on affected crates)

---

## [v0.65.0] - 2026-08-02 (upstream)

Upstream release by @1jehuang. See [1jehuang/jcode releases](https://github.com/1jehuang/jcode/releases/tag/v0.65.0) for full details.

Notable upstream changes merged into this build:
- Ctrl+L: collapse cleared screen so prompt sits at top
- Cmd+L: terminal-style clear (blank screen, history above)
- Claude browse recall improvement: 83% -> 100%
- Package-install false positive fix
- Discovery baseline reframing around Claude
- Various test coverage improvements
