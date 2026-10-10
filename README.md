# OpenDefence base

The platform operators for OpenDefence core, packaged as a native Zarf package: CloudNativePG, cert-manager, External Secrets, trust-manager, Linkerd, Traefik, and the PKI that ties them together. The repository also carries a disposable local kind environment with a push/pull registry and Docker Hub and GHCR pull-through caches for developing against the package.

## Development usage

Have Docker or Podman and POSIX utilities available, then install kind, kubectl, Task, and Zarf with mise or manually. Ports 80 and 443 must be free.

```sh
mise install
task up
task status
task down
task reset
```

- `up` (alias `dev`) starts registries, creates the cluster, connects registries, writes mirrors, publishes registry discovery, and deploys the base with `values/local-dev.yaml`.
- `down` deletes the selected cluster, leaving registry containers and data intact.
- `reset` also removes owned registry containers and their anonymous volumes. **Local images and cached data are deleted.**
- Existing cluster and registry resources are reused as-is; redeploying the base upgrades its Helm releases.

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

Custom kind configuration must enable containerd's hosts directory at `CERTS_D_DIR` and map node TCP ports 80/443 to host ports 80/443 on `127.0.0.1`. The generated configuration already does both. Non-loopback registry bindings expose an unauthenticated registry.

Caches are defined in `PULL_THROUGH_CACHES` in both registry and cluster Taskfiles. Those includes and `taskfiles/base.yaml` work independently and remotely without companion files. `taskfiles/build.yaml` requires this checkout.

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

## Base package

`zarf.yaml` is the single package definition, named `base`. Components deploy sequentially: CloudNativePG, cert-manager, External Secrets, Traefik/Gateway API CRDs, trust-manager, PKI, Linkerd identity, Linkerd, then Traefik. Chart pins live in `zarf.yaml`; chart configuration lives in `helm-values/`, and OpenDefence resources are raw YAML in `manifests/`. Only `manifests/public-pki.yaml` uses Go templates.

The downstream contract is:

- `ClusterIssuer/public-issuer` always selects the public certificate issuer.
- `ClusterIssuer/internal-root-issuer` signs mesh identity certificates from the internal root in `cert-manager`.
- `issuer.type: ca` creates `public-root-secret` in `cert-manager`, the local root that `public-issuer` signs with. Hostnames, their `Certificate` objects, and Traefik's default `TLSStore` belong to core, which requests them from `public-issuer`.
- Traefik uses hostPorts 80/443, a ClusterIP service, HTTPS redirect, JSON access logs, Linkerd injection, and a Recreate strategy. No Go plugins or entrypoint middlewares are installed.

Export the local public CA and import the resulting PEM into your browser or operating system trust store:

```sh
task base:export-ca > public-root-ca.pem
```

### Values and deployment

Deployment configuration is Zarf package values, templated into `manifests/public-pki.yaml` at deploy time. `values/values.yaml` bakes the Let's Encrypt production defaults into the package (`issuer.type: acme`, production ACME server, contact `letsencrypt@opendefence.fi`), so a deploy with no values file uses Let's Encrypt. Override with `--values <file>` or `--set-values`, for example your own contact or `issuer.type: ca`. There is no interactive prompt: `values/values.schema.json` requires a nonempty ACME email in `acme` mode, and a deploy that violates it fails validation before anything is applied. `values/local-dev.yaml` selects `ca`.

```sh
task build:deploy
zarf dev deploy . --connected --values values/local-dev.yaml
zarf dev deploy . --connected --set-values issuer.acme.email=operator@example.org
task build:lint
task build:inspect
task build:find-images
task build:package ARCH=amd64 BUILD_DIR=.build
```

Local development uses `zarf dev deploy` in connected mode without `zarf init` or image pushes; nodes pull through registry mirrors where configured. `zarf package deploy` installs a built archive or OCI package. The current definition targets connected deployment; before building a fully offline package, use `build:find-images` to populate component `images:` lists, then create the package and initialize the target with Zarf. No offline image inventory is baked in yet.

```sh
task base:install BASE_PACKAGE=/absolute/path/to/zarf-package-base-amd64.tar.zst
task base:install BASE_VERSION=<published-version>
task base:remove
```

`base:install` defaults to `oci://ghcr.io/opendefence/base:<BASE_VERSION>`. A version must already be published; this change does not publish a package. Override `BASE_PACKAGE` to use a local archive or another OCI reference. `base:remove` removes the package without requiring its creation settings. Root `down` and `reset` still delete the cluster rather than separately removing the package.

| Base setting       | Default                                                 |
| ------------------ | ------------------------------------------------------- |
| `BASE_VERSION`     | empty, required unless `BASE_PACKAGE` is overridden     |
| `BASE_PACKAGE`     | `oci://ghcr.io/opendefence/base:<BASE_VERSION>`         |
| `BASE_ISSUER`      | empty, package default (`acme`); `ca` for local CA mode |
| `BASE_VALUES_FILE` | empty, optional deployment values file                  |
| `KUBE_CONTEXT`     | `kind-<CLUSTER_NAME>`                                   |

Package values precedence is baked defaults, deployment values files, then `--set-values`. `base:install` passes `issuer.type` through `--set-values` only when `BASE_ISSUER` is set, so it wins over `BASE_VALUES_FILE`; everything else, such as the ACME contact or server, comes from the values file. Build tasks use `VALUES_FILE` (default `values/local-dev.yaml`), `BUILD_DIR` (default `.build`), and `ARCH` (default `amd64`). Zarf has no kube-context flag: deploy, remove, and status guard that the current context equals `KUBE_CONTEXT`; switch it explicitly before running them.

### Remote inclusion

Once these taskfiles are published, set `BASE_TASKFILE` to the raw URL of `taskfiles/base.yaml` at a pinned repository ref. A consuming Taskfile can include it without this checkout:

```yaml
version: "3"
includes:
  base:
    taskfile: "{{.BASE_TASKFILE}}"
vars:
  BASE_VERSION: <published-version>
  BASE_ISSUER: ca
```

Run `task base:install`, or override `BASE_PACKAGE` with an archive/OCI reference. For non-kind targets, set `KUBE_CONTEXT` explicitly. The registry and cluster includes remain remote-safe; the build include is repo-local.

## GitHub Actions

Do not edit `.github/workflows/actions.lock` by hand. When a workflow adds or removes a `uses` dependency, update the lock:

```sh
gh actions-lock
```

## Syntax checks

```sh
task --list
task -t taskfiles/registries.yaml --list
task -t taskfiles/cluster.yaml --list
task -t taskfiles/base.yaml --list
task -t taskfiles/build.yaml --list
zarf dev lint .
```

These checks do not run lifecycle commands. `task --dry` is not a syntax check. YAML/JSON parsing is also permitted. Optional render-only review uses `task build:inspect` (CA mode) and `zarf dev inspect manifests .` (Let's Encrypt defaults); these download sources and render templates without changing a cluster.
