# `pkg/registry/digest` Package

This package implements the image digest comparison logic that allows Watchtower to determine whether a container's image is stale without always pulling it. It makes an authenticated HTTP HEAD request to the registry's manifest endpoint, reads the `Docker-Content-Digest` response header, and compares it against the locally stored repo digests. It is consumed by `pkg/container/client.go` in `PullImage` to skip unnecessary pulls when the digest already matches.

---

## Files

### `digest.go`

Implements three public functions covering the full digest-check lifecycle: credential transformation, digest fetching, and digest comparison.

**Constants:**

| Constant | Value | Description |
|---|---|---|
| `ContentDigestHeader` | `"Docker-Content-Digest"` | The HTTP response header returned by the registry containing the manifest digest. Read by `GetDigest` to extract the remote digest value. |

---

**Public Functions:**

---

#### `CompareDigest(container types.Container, registryAuth string) (bool, error)`

The top-level entry point. Determines whether the remote registry has a newer image than the one the container is currently running. The full flow is:

1. Returns an error immediately if the container has no image info (`container.HasImageInfo()` is false).
2. Calls `TransformAuth` to convert the base64-encoded JSON auth config into the `user:pass` base64 format expected by the registry API.
3. Calls `auth.GetToken` to exchange the credentials for a usable `Authorization` header value via the registry's challenge-response flow.
4. Calls `manifest.BuildManifestURL` to construct the registry manifest endpoint URL for the container's image and tag.
5. Calls `GetDigest` to make a HEAD request and retrieve the `Docker-Content-Digest` header value.
6. Iterates over `container.ImageInfo().RepoDigests`, splitting each entry on `@` to extract the local digest, and compares it to the remote digest.

Returns `(true, nil)` if a matching local digest is found (image is up to date), `(false, nil)` if no match is found (image is stale), or `(false, error)` if any step fails.

---

#### `TransformAuth(registryAuth string) string`

Converts a registry auth credential from the Docker config format into the format expected by the registry API.

Docker stores credentials in `config.json` as a base64-encoded JSON object (`{"username":"...","password":"..."}`). Registry APIs expect a base64-encoded `username:password` string. This function:

1. Base64-decodes `registryAuth`.
2. Attempts to JSON-unmarshal the result into a `types.RegistryCredentials` struct.
3. If both `Username` and `Password` are non-empty, re-encodes them as `base64("username:password")` and returns the result.
4. If unmarshalling fails or either field is empty, returns `registryAuth` unchanged.

This means the function is a no-op for credentials that are already in the correct format.

---

#### `GetDigest(url string, token string) (string, error)`

Makes an authenticated HTTP HEAD request to the registry manifest endpoint and returns the value of the `Docker-Content-Digest` response header.

Request details:
- Uses a custom `http.Transport` with `InsecureSkipVerify: true` (TLS verification is disabled to support self-hosted registries with self-signed certificates).
- Sets `User-Agent` to `meta.UserAgent` (e.g. `"Watchtower/v1.x.y"`), making Watchtower identifiable in registry access logs.
- Returns an error immediately if `token` is empty.
- Sets the `Authorization` header to the provided token.
- Accepts all four manifest media types: `manifest.v2+json`, `manifest.list.v2+json`, `manifest.v1+json`, and `vnd.oci.image.index.v1+json`, ensuring compatibility with both legacy and OCI registries.

Returns an error if the HTTP request fails or the registry responds with any status other than `200`. The error message includes both the HTTP status and the `WWW-Authenticate` header (if present) to aid diagnosis.

---

## Test Coverage

`digest_test.go` bootstraps a Ginkgo suite (`"Digest Suite"`) and covers the following:

| Test | Description |
|---|---|
| `CompareDigest` — digests match (integration) | Calls the real GHCR API using credentials from `CI_INTEGRATION_TEST_REGISTRY_GH_USERNAME` / `CI_INTEGRATION_TEST_REGISTRY_GH_PASSWORD`. Skipped automatically if either is empty. Verifies that `true` is returned when the container's local digest matches the remote. |
| `CompareDigest` — digests differ | Placeholder test (no implementation body). |
| `CompareDigest` — registry unavailable | Placeholder test (no implementation body). |
| `CompareDigest` — no image info | Verifies that passing a container with a `nil` image info returns `false` and a non-nil error, without making any network requests. |
| `CompareDigest` — DockerHub (integration) | Skipped unless `CI_INTEGRATION_TEST_REGISTRY_DH_USERNAME` / `CI_INTEGRATION_TEST_REGISTRY_DH_PASSWORD` are set. Currently has no assertion body. |
| `CompareDigest` — GHCR (integration) | Skipped unless GHCR credentials are set. Currently has no assertion body. |
| `GetDigest` — custom User-Agent | Uses a `ghttp` mock server to verify that HEAD requests include the expected `User-Agent: Watchtower/v0.0.0-unknown` header, and that the `Docker-Content-Digest` header value from the response is correctly returned. |
