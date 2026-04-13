# `pkg/registry` Packages — Feature Summary

## `pkg/registry` — Credential Loading and Pull Options

The top-level registry integration layer. Handles credential loading, pull option construction, and HEAD request warning
strategy.

### Pull Options (`registry.go`)

- **Pull option construction**: `GetPullOptions` loads credentials for the target registry and returns a
  `types.ImagePullOptions` struct ready for the Docker SDK's `ImagePull` call. Returns empty options (allowing
  unauthenticated pulls) when no credentials are found.
- **Auth failure fallback**: `DefaultAuthHandler` is registered as the Docker SDK's `PrivilegeFunc`. If an initial
  authenticated pull is rejected, it retries without credentials rather than failing hard — allowing public images on
  strict registries to still be pulled.
- **HEAD request warning strategy**: `WarnOnAPIConsumption` determines whether a failed registry HEAD request should be
  logged as a warning. Returns `true` (warn) for Docker Hub and ghcr.io, which are known to support HEAD requests
  reliably, and for any image reference that cannot be parsed. Returns `false` for all other registries, where a HEAD
  failure may simply mean the endpoint is not implemented.

### Credential Loading (`trust.go`)

- **Credential source priority**: `EncodedAuth` tries `REPO_USER`/`REPO_PASS` environment variables first; falls back to
  the Docker config file if either variable is unset.
- **Environment variable credentials**: `EncodedEnvAuth` reads `REPO_USER` and `REPO_PASS`. When both are set, the same
  credentials are applied globally to all registries.
- **Docker config file credentials**: `EncodedConfigAuth` resolves the registry hostname from the image reference, loads
  the Docker config from `DOCKER_CONFIG` (defaulting to `/`), and retrieves the per-registry credentials. Returns an
  empty string without error when no credentials are found for the registry.
- **Credentials store selection**: `CredentialsStore` returns a native OS credential store (e.g.
  `docker-credential-osxkeychain`) if one is configured, or a plain file store reading from the config's `auths` section
  otherwise.
- **Auth encoding**: `EncodeAuth` JSON-marshals a `types.AuthConfig` and URL-safe base64-encodes the result, producing
  the value used as the `X-Registry-Auth` header.

---

## `pkg/registry/auth` — Challenge-Response Authentication

Implements the full Docker Registry HTTP API V2 authentication flow used when performing digest checks.

- **Token acquisition**: `GetToken` drives the complete challenge-response cycle — hits the registry's `/v2/` endpoint,
  reads the `WWW-Authenticate` header, and dispatches to either Basic or Bearer token handling. Returns a fully
  formatted `Authorization` header value ready for use in subsequent requests.
- **Basic auth**: When the challenge type is `basic`, returns `"Basic <registryAuth>"` directly. Returns an error if no
  credentials are available.
- **Bearer token exchange**: `GetBearerHeader` sends a `GET` to the token endpoint with optional `Basic` credentials,
  unmarshals the JSON response, and returns `"Bearer <token>"`.
- **Auth URL construction**: `GetAuthURL` parses the bearer challenge string, extracts `realm` and `service`, and
  constructs the token endpoint URL with a `scope` query parameter of the form `repository:<image-path>:pull`. Correctly
  prepends `library/` for Docker Hub official images and strips vanity host prefixes (`docker.io`, `index.docker.io`)
  without affecting other registries. Handles malformed challenge strings (trailing commas, valueless keys) without
  panicking.
- **Challenge URL construction**: `GetChallengeURL` builds the registry's `/v2/` HTTPS endpoint URL from an image
  reference, normalising `docker.io` to `index.docker.io`.

---

## `pkg/registry/digest` — Image Digest Comparison

Determines whether a container's image is stale by comparing local and remote digests, avoiding unnecessary image pulls.

- **Digest comparison**: `CompareDigest` orchestrates the full staleness check — transforms credentials, obtains a
  token, builds the manifest URL, fetches the remote digest via HEAD request, and compares it against each entry in the
  container's local `RepoDigests`. Returns `true` if any local digest matches (image is current), `false` if none match
  (image is stale).
- **Credential transformation**: `TransformAuth` converts Docker config's base64-encoded JSON credential format
  (`{"username":"...","password":"..."}`) into the `base64(username:password)` format expected by registry APIs. Acts as
  a no-op if credentials are already in the correct format.
- **Digest fetching**: `GetDigest` makes an authenticated HEAD request to the manifest endpoint. Accepts all four
  manifest media types (`manifest.v2+json`, `manifest.list.v2+json`, `manifest.v1+json`, `oci.image.index.v1+json`) for
  compatibility with both legacy and OCI registries. Disables TLS verification to support self-hosted registries with
  self-signed certificates. Identifies Watchtower via the `meta.UserAgent` `User-Agent` header. Returns a descriptive
  error including the HTTP status and `WWW-Authenticate` header on non-200 responses.

---

## `pkg/registry/helpers` — Registry Address Resolution

A shared low-level utility consumed by all other registry subpackages.

- **Registry hostname extraction**: `GetRegistryAddress` parses any Docker image reference string and returns the
  registry hostname. Normalises `docker.io` to `index.docker.io` (the actual Docker Hub API host). Handles bare image
  names, explicit registry prefixes, local hosts with ports, and fully qualified hostnames.
- **Domain constants**: Exports `DefaultRegistryDomain` (`"docker.io"`), `DefaultRegistryHost` (`"index.docker.io"`),
  and `LegacyDefaultRegistryDomain` (`"index.docker.io"`) for use across the registry packages.

---

## `pkg/registry/manifest` — Manifest URL Construction

Constructs the Docker Registry V2 manifest endpoint URL used by the digest comparison flow.

- **Manifest URL building**: `BuildManifestURL` parses a container's image name, normalises the reference (defaulting to
  `:latest` when no tag is specified), resolves the registry host, extracts the image path and tag, and returns an HTTPS
  URL of the form `https://<host>/v2/<image>/manifests/<tag>`.
- **Digest-pinned image rejection**: Returns an error for `sha256:`-pinned image references, which have no tag and
  therefore no manifest URL to construct.
- **Docker Hub path handling**: Uses `ref.Path` to correctly prepend `library/` for Docker Hub official single-segment
  images while leaving paths on other registries unchanged.
