# Vertex AI Provider

Routes Claude requests through Google Cloud Vertex AI instead of the direct Anthropic API.

## Activation

Set both environment variables:

```sh
export ANTHROPIC_VERTEX_PROJECT_ID=my-gcp-project
export CLOUD_ML_REGION=us-east5          # or ANTHROPIC_VERTEX_REGION
```

No `ANTHROPIC_API_KEY` or Claude OAuth login is required.

## Authentication (Google ADC)

Credentials are resolved in order:

1. **GCE/Cloud Run metadata server** - automatic when running on GCP
2. **`GOOGLE_APPLICATION_CREDENTIALS`** - path to a JSON key file
3. **`~/.config/gcloud/application_default_credentials.json`** - written by `gcloud auth application-default login`

Supported credential types: `authorized_user` (refresh token) and `service_account` (RSA JWT, signed with `ring`).

## Vertex AI vs direct Anthropic API differences

| Aspect | Direct Anthropic | Vertex AI |
|--------|-----------------|----------|
| Endpoint | `https://api.anthropic.com/v1/messages` | `https://{region}-aiplatform.googleapis.com/v1/projects/{project}/locations/{region}/publishers/anthropic/models/{model}:streamRawPredict` |
| Auth header | `x-api-key` or `Authorization: Bearer <oauth>` | `Authorization: Bearer <google-adc-token>` |
| `anthropic-version` | HTTP header | Request body field (`vertex-2023-10-16`) |
| `model` field | In request body | Stripped from body (encoded in URL) |
| `anthropic-beta` header | Required for prompt caching | Omitted (caching is native) |

## Where the code lives

| File | What it does |
|------|--------------|
| `crates/jcode-base/src/provider/anthropic.rs` | Google ADC fetch, JWT signing, `vertex_config()`, `vertex_endpoint()`, `has_vertex_credentials()`, `get_vertex_token()`, Vertex path in `complete()`/`complete_split()`/`stream_response()` |
| `crates/jcode-base/src/provider/route_builders.rs` | `build_anthropic_vertex_route()` |
| `crates/jcode-base/src/provider/catalog_routes.rs` | Injects Vertex routes via `auth.anthropic.has_vertex` |
| `crates/jcode-base/src/provider/startup.rs` | Initializes `AnthropicProvider` when Vertex creds present |
| `crates/jcode-base/src/auth/status_types.rs` | `ProviderAuth.has_vertex: bool` |
| `crates/jcode-base/src/auth/mod.rs` | `probe_anthropic_status()` populates `has_vertex` |

## Rebase checklist

When rebasing this branch onto a new upstream, re-apply changes to these files in dependency order:

1. **`crates/jcode-base/Cargo.toml`** - add `ring = "0.17"`
2. **`crates/jcode-base/src/auth/status_types.rs`** - add `has_vertex: bool` to `ProviderAuth`
3. **`crates/jcode-base/src/auth/mod.rs`** - populate `has_vertex` in `probe_anthropic_status()`
4. **`crates/jcode-base/src/provider/anthropic.rs`** - all Google ADC / Vertex logic
5. **`crates/jcode-base/src/provider/route_builders.rs`** - `build_anthropic_vertex_route()`
6. **`crates/jcode-base/src/provider/mod.rs`** - re-export `build_anthropic_vertex_route`
7. **`crates/jcode-base/src/provider/catalog_routes.rs`** - `auth.anthropic.has_vertex` checks
8. **`crates/jcode-base/src/provider/startup.rs`** - `(has_claude_creds || has_vertex)` condition

> **Tip:** `git show HEAD~1 -- <file>` on the feature branch to see exactly what was added vs upstream.
> The integration surface is intentionally narrow: most Vertex logic is self-contained in `anthropic.rs`;
> the other files only read `auth.anthropic.has_vertex` or call `build_anthropic_vertex_route()`.
