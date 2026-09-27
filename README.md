# plimsoll

A Plimsoll line is the mark painted on a hull that says how deep she may
legally sit. Load her past it and she does not sail.

plimsoll is that mark for a Kubernetes object. It makes Kubernetes objects
typed [wand](https://github.com/wand-lang/wand) values, with the types
generated from your cluster's own OpenAPI schema. An object that does not
fit the schema does not typecheck, so the error comes before anything
reaches `kubectl`.

## What it does

- **Generates types from the cluster.** `plimsoll gen` reads
  `/openapi/v3` from the API server that `kubectl` points at, and writes one
  wand module for each API group and version: `k8s/apps/v1.wand`,
  `k8s/core/v1.wand`, and so on. The CRDs installed on your cluster are in
  the same schema, so they get types the same way.
- **Builds objects as values.** A Deployment is a `Deployment` record.
  Required fields must be given, optional fields default to `None`, and
  `apiVersion` and `kind` are filled in for you.
- **Writes and reads JSON with no hand-written code.** wand derives the
  encoder and the decoder from each type. `kubectl` reads JSON, so no YAML
  is written.
- **Applies objects through `kubectl`.** Server-side apply is the default.
  A server-side dry run checks an object against admission, webhooks and
  defaults without changing the cluster.
- **Keeps the types honest.** `plimsoll check` compares the committed types
  with a cluster, and `plimsoll upgrade` regenerates them and reports every
  script that the new schema breaks.

## Why plimsoll

YAML is text, and text is where Kubernetes errors live:

- **A misspelled field is dropped.** Write `containerPorts` for
  `containerPort` and nothing tells you. The port is not there, and you find
  out when traffic does not arrive. With plimsoll it is a type error.
- **A wrong type is taken as something else.** `replicas: "3"` is a string.
  With plimsoll, `replicas` is an `Int`, and an enum such as
  `imagePullPolicy` accepts only its own values.
- **Values that must agree are written twice.** A Deployment's
  `selector.matchLabels` must match its `template.metadata.labels`. In YAML
  you type them twice and nothing compares them. In wand they are one value,
  used twice, so they cannot disagree.

The types come from your cluster, not from a copy of upstream Kubernetes.
They match the API server your scripts talk to, and your own CRDs are
typed too.

### Why wand

- **The code that builds an object is pure.** In wand, every effect a file
  can have is on its first line, the `uses` line, and the typechecker holds
  the file to it. The functions that build objects perform no effects, and
  the typechecker proves it: they cannot reach the network, read a
  credential or run a command. Only the step that applies an object reaches
  the cluster, and a reviewer sees that on line one:

  ```
  uses {Shell(kubectl)}
  ```

- **A rehearsal before a real run.** `wand --dry-run deploy.wand` withholds
  every change and reports it. `Plimsoll.check` asks the API server to
  validate an object without storing it.
- **Tests need no cluster.** Building and checking objects is pure, so a
  test runs anywhere. The calls to `kubectl` can be mocked with a handler.
- **A schema change is a type error.** When an upgrade removes a field or
  makes one required, every script that is affected fails `wand t` before
  the upgrade reaches the cluster.

## How to use it

You need wand 0.88.1 or later, and `kubectl` with a context for your
cluster.

### 1. Add plimsoll to your package

```sh
wand p add github.com/wand-lang/plimsoll
```

### 2. Generate the types

```sh
wand github.com/wand-lang/plimsoll/cli gen
```

This writes `k8s/apps/v1.wand` and the modules it needs. Give the group
versions you use to generate more of them:

```sh
wand github.com/wand-lang/plimsoll/cli gen apis/apps/v1 api/v1 apis/batch/v1
```

Commit `k8s/`. A script must typecheck with no cluster present. Do not edit
the generated files; run `gen` again.

To see what `gen` would write without writing it:

```sh
wand --dry-run github.com/wand-lang/plimsoll/cli gen
```

### 3. Build objects

```ocaml
import Map

let apps = import ./k8s/apps/v1
let core = import ./k8s/core/v1
let meta = import ./k8s/meta/v1

let labels = Map.from_list [("app", "web")]

let web =
  apps.Deployment(
    metadata = Some meta.ObjectMeta(name = Some "web", labels = Some labels),
    spec = apps.DeploymentSpec(
      replicas = Some 3,
      selector = meta.LabelSelector(matchLabels = Some labels),
      template = core.PodTemplateSpec(
        metadata = Some meta.ObjectMeta(labels = Some labels),
        spec = Some core.PodSpec(
          containers = [
            core.Container(
              name = "web",
              image = Some "nginx:1.27",
              imagePullPolicy = Some core.ContainerImagePullPolicy.IfNotPresent,
              ports = Some [core.ContainerPort(containerPort = 8080)]
            )
          ]
        )
      )
    )
  )
```

`labels` is one value, used in the selector and in the template. An enum is
written with its type, as in `core.ContainerImagePullPolicy.IfNotPresent`,
because two Kubernetes enums can have the same value names.

### 4. Check and apply them

```ocaml
uses {Shell(kubectl)}

import Plimsoll

-- The API server validates the object and stores nothing.
Plimsoll.check apps.Deployment.encoder web

-- Server-side apply.
Plimsoll.apply apps.Deployment.encoder web
```

Each function takes the encoder or the decoder of the type it works on, so
it works with every generated kind:

| Function | What it does |
|---|---|
| `Plimsoll.apply enc obj` | Server-side apply of `obj` |
| `Plimsoll.check enc obj` | The same with `--dry-run=server`; changes nothing |
| `Plimsoll.get dec kind name` | Reads one object and decodes it |
| `Plimsoll.list dec kind` | Reads every object of a kind and decodes them |
| `Plimsoll.delete kind name` | Deletes one object |
| `Plimsoll.manifest enc objs` | Writes objects as one JSON list, for a file or a review |

### 5. Keep the types in step with your clusters

```sh
wand github.com/wand-lang/plimsoll/cli check              # the current context
wand github.com/wand-lang/plimsoll/cli check --context prod
wand github.com/wand-lang/plimsoll/cli upgrade
```

`check` exits non-zero when a cluster's schema differs from `k8s/`, and
names the types that changed. Run it in CI for each cluster you deploy to.
`upgrade` regenerates `k8s/` from the oldest cluster you list, typechecks
your scripts, and writes a report of each type that changed and each
script that broke.

## License

See [LICENSE](LICENSE).
