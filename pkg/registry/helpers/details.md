# `pkg/registry/helpers` Package

This package provides a single utility function for extracting the registry host address from a Docker image reference.
It is the shared, low-level building block used across the registry packages — consumed by `pkg/registry/auth`,
`pkg/registry/digest`, `pkg/registry/manifest`, and `pkg/registry/trust` wherever a registry hostname needs to be
resolved from an image name.

---

## Files

### `helpers.go`

Declares three domain constants and one public function.

**Constants:**

| Constant | Value | Description |
|---|---|---|
| `DefaultRegistryDomain` | `"docker.io"` | The canonical domain used in Docker image references when no registry is specified (e.g. `nginx` → `docker.io/library/nginx`). |
| `DefaultRegistryHost` | `"index.docker.io"` | The actual API hostname for Docker Hub. `GetRegistryAddress` normalises `docker.io` references to this value, since `index.docker.io` is the address that accepts authenticated API requests. |
| `LegacyDefaultRegistryDomain` | `"index.docker.io"` | An alias for `DefaultRegistryHost`, retained for backward compatibility with older Docker config files that may store credentials under this key. |

---

**Public Functions:**

---

#### `GetRegistryAddress(imageRef string) (string, error)`

Parses an image reference string and returns the hostname of the registry that hosts it. Uses
`reference.ParseNormalizedNamed` to normalise the reference (adding default registry and tag components where absent)
and then calls `reference.Domain` to extract the host portion.

Applies one post-processing step: if the extracted domain is `DefaultRegistryDomain` (`"docker.io"`), it is replaced
with `DefaultRegistryHost` (`"index.docker.io"`), ensuring callers always receive the real API host rather than the
vanity domain.

Returns an error if `imageRef` is empty or cannot be parsed as a valid image reference.

**Behaviour by input format:**

| Input | Returned address |
|---|---|
| `"watchtower"` | `"index.docker.io"` |
| `"containrrr/watchtower"` | `"index.docker.io"` |
| `"docker.io/containrrr/watchtower"` | `"index.docker.io"` |
| `"ghcr.io/containrrr/watchtower"` | `"ghcr.io"` |
| `"github.com/containrrr/config"` | `"github.com"` |
| `"localhost/watchtower"` | `"localhost"` |
| `"henk:80/watchtower"` | `"henk:80"` |
| `""` | `""` + error |

---

## Test Coverage

`helpers_test.go` bootstraps a Ginkgo suite (`"Helper Suite"`) and covers `GetRegistryAddress` across five cases:

| Test | Description |
|---|---|
| Empty string | Verifies that an empty input returns an error. |
| No explicit registry | Verifies that bare image names (`"watchtower"`, `"containrrr/watchtower"`) resolve to `"index.docker.io"`. |
| `docker.io` domain | Verifies that `docker.io`-prefixed references also resolve to `"index.docker.io"`. |
| Local host | Verifies that `"localhost/watchtower"` and `"henk:80/watchtower"` return the host as-is. |
| Fully qualified name | Verifies that `"github.com/containrrr/config"` returns `"github.com"`. |
