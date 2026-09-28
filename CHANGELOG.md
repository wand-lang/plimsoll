# Changelog

## [Unreleased]

### Added

- **`gen --context name` reads the cluster of a kubectl context.** Before,
  `gen` read only the current context, and you had to switch contexts
  first. `check` and `upgrade` already take `--contexts`.

  ```sh
  wand github.com/wand-lang/plimsoll/cli gen --context prod
  ```

## [0.2.0] - 2026-09-28

This release needs wand 0.91.1 or later.

### Changed

- **Every Kubernetes enum is now a type, and 23 fields changed from
  `String` to one.** In 0.1.0, 17 sets of enum values stayed `String`
  because some of their values cannot be wand constructor names, such as
  `None`, `*` or `client auth`. Now each value gets a name, and the type
  keeps the real value for documents:

  ```
  type PodSpecDnsPolicy = ClusterFirst | ClusterFirstWithHostNet | Default | None_ "None"
  type NamedRuleWithOperationsOperations = All "*" | CONNECT | CREATE | DELETE | UPDATE
  ```

  A value that is a wand name or the name of a type gets a trailing `_`:
  `None_`, `IPv4_`. Any other value gets a name made from its parts:
  `"client auth"` is `ClientAuth`, `"SIGRTMAX-1"` is `SIGRTMAX1`, `"*"` is
  `All` and `""` is `Empty`.

  **This breaks scripts that set these fields as strings.** Write the
  constructor instead:

  ```
  dnsPolicy = Some "None"                        -- 0.1.0
  dnsPolicy = Some core.PodSpecDnsPolicy.None_   -- 0.2.0
  ```

  Run `wand t` over your scripts to find each place. The fields are:

  | Module | Fields |
  |---|---|
  | `admissionregistration/v1` | `MutatingWebhook.sideEffects`, `ValidatingWebhook.sideEffects`, `Mutation.patchType`, `NamedRuleWithOperations.operations` and `.scope`, `RuleWithOperations.operations` and `.scope` |
  | `certificates/v1` | `CertificateSigningRequestSpec.usages` |
  | `core/v1` | `AzureDiskVolumeSource.cachingMode` and `.kind`, `ContainerStatus.stopSignal`, `Lifecycle.stopSignal`, `HostPathVolumeSource.type`, `PodSpec.dnsPolicy`, `ServiceSpec.ipFamilies` and `.sessionAffinity`, `VolumeMount.mountPropagation` |
  | `discovery/v1` | `EndpointSlice.addressType` |
  | `networking/v1` | `NetworkPolicySpec.policyTypes` |
  | `resource/v1` | `DeviceRequestAllocationResult.skipNodeOperations`, `ResourceSliceSpec.skipNodeOperations`, `DeviceTaint.effect`, `DeviceToleration.effect` |

  If you generate your own types with `gen`, generate them again with
  plimsoll 0.2.0 to get the same change.

- **Helpers are no longer part of the API.** In 0.1.0 every helper in
  `plimsoll.wand`, `app.wand` and `cli.wand` was public, such as
  `plimsoll.pow10`, `app.container` and `cli.fetch!`. They are now in
  `_lib/`, which is private. The API is what the README describes: the
  `plimsoll` functions and types, `App` and its functions, and the `cli`
  command. If you used a helper, copy it into your own code.

- **A decode error from `get!` or `list!` says what to do.** When the
  cluster returns a value that your types do not have, the error now says
  that the cluster may be newer than the types in `k8s/`, and tells you to
  run `check` and `upgrade`:

  ```
  .spec.sessionAffinity: expected one of ClientIP, None, got "Sticky"
  The cluster may be newer than the types in k8s/. Run `wand github.com/wand-lang/plimsoll/cli check` to see what changed, and `upgrade` to generate the types again.
  ```

  The error itself stays: a script does not go on with a value that it
  does not know.

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
