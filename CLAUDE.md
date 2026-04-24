# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Build (requires assets/ directory to exist)
go build

# Run
./gohttpserver -r ./ --port 8000 --upload

# Run with config file
./gohttpserver --conf testdata/config.yml

# Cross-compile release builds (outputs to dist/)
./build.sh

# Docker build (from repo root)
docker build -t gohttpserver -f docker/Dockerfile .
```

## Testing

```bash
# Run all tests
go test ./...

# Run a single test
go test -run TestCleanPath ./...
```

Test coverage is minimal — only `utils_test.go` and `zip_test.go` exist. Uses `github.com/stretchr/testify`.

## Architecture

This is a self-contained HTTP file server with a web UI. All frontend assets are embedded into the binary via `go:embed`.

### Key Files

| File | Purpose |
|------|---------|
| `main.go` | Entry point, CLI flag parsing (kingpin), config loading, middleware setup |
| `httpstaticserver.go` | Core server: all HTTP handlers, file serving, upload/delete, search indexing |
| `openid-login.go` | OpenID Connect authentication |
| `oauth2-proxy.go` | OAuth2 proxy integration (reads user from `X-Auth-Request-*` headers) |
| `ipa.go` | iOS .ipa handling: plist generation, QR codes for install |
| `zip.go` | Directory ZIP download |
| `utils.go` | Path sanitization and string helpers |
| `assets.go` | Embeds the `assets/` directory |
| `res.go` | Resource/embed declarations |

### Request Flow

```
Request → ProxyHeaders → CORS → BasicAuth → AccessLog → HTTPStaticServer
                                                              ↓
                                                     Gorilla Mux routing
                                                              ↓
                                                   getRealPath() validates path
                                                              ↓
                                                   Check .ghs.yml permissions
                                                              ↓
                                                   Handler (hIndex/hUploadOrMkdir/hDelete)
```

- Routes use `{path:.*}` catch-all pattern. Handler dispatch is based on HTTP method and query parameters (e.g., `?json=true` for JSON API responses).
- `hIndex` handles GET/HEAD — serves files directly, renders directory listing HTML, or returns JSON.
- `hUploadOrMkdir` handles POST — multipart upload or mkdir via `mkdir=true`.
- `hDelete` handles DELETE.

### Per-Directory Access Control

`.ghs.yml` files in subdirectories control upload/delete permissions and access rules. Settings inherit from parent directories. Structure defined in `AccessConf` type with `Users` (per-email rules) and `AccessTables` (regex-based file visibility).

### Authentication

Three modes selected via `--auth-type`:
- `http` — Basic auth with SHA256-hashed passwords
- `openid` — OpenID Connect with session storage (gorilla/sessions)
- `oauth2-proxy` — Delegates auth to a reverse proxy

### Search

In-memory file index rebuilt every 10 minutes. Max 50 results per query. Supports keyword exclusion with `-` prefix.

### Frontend

Single-page app in `assets/index.html` using Vue.js 1.x. Themes (black, green) defined in `assets/themes/`. No build step for frontend — served directly from embedded assets.
