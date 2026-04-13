# `pkg/registry/manifest` Package

This package provides a single function for constructing the Docker Registry V2 manifest endpoint URL for a given container's image. The resulting URL is used by `pkg/registry/digest` when making HEAD requests to check whether a container's image is stale. It is the canonical place where image name, tag, and registry host are combined into a well-formed registry API URL.

---

## Files

### `manifest.go`

Contains a single public function.

**Public Functions:**

---

#### `BuildManifestURL(container types.Container) (string, error)`

Constructs and returns the HTTPS URL of the Docker Registry V2 manifest endpoint for the container's image. The full sequence is:

1. Parses `container.ImageName()` using `ref.ParseDockerRef`, which normalises the image reference and appends `:latest` if no tag is specified.
2. Asserts that the normalised reference implements `ref.NamedTagged`. Returns an error if the reference has no tag — this prevents requests for digest-pinned images (e.g. `image@sha256:...`), which have no manifest tag to look up.
3. Calls `helpers.GetRegistryAddress` on the tagged reference's name to resolve the registry host (e.g. `docker.io` → `index.docker.io`).
4. Extracts the image path via `ref.Path` (which correctly handles the `library/` prefix for Docker Hub official images) and the tag via `normalizedTaggedRef.Tag()`.
5. Constructs and returns an `https` URL of the form `https://<host>/v2/<image>/manifests/<tag>`.

Returns an error if the image reference cannot be parsed or if the reference has no tag (including digest-pinned images).

**Behaviour by input:**

| Image reference | Resulting URL |
|---|---|
| `"ghcr.io/containrrr/watchtower:mytag"` | `"https://ghcr.io/v2/containrrr/watchtower/manifests/mytag"` |
| `"containrrr/watchtower:latest"` | `"https://index.docker.io/v2/containrrr/watchtower/manifests/latest"` |
| `"containrrr/watchtower"` _(no tag)_ | `"https://index.docker.io/v2/containrrr/watchtower/manifests/latest"` |
| `"docker-registry.domain/imagename:latest"` | `"https://docker-registry.domain/v2/imagename/manifests/latest"` |
| `"image@sha256:daf703..."` _(pinned)_ | `""` + error |

---

## Test Coverage

`manifest_test.go` bootstraps a Ginkgo suite (`"Manifest Suite"`) and covers `BuildManifestURL` across five cases. Each test constructs a mock container via `mocks.CreateMockContainerWithImageInfo` with the image reference set as a repo tag, then calls `BuildManifestURL` on it.

| Test | Description |
|---|---|
| Fully qualified image | Verifies that `"ghcr.io/containrrr/watchtower:mytag"` produces the correct GHCR manifest URL. |
| No explicit registry | Verifies that `"containrrr/watchtower:latest"` defaults to `index.docker.io` as the registry host. |
| No explicit tag | Verifies that `"containrrr/watchtower"` (no tag) defaults to `latest` and resolves to `index.docker.io`. |
| Non-Docker Hub single-part image name | Verifies that `"docker-registry.domain/imagename:latest"` does not have `library/` prepended to the image path, since it is not on Docker Hub. |
| Digest-pinned image | Verifies that a `sha256:`-pinned image reference returns an error and an empty URL, since pinned references have no tag to construct a manifest URL from. |
