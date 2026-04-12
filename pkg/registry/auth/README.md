# `pkg/registry/auth` Package

This package implements the Docker registry authentication flow used when checking image digests. It handles the full OAuth2 bearer token and HTTP Basic challenge-response cycle required by the Docker Registry HTTP API V2. It is consumed by `pkg/registry/digest` when making HEAD requests to verify whether a container's image is stale. It is distinct from the credential-loading logic in `pkg/registry/trust.go`, which handles retrieving stored credentials — this package handles exchanging those credentials for a usable token.

---

## Files

### `auth.go`

Implements the complete challenge-response authentication flow in five public functions.

**Constants:**

| Constant | Value | Description |
|---|---|---|
| `ChallengeHeader` | `"WWW-Authenticate"` | The HTTP response header name that carries authentication challenge instructions from the registry. |

---

**Public Functions:**

---

#### `GetToken(container types.Container, registryAuth string) (string, error)`

The top-level entry point. Obtains a usable `Authorization` header value for making authenticated requests to the registry hosting the container's image. The full flow is:

1. Parses the container's image name into a normalised reference via `ref.ParseNormalizedNamed`.
2. Calls `GetChallengeURL` to build the registry's `/v2/` challenge endpoint URL.
3. Calls `GetChallengeRequest` to construct a `GET` request to that URL.
4. Sends the request and reads the `WWW-Authenticate` response header.
5. Dispatches based on the challenge type:
   - **`basic`**: Returns `"Basic <registryAuth>"` directly. Returns an error if `registryAuth` is empty.
   - **`bearer`**: Delegates to `GetBearerHeader` to exchange credentials for a token.
   - **anything else**: Returns an `"unsupported challenge type"` error.

Returns the fully formatted `Authorization` header value (e.g. `"Bearer <token>"` or `"Basic <encoded>"`), ready to be set on subsequent registry requests.

---

#### `GetChallengeRequest(URL url.URL) (*http.Request, error)`

Constructs and returns a `GET` `http.Request` for the given challenge URL. Sets `Accept: */*` and `User-Agent: Watchtower (Docker)` headers. Returns an error if the request cannot be constructed.

---

#### `GetBearerHeader(challenge string, imageRef ref.Named, registryAuth string) (string, error)`

Exchanges credentials for a bearer token by:

1. Calling `GetAuthURL` to parse the challenge string and construct the token endpoint URL (including `service` and `scope` query parameters).
2. Sending a `GET` request to the token endpoint, adding an `Authorization: Basic <registryAuth>` header if credentials are available. Logs a debug message in either case.
3. Reading and JSON-unmarshalling the response body into a `types.TokenResponse`.
4. Returning the token formatted as `"Bearer <token>"`.

Returns an error if the auth URL cannot be parsed, the HTTP request fails, or the JSON response cannot be unmarshalled.

---

#### `GetAuthURL(challenge string, imageRef ref.Named) (*url.URL, error)`

Parses the `WWW-Authenticate` bearer challenge string and constructs the token endpoint URL. The challenge string is expected to be in the format:

```
bearer realm="<url>",service="<service>",scope="..."
```

The function:

1. Strips the `"bearer"` prefix and splits on `,` to extract key-value pairs.
2. Uses `strings.Cut` on each pair to build a `map[string]string` of challenge values.
3. Returns an error if either `realm` or `service` is absent.
4. Parses `realm` as the base URL, then appends `service` and a `scope` query parameter of the form `repository:<image-path>:pull`, where `<image-path>` is derived from `ref.Path(imageRef)`.

The use of `ref.Path` correctly handles Docker Hub's `library/` prefix for official images and strips vanity host prefixes (`docker.io`, `index.docker.io`) — so `docker.io/nginx` becomes `library/nginx` but `ghcr.io/containrrr/watchtower` becomes `containrrr/watchtower`.

---

#### `GetChallengeURL(imageRef ref.Named) url.URL`

Constructs the challenge URL for the registry hosting `imageRef`. Calls `helpers.GetRegistryAddress` to extract the registry hostname from the image reference, then returns a `url.URL` with scheme `https`, the extracted host, and path `/v2/`. This is the standard Docker Registry V2 endpoint that responds with a `WWW-Authenticate` challenge when accessed without credentials.

---

## Test Coverage

`auth_test.go` bootstraps a Ginkgo suite (`"Registry Auth Suite"`) and covers the following:

| Test | Description |
|---|---|
| `GetToken` (integration) | Calls the real GHCR API with credentials from `CI_INTEGRATION_TEST_REGISTRY_GH_USERNAME` / `CI_INTEGRATION_TEST_REGISTRY_GH_PASSWORD`. Skipped automatically if either variable is empty. Verifies that a non-empty token is returned without error. |
| `GetAuthURL` — valid challenge | Parses a well-formed bearer challenge string for `ghcr.io` and verifies the resulting URL matches the expected scheme, host, path, and query parameters exactly. |
| `GetAuthURL` — missing service | Verifies that a challenge string with only `realm` and no `service` returns an error and a nil URL. |
| `GetAuthURL` — Docker Hub official image scope | Verifies that `registry`, `docker.io/registry`, and `index.docker.io/registry` all produce a scope of `library/registry`. |
| `GetAuthURL` — Docker Hub vanity host stripping | Verifies that `docker.io/containrrr/watchtower` and `index.docker.io/containrrr/watchtower` both produce a scope of `containrrr/watchtower` (no `library/` prefix). |
| `GetAuthURL` — three-segment image names | Verifies that `piksel/containrrr/watchtower` and `ghcr.io/piksel/containrrr/watchtower` both produce a scope of `piksel/containrrr/watchtower`. |
| `GetAuthURL` — non-Docker Hub single-segment images | Verifies that `ghcr.io/watchtower` produces a scope of `watchtower` (no `library/` prefix added for non-hub registries). |
| `GetAuthURL` — trailing comma in challenge | Verifies that a challenge string with a trailing empty field (e.g. `...,scope="...",`) does not panic and returns a valid URL without error. |
| `GetAuthURL` — empty key-value pair | Verifies robustness against a challenge string containing a bare `=` with no key. |
