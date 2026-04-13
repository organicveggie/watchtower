# `pkg/filters` Package

This package provides all container filtering logic used by Watchtower to determine which containers should be monitored
and updated. Filters are composable functions that each accept a `types.FilterableContainer` and return a boolean, and
are chained together to form a combined filter that a container must fully satisfy to be included in a session. The
package is consumed by `cmd/root.go`, `internal/actions/check.go`, and `pkg/container/client.go`.

---

## Files

### `filters.go`

Defines a set of filter constructor functions and the `BuildFilter` convenience function that composes them into a
single `types.Filter` for use in an update session.

All filter functions follow the same pattern: they accept a `baseFilter types.Filter` and return a new `types.Filter`
that first applies its own logic and, if the container passes, delegates to `baseFilter`. This allows filters to be
stacked without either function needing knowledge of the other.

---

**Public Functions:**

---

#### `WatchtowerContainersFilter(c types.FilterableContainer) bool`

A pre-built filter (not a constructor) that returns `true` only for containers identified as Watchtower instances via
`c.IsWatchtower()`. Used directly by `internal/actions/check.go` when searching for other running Watchtower containers
to clean up.

---

#### `NoFilter(types.FilterableContainer) bool`

A pre-built pass-through filter that always returns `true`. Used as the base filter when no other constraints apply, and
as the starting point when composing filter chains via `BuildFilter`.

---

#### `FilterByNames(names []string, baseFilter types.Filter) types.Filter`

Returns a filter that passes only containers whose name matches at least one entry in `names`. If `names` is empty,
returns `baseFilter` unchanged.

Name matching supports two modes for each entry:

- **Exact match**: The entry is compared directly against `c.Name()` and `c.Name()[1:]` (stripping the leading `/` that
  Docker prepends to container names).
- **Regex match**: If the entry compiles as a regular expression, it is tested against the full container name. The
  match must span the entire name (start index ≤ 1 and end index ≥ `len(name)-1`) to avoid partial substring matches.

If a container passes the name check, `baseFilter` is called and its result is returned.

---

#### `FilterByDisableNames(disableNames []string, baseFilter types.Filter) types.Filter`

Returns a filter that excludes containers whose name exactly matches any entry in `disableNames`. If `disableNames` is
empty, returns `baseFilter` unchanged. Matching uses the same exact-match logic as `FilterByNames` (both with and
without the leading `/`). Containers that do not match any disabled name are passed to `baseFilter`.

---

#### `FilterByEnableLabel(baseFilter types.Filter) types.Filter`

Returns a filter that passes only containers for which the `com.centurylinklabs.watchtower.enable` label is present
(regardless of its value). Containers where the label is absent (`ok == false` from `c.Enabled()`) are excluded. Used
when `--label-enable` is set, to restrict monitoring to explicitly opted-in containers.

---

#### `FilterByDisabledLabel(baseFilter types.Filter) types.Filter`

Returns a filter that excludes containers where the `com.centurylinklabs.watchtower.enable` label is explicitly set to
`false`. Containers where the label is absent or set to `true` are passed through to `baseFilter`. Applied
unconditionally by `BuildFilter` as the final stage in every filter chain, providing a universal opt-out mechanism via
label.

---

#### `FilterByScope(scope string, baseFilter types.Filter) types.Filter`

Returns a filter that passes only containers whose scope matches the given `scope` string. A container with no scope
label, or with an empty scope value, is treated as having the scope `"none"`. A container passes if its effective scope
equals `scope`. Used to implement multi-instance Watchtower deployments where each instance manages a distinct subset of
containers.

---

#### `FilterByImage(images []string, baseFilter types.Filter) types.Filter`

Returns a filter that passes only containers whose image name (stripped of its tag) matches at least one entry in
`images`. If `images` is `nil`, returns `baseFilter` unchanged. Used by the HTTP API update handler to restrict an
on-demand update to specific images.

---

#### `BuildFilter(names []string, disableNames []string, enableLabel bool, scope string) (types.Filter, string)`

Composes a complete filter from the supplied parameters and returns both the combined `types.Filter` and a
human-readable description string suitable for logging at startup. The filter chain is constructed in the following
order:

1. `NoFilter` as the base.
2. `FilterByNames` — if `names` is non-empty.
3. `FilterByDisableNames` — if `disableNames` is non-empty.
4. `FilterByEnableLabel` — if `enableLabel` is `true`.
5. `FilterByScope` — if `scope` is `"none"` or any non-empty value.
6. `FilterByDisabledLabel` — always applied as the final stage.

The description string begins with `"Checking all containers (except explicitly disabled with label)"` when no
constraints are active, or `"Only checking containers ..."` with a summary of the active constraints when any are set.

---

## Test Coverage

`filters_test.go` tests each filter function independently using the `FilterableContainer` mock from
`pkg/container/mocks`:

| Test | Description |
|---|---|
| `TestWatchtowerContainersFilter` | Verifies that `WatchtowerContainersFilter` returns `true` for a container where `IsWatchtower()` returns `true`. |
| `TestNoFilter` | Verifies that `NoFilter` always returns `true`. |
| `TestFilterByNames` | Verifies that `FilterByNames` returns `baseFilter` unchanged for an empty name list, passes a container whose name matches, and rejects a container whose name does not match. |
| `TestFilterByNamesRegex` | Verifies regex name matching: a container matching the full pattern passes, a container not matching the pattern is rejected, and a container where the pattern matches only a substring is also rejected. |
| `TestFilterByEnableLabel` | Verifies that containers with the enable label set to `true` or `false` both pass (label is present), while containers without the label are rejected. |
| `TestFilterByScope` | Verifies that a container with the matching scope passes, a container with a different scope is rejected, and a container with no scope label is rejected. |
| `TestFilterByNoneScope` | Verifies the special `"none"` scope: containers with any explicit non-none scope are rejected, while containers with no scope label, an empty scope, or `scope=none` all pass. |
| `TestBuildFilterNoneScope` | Integration test for `BuildFilter` with `scope="none"`: verifies that scoped containers are rejected and unscoped containers pass. |
| `TestFilterByDisabledLabel` | Verifies that `FilterByDisabledLabel` rejects containers with the enable label explicitly set to `false`, passes containers with the label set to `true`, and passes containers where the label is absent. |
| `TestFilterByImage` | Verifies that a nil image list returns `baseFilter` unchanged; that containers whose image name matches one of the filter images pass; and that containers with non-matching image names are rejected. Also verifies multi-image filter behaviour. |
| `TestBuildFilter` | Integration test for `BuildFilter` with a name list: verifies the description string contains all names joined with `"or"`, and tests the interaction between name matching and the disabled-label filter. |
| `TestBuildFilterEnableLabel` | Integration test for `BuildFilter` with `enableLabel=true`: verifies the description contains `"using enable label"` and that both name and enable-label constraints must be satisfied. |
| `TestBuildFilterDisableContainer` | Integration test for `BuildFilter` with a `disableNames` list: verifies the description contains `"not named"` and all excluded names, that listed containers are rejected even when the enable label is set, and that substring matches are not treated as full-name matches. |
