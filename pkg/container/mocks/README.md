# `pkg/container/mocks` Package

This package is a test support library. It provides mock implementations of container-related interfaces, a mock HTTP API server that simulates the Docker daemon, reference types for describing containers in tests, and a directory of JSON fixture files that represent real Docker API responses. It has no production use and is imported exclusively by test files in `pkg/container`, `pkg/filters`, `pkg/registry`, and `internal/actions`.

---

## Files

### `ApiServer.go`

Provides a collection of `http.HandlerFunc` constructors that simulate specific Docker API endpoints using [`gomega/ghttp`](https://pkg.go.dev/github.com/onsi/gomega/ghttp). Tests compose these handlers onto a `ghttp.Server` to create a mock Docker daemon without needing a real Docker installation.

Also declares the canonical set of named `ContainerRef` variables representing the containers available in the fixture data, and defines the image references they depend on.

---

**Package-level Variables:**

| Variable | Type | Description |
|---|---|---|
| `defaultImage` | `imageRef` | The default image used by watchtower containers in fixtures (`sha256:4dbc5f...`, tagged `portainer/portainer:latest`). |
| `Watchtower` | `ContainerRef` | A running watchtower container fixture. |
| `Stopped` | `ContainerRef` | A stopped (exited) container fixture. |
| `Running` | `ContainerRef` | A running non-watchtower container fixture (watchtower image). |
| `Restarting` | `ContainerRef` | A restarting container fixture. |
| `NetConsumerOK` | `ContainerRef` | A container using `network_mode: container:...` with a resolvable network supplier. |
| `NetConsumerInvalidSupplier` | `ContainerRef` | A container referencing a network supplier container that does not exist. |
| `NetSupplierNotFoundID` | `string` (const) | The container ID used for the missing network supplier, for assertion in tests. |
| `NetSupplierContainerName` | `string` (const) | The container name of the network supplier, for assertion in tests. |

---

**Public Functions:**

---

#### `RespondWithJSONFile(relPath string, statusCode int, optionalHeader ...http.Header) http.HandlerFunc`

Returns a `ghttp` response handler that reads the JSON file at `relPath` and responds with it at the given HTTP status code. Calls `gomega.ExpectWithOffset` to fail the test immediately if the file cannot be read. An optional set of response headers may be supplied.

---

#### `GetContainerHandlers(containerRefs ...*ContainerRef) []http.HandlerFunc`

Accepts one or more `ContainerRef` values and returns a slice of `http.HandlerFunc` values suitable for appending to a `ghttp.Server`. For each ref it produces:

1. A handler for `GET /containers/{id}/json` that responds with the container's fixture JSON (or a 404 if `isMissing` is set).
2. Handlers for any containers that the ref directly references (e.g. a network supplier), one level deep.
3. A handler for `GET /images/{imageID}/json` that responds with the image fixture JSON.

This is the primary entry point for setting up a mock Docker daemon in container client tests.

---

#### `GetContainerHandler(containerID string, containerInfo *types.ContainerJSON) http.HandlerFunc`

Returns a handler for the `GET /containers/{id}/json` endpoint. If `containerInfo` is non-nil, responds with it as JSON at HTTP 200. If `containerInfo` is nil, responds with a 404 not-found response.

---

#### `GetImageHandler(imageInfo *types.ImageInspect) http.HandlerFunc`

Returns a handler for the `GET /images/{id}/json` endpoint that responds with the provided `imageInfo` as JSON at HTTP 200.

---

#### `ListContainersHandler(statuses ...string) http.HandlerFunc`

Returns a handler for the `GET /containers/json` endpoint. Reads `mocks/data/containers.json`, filters the container list to only those whose `State` field matches one of the provided `statuses`, and responds with the filtered list. Verifies that the request's `filters` query parameter matches the expected filter arguments.

---

#### `KillContainerHandler(containerID string, found FoundStatus) http.HandlerFunc`

Returns a handler for the `POST /containers/{id}/kill` endpoint. Responds with HTTP 204 No Content if `found` is `Found`, or HTTP 404 if `found` is `Missing`.

---

#### `RemoveContainerHandler(containerID string, found FoundStatus) http.HandlerFunc`

Returns a handler for the `DELETE /containers/{id}` endpoint. Responds with HTTP 204 No Content if `found` is `Found`, or HTTP 404 if `found` is `Missing`.

---

#### `RemoveImageHandler(imagesWithParents map[string][]string) http.HandlerFunc`

Returns a handler for the `DELETE /images/{id}` endpoint. Extracts the image ID from the request URL and looks it up in `imagesWithParents`. If found, responds with a JSON array of `ImageDeleteResponseItem` entries covering the image and its parents. If not found, responds with HTTP 404.

---

**Types:**

#### `FoundStatus`

A boolean type alias used to make handler construction calls self-documenting.

| Constant | Value | Description |
|---|---|---|
| `Found` | `true` | The resource exists; respond with success. |
| `Missing` | `false` | The resource does not exist; respond with 404. |

---

### `FilterableContainer.go`

An auto-generated [testify/mock](https://pkg.go.dev/github.com/stretchr/testify/mock) implementation of the `types.FilterableContainer` interface. Used by tests in `pkg/filters` and `pkg/registry` to verify filter logic without constructing real container objects.

**Type:**

#### `FilterableContainer`

Embeds `mock.Mock` and implements all methods of `types.FilterableContainer`. Each method delegates to testify's `Called()` mechanism, allowing tests to set up expectations and return values with `On(...)` and assert them with `AssertExpectations(t)`.

| Method | Return type | Description |
|---|---|---|
| `Enabled()` | `(bool, bool)` | Returns whether the container's enable label is set and its value. |
| `IsWatchtower()` | `bool` | Returns whether the container is a Watchtower instance. |
| `Name()` | `string` | Returns the container name. |
| `Scope()` | `(string, bool)` | Returns the container's scope label value and whether it was set. |
| `ImageName()` | `string` | Returns the container's image name. |

---

### `container_ref.go`

Defines the `ContainerRef` and `imageRef` types used to describe mock containers declaratively. These types drive the handler generation in `ApiServer.go` and map container names to their fixture JSON files.

**Types:**

#### `imageRef` _(unexported)_

Describes a Docker image used by a mock container.

| Field | Type | Description |
|---|---|---|
| `id` | `types.ImageID` | The full SHA256 image ID. |
| `file` | `string` | The base filename (without path or extension) of the image's JSON fixture under `mocks/data/`. |

#### `ContainerRef`

Describes a mock container, including how to locate its fixture data and any containers it references.

| Field | Type | Description |
|---|---|---|
| `name` | `string` | The container name. |
| `id` | `types.ContainerID` | The container ID. |
| `image` | `*imageRef` | The image used by this container. |
| `file` | `string` | Optional override for the fixture filename. Defaults to `name` if empty. |
| `references` | `[]*ContainerRef` | Other containers that this container references (e.g. a network supplier). |
| `isMissing` | `bool` | If `true`, the API server will respond with a 404 for this container. |

**Public Methods:**

---

##### `(cr *ContainerRef) ContainerID() types.ContainerID`

Returns the container's ID. Used by tests to retrieve the ID of a named fixture for use in API handler setup and assertions.

---

## `data/` Fixture Files

The `data/` subdirectory contains JSON files that represent real Docker API responses, captured from a live Docker daemon. They are loaded by `RespondWithJSONFile` and `ListContainersHandler` at test runtime.

**Container fixtures** (`GET /containers/{id}/json`):

| File | Container | Description |
|---|---|---|
| `container_watchtower.json` | `watchtower` | A running watchtower container. |
| `container_running.json` | `running` | A running portainer container. |
| `container_stopped.json` | `stopped` | An exited watchtower container. |
| `container_restarting.json` | `restarting` | A restarting container. |
| `container_net_supplier.json` | `net_supplier` | A running gluetun VPN container acting as a network supplier. |
| `container_net_consumer.json` | `net_consumer` | An nginx container using `network_mode: container:...` with a valid supplier. |
| `container_net_consumer-missing_supplier.json` | `net_consumer` | An nginx container referencing a non-existent network supplier. |

**Image fixtures** (`GET /images/{id}/json`):

| File | Image | Description |
|---|---|---|
| `image_default.json` | `portainer/portainer:latest` | Default image used by the `Watchtower`, `Stopped`, and `Restarting` container fixtures. |
| `image_running.json` | `containrrr/watchtower:latest` | Image used by the `Running` container fixture. |
| `image_net_producer.json` | `qmcgaw/gluetun:latest` | Image used by the network supplier container fixture. |
| `image_net_consumer.json` | `nginx:latest` | Image used by the network consumer container fixture. |

**Container list fixture** (`GET /containers/json`):

| File | Description |
|---|---|
| `containers.json` | An array of all available containers in summary form, used by `ListContainersHandler` and filtered by status before being returned. |
