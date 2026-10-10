# OpenDefence development base

A disposable kind cluster with a local push/pull registry and Docker Hub and GHCR pull-through caches.

## Usage

Have Docker or Podman available, then install the required tooling with mise:

```sh
mise install
task up
task status
task down
task reset
```

- `up` starts registries, creates the cluster, connects registries, writes mirrors, and publishes registry discovery.
- `down` deletes the selected cluster, leaving registry containers and data intact.
- `reset` also removes owned registry containers and their anonymous volumes. **Local images and cached data are deleted.**
- Existing resources are reused as-is.

### Short aliases

`c` aliases the `cluster` include, and `r` aliases `registries`. All task names work with either namespace.

| Shortcut                         | Equivalent                                                         |
| -------------------------------- | ------------------------------------------------------------------ |
| `task c:up`                      | `task cluster:create` (also `cluster:up`), create the cluster only |
| `task c:down`                    | `task cluster:down`                                                |
| `task c:mirrors`                 | `task cluster:configure-mirrors`                                   |
| `task c:publish-registry`        | `task cluster:document-local-registry`                             |
| `task r:up`                      | `task registries:up`                                               |
| `task r:local`                   | `task registries:start-local`                                      |
| `task r:caches`                  | `task registries:start-caches`                                     |
| `task s`, `task c:s`, `task r:s` | Root, cluster, or registry status                                  |

Use `task up` for the complete stack. `task r:down` removes registry containers and data.

## Configuration

```sh
task reset
task up REGISTRY_PORT=5001
```

| Setting                       | Default                                        |
| ----------------------------- | ---------------------------------------------- |
| `CLUSTER_NAME`                | `opendefence-dev`                              |
| `KIND_NETWORK`                | `kind-<CLUSTER_NAME>`                          |
| `REGISTRY_PORT`               | `5000`                                         |
| `LOCAL_REGISTRY_NAME`         | `local-registry`                               |
| `LOCAL_REGISTRY_BIND_ADDRESS` | `127.0.0.1`                                    |
| `REGISTRY_IMAGE`              | `registry:3`                                   |
| `REGISTRY_RESTART_POLICY`     | `always`                                       |
| `CERTS_D_DIR`                 | `/etc/containerd/certs.d`                      |
| `KIND_CONFIG_FILE`            | empty, generate minimal configuration on stdin |
| `KUBE_CONTEXT`                | `kind-<CLUSTER_NAME>`                          |
| `CONTAINER_RUNTIME`           | detected, or explicitly `docker` / `podman`    |

```sh
task up CONTAINER_RUNTIME=podman
task up KIND_CONFIG_FILE=/absolute/path/to/custom-kind.yaml
```

Custom kind configuration must enable containerd's hosts directory at `CERTS_D_DIR`. Non-loopback registry bindings expose an unauthenticated registry.

Caches are defined in `PULL_THROUGH_CACHES` in both included Taskfiles. Each Taskfile also works independently.

## Cleanup

Cleanup only affects the selected cluster, its owned registries, and their anonymous volumes. Shared networks and unrelated resources are left alone. Run lifecycle commands one at a time.

## Registry configuration

Push and pull local images using `localhost:<REGISTRY_PORT>`. Cache mirrors pull from Docker Hub and GHCR, with upstream fallback. Mirror configuration and registry discovery require an existing cluster.

With default settings, each kind node has these files under `/etc/containerd/certs.d`:

`localhost:5000/hosts.toml`

```toml
[host."http://local-registry:5000"]
```

`docker.io/hosts.toml`

```toml
server = "https://registry-1.docker.io"

[host."http://proxy-dockerhub:5000"]
  capabilities = ["pull", "resolve"]
```

`ghcr.io/hosts.toml`

```toml
server = "https://ghcr.io"

[host."http://proxy-ghcr:5000"]
  capabilities = ["pull", "resolve"]
```

## Syntax checks

```sh
task --list
task -t taskfiles/registries.yaml --list
task -t taskfiles/cluster.yaml --list
```

These checks do not run lifecycle commands. `task --dry` is not a syntax check.
