# `internal/meta` Package

This package exposes build-time metadata about the Watchtower binary. It is the single source of truth for the application's version string and the HTTP `User-Agent` header used in outbound registry requests.

---

## Files

### `meta.go`

Declares two package-level variables that are intended to be set at compile time via `-ldflags` and initialises derived values from them in an `init` function.

**Variables:**

| Variable | Default | Description |
|---|---|---|
| `Version` | `"v0.0.0-unknown"` | The version string for this build of Watchtower. Overridden at build time with `-X github.com/containrrr/watchtower/internal/meta.Version=$(git describe --tags)`. Falls back to `"v0.0.0-unknown"` in development builds. |
| `UserAgent` | _(set by `init`)_ | The `User-Agent` header value sent with outbound HTTP requests to image registries. Derived from `Version` as `"Watchtower/" + Version`. Consumed by `pkg/registry/digest` when making HEAD requests to check image digests. |

There are no exported functions in this package. All behaviour is initialised automatically via `init()` when the package is first imported.

---

## Usage

The `Version` variable is injected during the Docker image build in both `dockerfiles/Dockerfile.dev-self-contained` and `dockerfiles/Dockerfile.self-contained`:

```sh
go build -ldflags "-X github.com/containrrr/watchtower/internal/meta.Version=$(git describe --tags)"
```

`UserAgent` is referenced in `pkg/registry/digest/digest.go` to identify Watchtower in registry API requests, and `Version` is referenced in `cmd/root.go` to include the version in startup log messages.
