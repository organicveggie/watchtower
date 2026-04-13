# `internal/util` Package

This package provides general-purpose utility functions used across Watchtower. It covers three areas: collection
manipulation (slice and map diffing), random name generation for temporary container names, and random SHA-256 hash
generation for use in tests. None of these functions are specific to Docker or container management — they are pure
helpers with no external dependencies beyond the standard library.

---

## Files

### `util.go`

Provides functions for comparing and subtracting the contents of slices and maps. These are used primarily in
`pkg/container/container.go` when computing the delta between a running container's configuration and its source image
configuration, so that only the user-supplied overrides are carried forward when the container is recreated.

**Public Functions:**

---

#### `SliceEqual(s1, s2 []string) bool`

Returns `true` if both slices have the same length and contain the same elements in the same order. Returns `false`
otherwise. Used to compare environment variable lists, entrypoints, and command arrays between container and image
configs.

---

#### `SliceSubtract(a1, a2 []string) []string`

Returns a new slice containing all elements of `a1` that do not appear anywhere in `a2`. Order of the surviving elements
is preserved. Neither input slice is modified. Used to strip image-default environment variables from a container's env
list before recreating it, so that image defaults are not duplicated.

---

#### `StringMapSubtract(m1, m2 map[string]string) map[string]string`

Returns a new map containing entries from `m1` that are either absent from `m2` or present in `m2` with a different
value. Entries in `m1` whose key and value both match an entry in `m2` are excluded. Neither input map is modified. Used
to subtract image-default labels from a container's label set.

---

#### `StructMapSubtract(m1, m2 map[string]struct{}) map[string]struct{}`

Returns a new map containing entries from `m1` whose keys do not appear in `m2` at all. Neither input map is modified.
Used to subtract image-default exposed ports from a container's exposed port set.

---

### `rand_name.go`

Provides random container name generation. Used by `internal/actions/update.go` when Watchtower needs to rename its own
running container before starting a new instance under the original name.

**Variables:**

| Variable | Description |
|---|---|
| `letters` | The alphabet of characters used when generating random names: `a–z` and `A–Z`. |

**Public Functions:**

---

#### `RandName() string`

Generates and returns a random 32-character string composed of upper- and lower-case ASCII letters. The result is
compatible with Docker container name constraints. Uses `math/rand` (not cryptographically secure); suitable for
temporary rename operations where uniqueness rather than security is the goal.

---

### `rand_sha256.go`

Provides random SHA-256-style hash generation. Used in tests (`pkg/container/client_test.go`) to produce
realistic-looking image and container IDs without needing a live Docker daemon.

**Public Functions:**

---

#### `GenerateRandomSHA256() string`

Returns a random 64-character lowercase hexadecimal string representing a SHA-256 hash, without a `sha256:` prefix.
Implemented by calling `GenerateRandomPrefixedSHA256()` and stripping the leading `sha256:` prefix.

---

#### `GenerateRandomPrefixedSHA256() string`

Generates and returns a random SHA-256 hash string with a `sha256:` prefix, in the format `sha256:<64 hex characters>`.
This matches the full image ID format used by the Docker API. Uses `crypto/rand` for cryptographically secure random
bytes.

---

## Test Coverage

`util_test.go` covers all functions in `util.go` and both functions in `rand_sha256.go`:

| Test | Description |
|---|---|
| `TestSliceEqual_True` | Verifies that two identical slices are considered equal. |
| `TestSliceEqual_DifferentLengths` | Verifies that slices of different lengths are not equal. |
| `TestSliceEqual_DifferentContents` | Verifies that slices of the same length but different contents are not equal. |
| `TestSliceSubtract` | Verifies that elements present in `a2` are removed from `a1`, and that neither input is mutated. |
| `TestStringMapSubtract` | Verifies that matching key-value pairs are removed and differing values are retained, without mutating inputs. |
| `TestStructMapSubtract` | Verifies that keys present in `m2` are removed from `m1`, without mutating inputs. |
| `TestGenerateRandomSHA256` | Verifies that the result is exactly 64 characters long and does not contain a `sha256:` prefix. |
| `TestGenerateRandomPrefixedSHA256` | Verifies that the result matches the pattern `sha256:[0-9a-f]{64}`. |

`rand_name.go` has no dedicated unit tests; its output is implicitly exercised through the update action tests.
