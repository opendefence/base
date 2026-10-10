# Agent guide

This repository packages the platform operators for OpenDefence core as a Zarf package and ships a disposable local kind environment for developing it. Read [README.md](README.md) for settings and lifecycle commands. Tool versions are in `mise.toml` (Task 3.51.1). Docker or Podman, kind, kubectl, Zarf 0.87.0, jq, and POSIX utilities are prerequisites.

## Structure

- `Taskfile.yaml` owns sequential `up`, `down`, `reset`, and `status` orchestration.
- `taskfiles/registries.yaml` owns registry containers and their anonymous volumes.
- `taskfiles/cluster.yaml` owns cluster creation, mirror configuration, and registry discovery.
- `taskfiles/base.yaml` owns connected package install/remove/status and public CA export; it is remote-safe without companion files.
- `taskfiles/build.yaml` owns checkout-local Zarf dev deploy, lint, inspect, image discovery, and package creation.
- `zarf.yaml` owns the base operator package; `helm-values/`, `manifests/`, and `values/` hold its chart configuration, raw resources, and deployment values.

The registry, cluster, and base includes must work independently, including remotely without companion files. The build include requires the checkout. The default kind configuration lives inline in `taskfiles/cluster.yaml` and is generated on stdin. `KIND_CONFIG_FILE` optionally selects a configuration supplied by the consuming repository; do not add a required companion configuration file.

## Keep it small and readable

Prefer direct commands, Task `status:`, `requires:`, simple preconditions, and explicit operation-specific helpers. Accept small YAML repetition. Do not add action dispatchers, exhaustive input validators, configuration inspection frameworks, or drift reconciliation. Configuration is trusted developer input; let the underlying tools report invalid settings.

Reuse existing cluster and registry resources unchanged. Base package redeploys are Helm upgrades. Use `reset` for changed cluster/registry settings or broken state. Teardown must not compare creation settings. Reset with the runtime, cluster name, network, and container names used at creation, then apply changed settings with `up`.

Defaults belong in each included file's top-level `vars:`. Scalars use template defaults, such as `CLUSTER_NAME: '{{.CLUSTER_NAME | default "opendefence-dev"}}'`. Do not copy defaults into tasks or root include entries. Keep the intentionally duplicated cache maps and runtime detectors aligned. The maps are not scalar overrides. Runtime detection prefers real Docker, then Podman, and treats podman-docker as Podman.

Public tasks need descriptions and relevant tool preconditions. Helpers are internal and declare required caller variables. Keep lifecycle operations sequential. Mirror configuration and ConfigMap publication must not implicitly create a cluster. Keep `NODES` on `configure-nodes`, after cluster creation; do not cache mutable container state in dynamic variables.

## Essential safety only

Quote shell arguments. Use exact, regex-escaped container-name filters. Keep query errors distinct from resource absence. Registry ownership checks are command-running dependencies so status/force handling cannot skip them. Mutations select owned container IDs rather than unchecked names.

Keep owner/network labels and ownership checks. Default registry and cluster networks and Zarf task kube-contexts to `kind-<CLUSTER_NAME>`. Teardown only the selected cluster and owned registries; do not inspect other clusters or network users. `down` keeps registry data; `reset` removes owned containers and their anonymous volumes. Never globally prune or delete shared networks. Run lifecycle operations sequentially.

## Registry configuration

Use containerd hosts files for registry configuration. The generated configuration enables `CERTS_D_DIR`; custom kind configuration is responsible for matching it. The registry port configures listeners, published ports, and node endpoints. Local-registry hosts keep default push/pull/resolve capabilities; cache hosts use pull/resolve with upstream fallback. Preserve atomic hosts-file writes and correct heredoc indentation.

## Zarf conventions

Component order is dependency order: cloudnative-pg, cert-manager, external-secrets, traefik-crds, trust-manager, pki, linkerd-identity, linkerd, traefik. Preserve Helm/kstatus waits and identity Certificate/ConfigMap health checks before the Linkerd control plane. Identity manifests must precede the Linkerd charts in a separate component because charts install before manifests within a component.

Chart pins live only in `zarf.yaml`. Use native charts and remote release manifests, not pre-rendered Helm output, metadata stripping, or a separate `dev.yaml`. Only our small public PKI manifest uses `template: true`; never template operator bundles. Deployment configuration uses Zarf package values only; do not add legacy `variables:` or `###ZARF_VAR_###` templating. Define every referenced package values key in `values/values.yaml`, keep Let's Encrypt as the baked issuer default, and keep baked defaults schema-valid because lint/package creation validate them. Missing deployment input fails schema validation rather than prompting. `values/local-dev.yaml` selects CA mode. Keep stable `public-issuer` and `internal-root-issuer` names. Domains, hostname Certificates, and Traefik's default TLSStore are core concerns; do not add them here.

Use connected mode for development and base task deployments without initializing Zarf or pushing images. Before claiming offline support, discover and add the component image inventory. Zarf uses the current kube-context, so retain explicit context preconditions on deploy/remove/status. Do not modify `kubernetes-test/`; it is source reference material, not part of this package.

## Syntax-only validation

The user explicitly excludes behavioral, compatibility, integration, and destructive lifecycle tests. Do not add a test harness or run lifecycle commands for verification.

```sh
task --list
task -t taskfiles/registries.yaml --list
task -t taskfiles/cluster.yaml --list
task -t taskfiles/base.yaml --list
task -t taskfiles/build.yaml --list
zarf dev lint .
```

Run the relevant syntax check after each edit and all five Task list checks plus Zarf lint before finishing. YAML/JSON/shell parsing without execution is also allowed. The Go-templated manifest must be rendered before YAML parsing and is excluded from Prettier via `.prettierignore`. Optional `zarf dev inspect manifests` review with local-dev values and ACME defaults is permitted, without a cluster; it downloads sources but does not deploy. Task dry runs are not syntax-only checks. Editor schema associations are in `.vscode/settings.json`.
