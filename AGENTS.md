# Agent guide

This is a disposable local kind environment. Read [README.md](README.md) for settings and lifecycle commands. Tool versions are in `mise.toml` (Task 3.51.1). Docker or Podman, kind, kubectl, jq, and POSIX utilities are prerequisites.

## Structure

- `Taskfile.yaml` owns sequential `up`, `down`, `reset`, and `status` orchestration.
- `taskfiles/registries.yaml` owns registry containers and their anonymous volumes.
- `taskfiles/cluster.yaml` owns cluster creation, mirror configuration, and registry discovery.

Both includes must work independently, including remotely without companion files. The default kind configuration lives inline in `taskfiles/cluster.yaml` and is generated on stdin. `KIND_CONFIG_FILE` optionally selects a configuration supplied by the consuming repository; do not add a required companion configuration file.

## Keep it small and readable

Prefer direct commands, Task `status:`, `requires:`, simple preconditions, and explicit operation-specific helpers. Accept small YAML repetition. Do not add action dispatchers, exhaustive input validators, configuration inspection frameworks, or drift reconciliation. Configuration is trusted developer input; let the underlying tools report invalid settings.

Reuse existing resources unchanged. Use `reset` for changed settings or broken state. Teardown must not compare creation settings. Reset with the runtime, cluster name, network, and container names used at creation, then apply changed settings with `up`.

Defaults belong in each included file's top-level `vars:`. Scalars use template defaults, such as `CLUSTER_NAME: '{{.CLUSTER_NAME | default "opendefence-dev"}}'`. Do not copy defaults into tasks or root include entries. Keep the intentionally duplicated cache maps and runtime detectors aligned. The maps are not scalar overrides. Runtime detection prefers real Docker, then Podman, and treats podman-docker as Podman.

Public tasks need descriptions and relevant tool preconditions. Helpers are internal and declare required caller variables. Keep lifecycle operations sequential. Mirror configuration and ConfigMap publication must not implicitly create a cluster. Keep `NODES` on `configure-nodes`, after cluster creation; do not cache mutable container state in dynamic variables.

## Essential safety only

Quote shell arguments. Use exact, regex-escaped container-name filters. Keep query errors distinct from resource absence. Registry ownership checks are command-running dependencies so status/force handling cannot skip them. Mutations select owned container IDs rather than unchecked names.

Keep owner/network labels and ownership checks. Default both includes to `kind-<CLUSTER_NAME>`. Teardown only the selected cluster and owned registries; do not inspect other clusters or network users. `down` keeps registry data; `reset` removes owned containers and their anonymous volumes. Never globally prune or delete shared networks. Run lifecycle operations sequentially.

## Registry configuration

Use containerd hosts files for registry configuration. The generated configuration enables `CERTS_D_DIR`; custom kind configuration is responsible for matching it. The registry port configures listeners, published ports, and node endpoints. Local-registry hosts keep default push/pull/resolve capabilities; cache hosts use pull/resolve with upstream fallback. Preserve atomic hosts-file writes and correct heredoc indentation.

## Syntax-only validation

The user explicitly excludes behavioral, compatibility, integration, and destructive lifecycle tests. Do not add a test harness or run lifecycle commands for verification.

```sh
task --list
task -t taskfiles/registries.yaml --list
task -t taskfiles/cluster.yaml --list
```

Run the relevant syntax check after each edit and all three before finishing. YAML/shell parsing without execution is also allowed. Task dry runs are not syntax-only checks. Editor schema associations are in `.vscode/settings.json`.
