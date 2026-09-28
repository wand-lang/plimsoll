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
  uses {Shell}
  ```

  A file with no `Shell` on its first line cannot reach the cluster at all.
  plimsoll's own first line is `uses {Shell(kubectl)}`, so the commands it
  runs for a script are `kubectl` and nothing else.

- **A rehearsal before a real run.** `wand --dry-run deploy.wand` withholds
  every change and reports it. `Plimsoll.check` asks the API server to
  validate an object without storing it.
- **Tests need no cluster.** Building and checking objects is pure, so a
  test runs anywhere. The calls to `kubectl` can be mocked with a handler.
- **A schema change is a type error.** When an upgrade removes a field or
  makes one required, every script that is affected fails `wand t` before
  the upgrade reaches the cluster.

## How to use it

You need wand 0.91.1 or later, and `kubectl` with a context for your
cluster.

### 1. Add plimsoll to your package

```sh
wand p add github.com/wand-lang/plimsoll
```

### 2. Generate the types

```sh
wand github.com/wand-lang/plimsoll/cli gen
```

`gen` runs `kubectl`, so it reads the cluster of your current kubectl
context. To read another cluster, name its context:

```sh
wand github.com/wand-lang/plimsoll/cli gen --context prod
```

This writes one module for each API group and version your cluster
serves, CRDs included: `k8s/core/v1.wand`, `k8s/apps/v1.wand`,
`k8s/batch/v1.wand`, and so on. To generate only some of them, name them:

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

From here on, your Kubernetes objects are wand values in scripts, not YAML
files. A script builds them, checks them against the cluster, and applies
them.

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

Some Kubernetes values cannot be wand constructor names. Such a value gets a
name, and the generated type keeps the value as the constructor's word in
the document: `core.PodSpecDnsPolicy.None_` is written as `"None"`, and
`NamedRuleWithOperationsOperations.All` as `"*"`. A value that is a wand
name or the name of a type gets a trailing `_`. Another value gets a name
made from its parts: `"client auth"` is `ClientAuth`.

### 4. Check and apply them

```ocaml
uses {Shell}

import github.com/wand-lang/plimsoll

-- The API server validates the object and stores nothing. It gives the
-- object the server would store.
let stored = plimsoll.check! apps.Deployment.encoder web

-- Server-side apply.
plimsoll.apply! apps.Deployment.encoder web
```

Each function takes the encoder or the decoder of the type it works on, so
it works with every generated kind. Each one that can fail has a `!` form
that raises, and a plain form that gives a `Result`:

| Function | What it does |
|---|---|
| `plimsoll.apply! enc obj` | Server-side apply of `obj` |
| `plimsoll.check! enc obj` | The same as a server-side dry run: the server validates `obj`, stores nothing, and gives the object it would store |
| `plimsoll.get! dec kind name` | Reads one object and decodes it |
| `plimsoll.list! dec kind` | Reads every object of a kind and decodes them |
| `plimsoll.delete! kind name` | Deletes one object |
| `plimsoll.get_in! dec ns kind name`, `list_in!`, `delete_in!` | The same, in the namespace `ns` |
| `plimsoll.manifest enc objs` | Writes objects as one JSON list, for a file or a review |

`get!`, `list!` and `delete!` work in the current namespace, which the
kubectl context sets. The `_in` forms name the namespace at the call.
`apply!` and `check!` use the namespace in the object's `metadata`.

`get!`, `list!` and `check!` change nothing, so a rehearsal (`wand
--dry-run`) runs them, and a script that reads or checks before it changes
the cluster rehearses the path a real run takes. `apply!` and `delete!` are
withheld.

`plimsoll.millicores` and `plimsoll.bytes` read a `Quantity`: "500m" is 500
millicores, and "128Mi" is 134217728 bytes.

#### When an object does not decode

An enum in `k8s/` holds the values that your cluster had when you
generated the types. If the cluster is upgraded later, it can return a
value that the enum does not have, or leave out a field that is now
required. Then `get!` and `list!` stop with an error that names the field
and the value, and says what to do:

```
.spec.sessionAffinity: expected one of ClientIP, None, got "Sticky"
The cluster may be newer than the types in k8s/. Run `wand github.com/wand-lang/plimsoll/cli check` to see what changed, and `upgrade` to generate the types again.
```

This is on purpose. A script does not continue with a value that it does
not know. To fix it, generate the types again with `upgrade` (see step 6),
then fix each script that it reports. Run `check` in CI against each
cluster, and you see a new value before a script does.

If your scripts use clusters of different versions, `upgrade` takes the
types from the oldest one. A value that only the newer clusters have stays
unknown until the oldest cluster is upgraded too.

### 5. Or start from an App

For a usual web service, `app` makes the Deployment and the Service from a
few fields:

```ocaml
uses {Shell}

import Map

let app = import github.com/wand-lang/plimsoll/app

let web =
  app.App(
    name = "web",
    image = "nginx:1.27",
    port = 8080,
    replicas = 3,
    env = Map.from_list [("MODE", "prod")],
    cpu = Some "250m",
    memory = Some "128Mi"
  )

app.apply! web
```

| Function | What it gives |
|---|---|
| `app.deployment a` | The Deployment: `a.replicas` pods that run `a.image` and listen on `a.port` |
| `app.service a` | The Service, with the app's name, that sends `a.port` to the pods |
| `app.labels a` | `{app = <name>}`: the labels of the pods, and of both selectors |
| `app.manifest a` | The two objects as one JSON list. It reaches nothing |
| `app.apply! a` | Server-side apply of the Deployment, then the Service |

Only `replicas` (1), `env`, `cpu` and `memory` can be left out. The labels
are one value, so the selectors and the pods cannot disagree.

The objects are records of plimsoll's own types, which come from
Kubernetes v1.37. To set a field that `App` does not have, use a record
update, and apply the object with its encoder:

```ocaml
let apps = import github.com/wand-lang/plimsoll/k8s/apps/v1

let d = app.deployment web
let slow = apps.Deployment(d, spec = apps.DeploymentSpec(d.spec, minReadySeconds = Some 30))

plimsoll.apply! apps.Deployment.encoder slow
```

### 6. Keep the types in step with your clusters

```sh
wand github.com/wand-lang/plimsoll/cli check
wand github.com/wand-lang/plimsoll/cli check --contexts staging,prod
wand github.com/wand-lang/plimsoll/cli upgrade --contexts staging,prod
```

`check` generates the types again, in memory, from each kubectl context,
and compares them with `k8s/`. It names each type, field and enum value
that differs, and exits non-zero when anything does. Run it in CI for each
cluster you deploy to. With no `--contexts`, it uses the current context.

`upgrade` finds the oldest of the clusters, writes `k8s/` from it,
typechecks your scripts, and prints a report: what changed in the schema,
and each script that no longer typechecks, with its file, line and
message. It exits non-zero when a script broke, so a CI job can open the
upgrade as a pull request with the report as its body.

Neither command needs the group-versions again: each module's first line
names the ones `gen` read.

## License

See [LICENSE](LICENSE).
