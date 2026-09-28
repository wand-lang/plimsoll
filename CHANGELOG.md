# Changelog

## [0.1.0] - 2026-09-28

The first release. plimsoll makes Kubernetes objects typed wand values,
with the types generated from the cluster's own OpenAPI v3 schema. It needs
wand 0.90.0 or later.

### Added

- **`gen` writes the types.** It reads `/openapi/v3` from the API server
  that kubectl points at, and writes one module for each group-version:
  `k8s/apps/v1.wand`, `k8s/core/v1.wand`, and so on. With no arguments it
  reads every group-version that the cluster serves, CRDs included. A CRD's
  nested objects get records of their own (`WidgetSpec`). A key that a
  field name cannot be, such as `$ref`, is kept as the field's document
  key. `gen` removes the modules that it did not write.
- **Objects are records.** Required fields must be given, optional fields
  are `Option` and default to `None`, and `apiVersion` and `kind` are
  filled in. An enum is written with its type:
  `core.ContainerImagePullPolicy.IfNotPresent`. wand derives the JSON
  encoder and decoder of each type.
- **The cluster, through kubectl.** `apply!` (server-side apply, field
  manager `plimsoll`), `check!` (a server-side dry run that gives the
  object the server would store), `get!`, `list!` and `delete!`, the `_in`
  forms that name a namespace, and `manifest`. Each one that can fail has
  a plain form that gives a `Result`. A rehearsal (`wand --dry-run`) runs
  `get!`, `list!` and `check!`, and withholds `apply!` and `delete!`.
- **Quantities.** `millicores "250m"` is 250 and `bytes "128Mi"` is
  134217728, counted exactly.
- **An App in a few fields.** `app.App(name, image, port, replicas, env,
  cpu, memory)` makes a Deployment and a Service with one set of labels.
  Change either object with a record update before you apply it.
- **`check` and `upgrade` keep the types in step with the clusters.**
  `check --contexts a,b` compares `k8s/` with each cluster and names each
  type, field and enum value that differs. `upgrade` writes `k8s/` from
  the oldest cluster, typechecks your scripts, and reports each script
  that the new schema breaks.

The committed `k8s/` modules come from Kubernetes v1.37.0.
