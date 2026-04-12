# `pkg/registry` Package

This package is the top-level registry integration layer for Watchtower. It is responsible for loading and encoding Docker registry credentials from environment variables or the Docker config file, constructing the pull options passed to the Docker SDK, and determining whether a failed registry HEAD request should be surfaced as a warning. It is consumed by `pkg/container/client.go` for both image pulls (`GetPullOptions`) and digest comparisons (`WarnOnAPIConsumption`). Credential loading lives here; the lower-level token exchange and digest comparison logic lives in the `auth` and `digest` subpackages respectively.

---

## Files

### `registry.go`

Provides the pull option constructor, the auth failure handler, and the API consumption warning predicate.

**Public Functions:**

---

#### `GetPullOptions(imageName string) (types.ImagePullOptions, error)`

Constructs and returns a `types.ImagePullOptions` struct suitable for passing to the Docker SDK's `ImagePull` call. The function:

1. Calls `EncodedAuth(imageName)` to load credentials for the image's registry, trying environment variables first and falling back to the Docker config file.
2. If `auth` is empty (no credentials found), returns an empty `ImagePullOptions` with no auth — allowing unauthenticated pulls to proceed normally.
3. If credentials are present, returns `ImagePullOptions` with `RegistryAuth` set to the encoded credentials and `PrivilegeFunc` set to `DefaultAuthHandler`.

Returns an error only if `EncodedAuth` itself returns one (i.e. the image reference cannot be parsed).

---

#### `DefaultAuthHandler() (string, error)`

The `PrivilegeFunc` callback registered in `GetPullOptions`. Called by the Docker SDK if the initial authenticated pull attempt is rejected. Always returns an empty string and `nil`, effectively retrying without authentication. Logs a debug message to indicate the fallback. This prevents a rejected credential from causing a hard failure when an unauthenticated pull might still succeed (e.g. for public images on registries with strict auth enforcement).

---

#### `WarnOnAPIConsumption(container watchtowerTypes.Container) bool`

Returns `true` if a failed HEAD request to this container's registry should be logged as a warning, or `false` if it should be silently ignored. Used by `pkg/container/client.go` to implement the `WarnAuto` strategy for `--warn-on-head-failure`.

Parses the container's image name into a normalised reference and resolves the registry host via `helpers.GetRegistryAddress`. Returns `true` (warn) in three cases:

- The image name cannot be parsed.
- The registry address cannot be resolved.
- The resolved host is `index.docker.io` (Docker Hub) or `ghcr.io` — both of which are known to support and reliably respond to HEAD requests, so a failure there is genuinely noteworthy.

Returns `false` for all other registries, where HEAD request support is unknown and a failure may simply mean the registry does not implement the endpoint.

---

### `trust.go`

Implements credential loading from environment variables and the Docker config file, and provides the encoding helpers used throughout the registry package.

**Public Functions:**

---

#### `EncodedAuth(ref string) (string, error)`

The top-level credential loader. Attempts to load encoded registry credentials for the image referenced by `ref`, trying sources in priority order:

1. **Environment variables** (`REPO_USER` / `REPO_PASS`) via `EncodedEnvAuth`. If both are set, their values are used and the Docker config is not consulted.
2. **Docker config file** via `EncodedConfigAuth(ref)`, which looks up credentials for the specific registry hosting `ref`.

Returns the encoded credential string from whichever source succeeds first, or an error if both fail.

---

#### `EncodedEnvAuth() (string, error)`

Reads the `REPO_USER` and `REPO_PASS` environment variables. If both are non-empty, constructs a `types.AuthConfig` from them and returns the result of `EncodeAuth`. Returns an error if either variable is unset or empty. This source applies globally — the same credentials are used for all registries when set.

---

#### `EncodedConfigAuth(imageRef string) (string, error)`

Loads registry credentials for the specific registry hosting `imageRef` from the Docker config file. The full sequence is:

1. Calls `helpers.GetRegistryAddress(imageRef)` to resolve the registry hostname.
2. Reads the config directory from `DOCKER_CONFIG`, falling back to `"/"` if unset (the path where Watchtower expects the config to be mounted inside the container).
3. Loads the config file via `cliconfig.Load`.
4. Obtains a `credentials.Store` via `CredentialsStore`, which uses a native OS credential store if `CredentialsStore` is set in the config, or a plain file store otherwise.
5. Calls `credStore.Get(server)` to look up credentials for the resolved server address.
6. Returns an empty string (no error) if no credentials are found for the server, or the encoded credential string if found.

Returns an error if the registry address cannot be resolved or if the config file cannot be loaded.

---

#### `CredentialsStore(configFile configfile.ConfigFile) credentials.Store`

Returns the appropriate Docker credentials store for the given config file. If `configFile.CredentialsStore` is non-empty, returns a `credentials.NativeStore` backed by the named external credential helper (e.g. `docker-credential-osxkeychain`). Otherwise returns a `credentials.FileStore` that reads credentials directly from the config file's `auths` section.

---

#### `EncodeAuth(authConfig types.AuthConfig) (string, error)`

Base64-encodes a `types.AuthConfig` struct (using URL-safe base64) for transmission as the `X-Registry-Auth` HTTP header value expected by the Docker SDK and registry APIs. Marshals the struct to JSON first, then encodes the result. Returns an error if JSON marshalling fails.

---

## Test Coverage

| File | Description |
|---|---|
| `registry_suite_test.go` | Bootstraps the Ginkgo test suite (`"Registry Suite"`) for the `registry_test` package. Redirects logrus output to `GinkgoWriter` so log messages appear in test output on failure. |
| `registry_test.go` | External package tests (`package registry_test`) covering `WarnOnAPIConsumption` across four cases: a `ghcr.io` image returns `true`; an implicit Docker Hub image (no registry prefix) returns `true`; explicit Docker Hub images (`index.docker.io/` and `docker.io/` prefixed) return `true`; images from other third-party registries (`docker.fsf.org`, `altavista.com`, `gitlab.com`) return `false`. Uses a `testContainerWithImage` helper that constructs a mock container via `mocks.CreateMockContainer`. |
| `trust_test.go` | Internal package tests (`package registry`) covering: `EncodedAuth` — verifies that env credentials (`REPO_USER`/`REPO_PASS`) produce the expected base64-encoded JSON string; `EncodedEnvAuth` — verifies that an error is returned when both env vars are unset; `EncodedConfigAuth` — verifies that an error is returned when `DOCKER_CONFIG` points to a non-existent path. |
