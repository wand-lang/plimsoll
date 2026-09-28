# LLM authoring log

An LLM (Claude Code) writes plimsoll. This log records the ergonomics of an
LLM writing wand: what came naturally, where habits from other languages
got in the way, whether a diagnostic led to the fix, and what each task
cost. The LLM adds one entry at the end of each task (one commit or one
PR), before it reports the task done.

Rules:

- Write the entry during the task, not later from memory.
- Quote the real diagnostic, with its code from `wand t --json`.
- Give evidence, not impressions: attempts, lines, the fix.
- Record easy items too. They show which parts of wand work.
- Do not soften. If the cause was the LLM's own habit, say so.
- A bug is not an entry. A wand bug is fixed in wand or gets an issue in
  wand-lang/wand; a plimsoll bug is fixed in plimsoll.

Causes: habit from another language · type error · effect or manifest ·
missing language or stdlib feature · docs missing or wrong · the LLM's own
mistake.

---

## 2026-09-27 — The generator for apps/v1 (build step 1)

**Task.** Wrote `_gen.wand` (the pure generator, 501 lines),
`cli.wand` (`gen`), `plimsoll.wand` (`IntOrString`, `Quantity`),
`test_gen.wand` (29 tests) and `testdata/apps_v1_cut.json`. `gen` read
`/openapi/v3/apis/apps/v1` from a kind cluster (Kubernetes v1.37.0) and
wrote four modules: `k8s/apps/v1` (30 records, 1 enum), `k8s/core/v1`
(112 records, 14 enums), `k8s/meta/v1` (15 records), `k8s/autoscaling/v1`
(3 records). wand 0.87.1. Commit: not committed yet.

### Easy

- **The generated modules were right on the first run.** All four
  typechecked, and all four were `wand f` fixed points. Cause of the ease:
  wand reads a keyword as a field name (`type: Option String = None`), so
  the Kubernetes field names needed no renaming.
- **Cross-module types.** `metadata: Option meta_v1.ObjectMeta = None` with
  `let meta_v1 = import ../meta/v1` worked with no change.
- **The first part of `_gen.wand` (about 190 lines) typechecked on the first
  attempt.** Pipelines over `Option` and `Result` (`Option.and_then`,
  `Result.to_option`, `Option.or_else`) read the JSON accessors cleanly.
- **`cli.wand` typechecked on the first attempt.** `wand t --fix` wrote the
  manifest: `uses {FS.Read, FS.Write, IO, Proc, Shell(kubectl)}`. The LLM
  wrote no effect by hand.
- **Lints found two real problems.** V-DROP2 found three tests that made
  two assertions and reported only the last one. V-PRED3 and V-BANG1 asked
  for `flag?`, `run_gen!` and `main!`.
- **A private module stays private.** A script outside the package that
  imported `_gen` got: "`_gen` is private to the package at ..., so no
  other package can import it".
- **Speed.** `gen` ran in 2.0 s, and a second run wrote the same bytes.

### Hard

1. **A pun where an update was meant.** First attempt:
   `Candidate(mod, key = "", ...)`. E-TYPE: "expected Candidate, got String
   -- 'mod' here is the record being updated, not a field. Write
   'Candidate(key = ..., mod)' to pun it". Fixed with `mod = mod`.
   Cause: the LLM's own mistake (the reference documents this rule).
   The diagnostic said what to do.
2. **A missing `import Option` in the test.** E-TYPE: "did you forget to
   import the standard library Option?". The JSON form carries a fix
   (`insert_line`). Cause: the LLM's own mistake.
3. **Two wrong test expectations.** The LLM expected
   `spec: DeploymentSpec,` and adjacent `apiVersion`/`kind` lines, but the
   output had field doc comments between lines, and `spec` was the last
   field. Cause: the LLM's own mistake.
4. **The formatter's enum layout is not documented.** The generator must
   write what `wand f` writes. By experiment: an enum stays on one line up
   to 92 characters, and a longer one gets one constructor on each line.
   Six probe runs. Cause: docs missing (the width).

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 6 runs (1 for part 1, 3 for part 2, 2 for part 3) | 0 | 4 |
| `cli.wand` | 2 | 1 (manifest) | 2 (names) |
| `test_gen.wand` | 3, then 2 runs of `wand s` (2 failures, then 0) | 1 (manifest) | 6 |
| generated modules | 1 | 0 | 0 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| 1 pun | E-TYPE | yes |
| 2 missing import | E-TYPE | yes, with a fix in JSON |
| lints | V-PRED3, V-BANG1, V-DROP2, V-USES2 | yes |

## 2026-09-27 — wand 0.88.0: shared enum names, a rehearsal of gen, the round trip

**Task.** Moved plimsoll to wand 0.88.0. `wand.pkg` requires `wand =
0.88.0`. `cli.wand` reads the schema with `Shell.inspect!`. `_gen.wand`
lets two enums in one module share a constructor name, and writes a wrapped
enum with no space after the `=`. `examples/deployment.wand` is now the
round trip: build, encode, `kubectl apply --dry-run=server`, decode. `gen`
against the kind cluster (Kubernetes v1.37.0) wrote 26 enum sums, where it
wrote 15 before: `k8s/apps/v1` 4, `k8s/core/v1` 22. Six value sets stay
`String`: five have a value that is not a constructor name, and one
(`AzureDiskVolumeSource.kind`) has the value `Shared`, a wand type name.
Commit: not committed yet.

### Easy

- **`wand --dry-run cli.wand gen` rehearsed on the first try.** It printed
  `ran (inspect): kubectl get --raw '/openapi/v3/apis/apps/v1'`, then a
  `would write` line for each module. Cause of the ease: `Shell.inspect!`
  (wand 0.88.0).
- **Qualified constructors in the example worked on the first try.**
  `apps.DeploymentStrategyType.RollingUpdate`,
  `core.PodSpecRestartPolicy.Always` and
  `core.ContainerImagePullPolicy.IfNotPresent` typechecked, encoded as
  `"RollingUpdate"`, `"Always"` and `"IfNotPresent"`, and decoded back from
  the server's object. Cause: derived enum sums and constructors qualified
  by type (wand 0.88.0).
- **The generator change was two small edits.** The clash rule and the
  enum header. `wand t` was clean on the first attempt for `_gen.wand` and
  `cli.wand`.

### Hard

1. **A statement sequence in a `match` arm.** I wrote three `IO.println`
   lines under `| Ok d ->` with no brackets. Diagnostic: E-TYPE "64:3:
   expected Unit, got ('a -> Unit ! {IO}) -> 'b". Fixed by `(...; ...; ...)`.
   Cause: the LLM's own mistake (`../wand/CLAUDE.md` says a sequence needs
   the brackets).

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `cli.wand` | 1 | 0 | 1 |
| `_gen.wand` | 1 | 0 | 2 |
| `test_gen.wand` | 1, then 2 runs of `wand s` (2 failures by design, then 0) | 0 | 1 |
| `examples/deployment.wand` | 4 | 2 (an import; the manifest) | 1 |
| generated modules | 1 | 0 | 0 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| missing `import Option` | E-TYPE | yes, with a fix |

## 2026-09-27 — The runtime API and the quantity helpers

**Task.** Wrote the runtime API in `plimsoll.wand`: `apply!`, `check!`,
`get!`, `list!`, `delete!`, each with a plain form that gives a `Result`,
and `manifest`. Wrote `millicores` and `bytes` for `Quantity`. Wrote
`test_plimsoll.wand` (16 tests, no cluster). Changed
`examples/deployment.wand` to use `check!`. Ran apply, get, list and delete
against the kind cluster (Kubernetes v1.37.0). wand 0.88.1.

### Easy

- **`plimsoll.wand` typechecked on the first attempt.** The only finding
  was the manifest (V-USES2), and `wand t --fix` wrote it: `uses
  {Shell(kubectl)}`.
- **Mocking needed no library.** A `Shell!command c _ -> c` case that does
  not resume gives the command line, so a test reads exactly what would
  run. A `Shell!run (_, input) _ -> input` case gives what was piped to it,
  so a test reads the JSON that `apply!` sends. Both worked on the first
  try.
- **The `!` and `Result` pair is one line.** `let apply enc obj = try apply!
  enc obj`, the same shape as `Shell.run` in the standard library.
- **Derived decoders made `get!` and `list!` two lines each.**
  `JSON.decode (Decode.list dec) (JSON.field! "items" doc)` read a
  `kubectl get -o json` list on the first run.
- **Overflow is not silent.** `millicores "1Ei"` raised "integer overflow in
  '*': Int holds -4611686018427387904 to 4611686018427387903" on the first
  run, with the line and column.

### Hard

1. **A function that gives a `Result` raised.** `millicores` computed
   `n * 1000` outside a `try`, so an overflow raised instead of giving an
   `Error`. Fixed with `Result.and_then (fn (n, d) -> try ...)`. Cause: the
   LLM's own mistake.
2. **Three wrong test expectations.** The LLM expected `kubectl get
   deployment web -o json`; wand writes `kubectl get 'deployment' 'web' -o
   json`, since `%{x}` in a command is quoted as one argument. And it
   expected "'128MB' is not a quantity" where the code says "'MB' is not a
   quantity suffix". Cause: the LLM's own mistake.
3. **A script that uses the API is told to widen its manifest.** The
   example declared `uses {IO, Shell(kubectl)}`. `wand t` gave A-USES1:
   "the manifest allows 'kubectl', which no command here runs; it could be
   "uses {IO, Shell}"". The command words are in `plimsoll.wand`, whose own
   manifest bounds them, so the script is told to declare `Shell` with no
   list. That is wand's rule, but the design wants a script's first line to
   say `Shell(kubectl)`. Cause: effect or manifest.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `plimsoll.wand` | 1, then 1 fix after a run | 1 (manifest) | 1 |
| `test_plimsoll.wand` | 2, then 2 runs of `wand s` (3 failures, then 0) | 3 (two imports, manifest) | 3 |
| `examples/deployment.wand` | 1 | 1 (manifest) | 1 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| overflow | runtime error | yes: it named the limits and the place |
| missing `import Result` | E-TYPE | yes, with a fix |
| 3 wider manifest | A-USES1 | yes, with a fix |

## 2026-09-27 — Namespaces for get, list and delete

**Task.** Added `get_in!`, `list_in!` and `delete_in!`, and their `Result`
forms, to `plimsoll.wand`, with 3 tests. Changed the README to say that a
script using plimsoll declares `uses {Shell}`. wand 0.88.1.

### Easy

- **All of it was clean on the first attempt.** `wand t` reported nothing
  on `plimsoll.wand` and `test_plimsoll.wand`, and `wand s` passed 51 of
  51. The new functions copy the shape of the old ones, with `-n %{ns}` in
  the command; the tests copy the `Shell!command` case.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `plimsoll.wand` | 1 | 0 | 0 |
| `test_plimsoll.wand` | 1, then 1 run of `wand s` | 0 | 0 |

## 2026-09-27 — check and upgrade

**Task.** `gen` writes the group-versions it read into each module's first
line. Wrote `_drift.wand` (the pure comparison) and `test_drift.wand` (7
tests). Added `check [--contexts a,b,c]` and `upgrade [--contexts a,b,c]`
to `cli.wand`. Against the kind cluster (Kubernetes v1.37.0): `check`
matched; after a field was added to `Deployment` by hand, `check` named it
and exited 1, and `upgrade` wrote `k8s/` again and named the script that
used the field. wand 0.88.1.

### Easy

- **`_drift.wand` typechecked on the first attempt,** and its 7 tests
  passed on the first run. A fold over the lines with a small `Parse`
  record read records and sums, both one-line and wrapped.
- **The output of `wand t --json` decoded with a derived decoder.** `type
  Diagnostic(severity: String, file: String, line: Int, col: Int, message:
  String)` read it; the keys it does not name were left out.
- **The oldest cluster was one line.** `List.sort_by (fn (_, v) -> v)`
  sorted `Version` values, since `Version` is ordered.
- **The manifest named the new binary.** `wand t` gave "this command runs
  'wand', which Shell(kubectl) does not allow", and `wand t --fix` wrote
  `Shell(kubectl, wand)`.

### Hard

1. **A body with statements, again.** `run_upgrade!` had a statement after
   its `let` lines with no brackets. Diagnostic: "the ';' above ended the
   definition, so this line is a statement of its own rather than part of
   it -- put the body in parentheses to sequence it". Fixed with `( ... )`.
   Cause: the LLM's own mistake, the second time in two tasks. The message
   said what to do.
2. **Two `!` names that cannot raise.** `sources_of!` and `with_contexts!`
   end the process with `Proc.exit`, and do not raise. V-BANG2: "'sources_of!'
   cannot raise, so the `!` promises a risk that is not there". Renamed.
   Cause: the LLM's own mistake (it read "may stop the script" as "raises").
3. **`FS.glob` gives whole paths.** The LLM stripped a leading `./` only, so
   every module on disk looked removed and every generated one new. Found on
   the first run against the cluster. Fixed by writing each path from the
   working directory. Cause: the LLM's own mistake (it did not read `wand d
   FS.glob` first).

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 1 | 0 | 0 |
| `_drift.wand` | 1 | 0 | 0 |
| `test_drift.wand` | 2, then 1 run of `wand s` | 2 (imports) | 0 |
| `cli.wand` | 4, then 2 fixes after runs | 1 (manifest) | 4 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| 1 body with statements | parse error | yes |
| 2 `!` names | V-BANG2 | yes |
| manifest | E-TYPE | yes, with a fix |

## 2026-09-27 — The full API: a layout the generator did not match

**Task.** Ran `gen` on all 23 group-versions a Kubernetes v1.37.0 API
server lists, in a scratch copy: 25 modules, 603 records and 60 sums, in
6.7 s. Two modules were not `wand f` fixed points; fixed the generator.
wand 0.88.1.

### Easy

- **Scale was not a problem.** 603 records typechecked, where they parsed,
  with no change to the generator for size.
- **The fix was one rule in one function.** `render_rec` writes a record
  with no field comments on one line when it fits in 92 columns. `wand t`
  was clean on the first attempt.

### Hard

1. **The generator did not write what `wand f` writes for a short record.**
   `wand f` writes `type GroupResource(group: String, resource: String)`
   on one line; the generator wrote one field per line. The `apps/v1` cut
   had no record without field comments, so the tests did not show it.
   Cause: docs missing (the reference does not say when `wand f` puts a
   record on one line; the LLM found it by diffing).
2. **A test used a value defined further down the file.** E-TYPE:
   "'odd_out' needs its type before '.files' can be read: write '(odd_out:
   Output)'". The real cause was the order of definitions, which the
   message did not name. Fixed by removing the test, which added nothing.
   Cause: the LLM's own mistake.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 1 | 0 | 0 |
| `test_gen.wand` | 2, then 2 runs of `wand s` | 0 | 3 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| 2 order of definitions | E-TYPE | no: it asked for a type annotation, and the fix was the order |

## 2026-09-27 — Keys a field name cannot be

**Task.** The generator gives a property that wand cannot spell a field
name made from its key, and writes the key: `port "Port": Int`, `ref
"$ref": Option String = None`. This uses the document key of wand 0.89.0
(not released yet; checked with a local build). On all 23 group-versions:
25 modules typecheck, all are `wand f` fixed points, and no property is
left out. A real Node from the kind cluster decoded its kubelet `Port`,
and a CRD from a server-side dry run decoded and encoded with its
`x-kubernetes-*` keys.

### Easy

- **The change to the generator was small.** One function makes names
  from keys (`field_names`), and `render_field` writes the key. `wand t`
  was clean on the first attempt after the edit.
- **Nothing else changed for keys.** Every record that holds a keyed type
  (`NodeStatus`, `CustomResourceDefinition`) kept its derived decoder with
  no generated code.

### Hard

1. **An edit removed five functions.** The LLM replaced the text from one
   function to another in a script, and five functions stood between them.
   `wand t` gave "unbound variable 'doc_lines'" and three more, which named
   the missing functions at once; they were restored from git. Cause: the
   LLM's own mistake.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 2 | 0 | 1 |
| `test_gen.wand` | 1, then 2 runs of `wand s` (2 failures by design, then 0) | 0 | 1 |

## 2026-09-27 — gen reads every group-version

**Task.** `gen` with no group-version reads every one the API server
lists at `/openapi/v3`, CRDs included. `k8s/` now holds 25 modules
(956 KB) for Kubernetes v1.37.0. wand 0.89.0.

### Easy

- **The change was one function and one line.** `group_versions!` reads the
  paths with `Shell.inspect!`, keeps `api/v1` and `apis/<group>/<version>`
  with one `Regex.match?`, and `gen` calls it. `wand t --fix` added the two
  imports it named (`Map`, `Regex`).
- **Nothing else needed a change.** All 25 modules typecheck and are
  `wand f` fixed points; `check` matched, and the round trip still decoded
  the server's object.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `cli.wand` | 2 | 2 (imports) | 0 |

## 2026-09-27 — CRDs: nested objects become records

**Task.** Installed a test CRD (`widgets.example.com`) in the kind cluster
and ran `gen`. The first run made `spec: Option JSON`, since a CRD writes
its nested objects in place and the generator read an object with no
`$ref` as JSON. Added `lift` to `_gen.wand`: each inline object becomes a
record named for its place (`WidgetSpec`, `WidgetSpecOwner`), in arrays
and maps too; an object that keeps unknown fields stays JSON. Then a
Widget round-tripped through the API server with `check!`, `apply!`,
`get_in!`, `list!` and `delete!`, with the same spec back. 7 tests. The
built-in modules did not change. wand 0.89.0.

### Easy

- **`lift` typechecked on the first attempt.** One recursive function that
  gives the schema to use and the new definitions, and `JSON.of_map` with
  `Map.set` to rebuild a schema.
- **Everything else came free.** The lifted definitions went through the
  same code as built-in ones: the enum became `WidgetSpecMode = Fast |
  Slow`, `max-replicas` got its key, `port` became `IntOrString`, and the
  derived decoder read the server's object back.
- **Qualified constructors read well in a CRD.**
  `ex.WidgetSpecMode.Fast` in the script needed no thought.

### Hard

1. **A pipe in a test argument.** `t.eq [..] (x).files |> List.map f` read
   as `(t.eq [..] x.files) |> List.map f`. E-TYPE: "expected String, got
   File". Fixed with brackets. Cause: the LLM's own mistake. The message
   named the types, not the precedence.
2. **A test expected the multi-line layout** for a record the generator
   writes on one line. Cause: the LLM's own mistake.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 1 | 0 | 0 |
| `test_gen.wand` | 2, then 2 runs of `wand s` | 0 | 2 |

### Diagnostics

| Item | Code | Did it say what to do? |
|---|---|---|
| 1 pipe in an argument | E-TYPE | no: types only |

## 2026-09-27 — CI and the end-to-end test

**Task.** Wrote `tools/e2e.wand` (the checks against a cluster: `check`,
the Deployment round trip, and a CRD from `testdata/widget-crd.yaml`
round-tripped by `testdata/widget.wand` in a copy of the package) and
`.github/workflows/ci.yml` (the checks with no cluster, and `e2e.wand` on
kind). On a local kind cluster `e2e.wand` passed in 6 s and left the
cluster as it was. wand 0.89.0.

### Easy

- **`e2e.wand` typechecked on the first attempt,** and `wand t --fix` wrote
  its manifest: `uses {FS.Read, FS.Write, IO, Proc, Shell(kubectl, sh,
  wand)}`. The manifest names every binary the test runs, which is what a
  reviewer of a CI script wants to see.
- **Cleanup whatever happens was two lines.** `with FS.temp_dir ... as dir
  -> try widget! dir` removes the copy, and the CRD is deleted before the
  outcome is matched.

### Hard

1. **A script that imports a module that is not there.**
   `testdata/widget.wand` imports the module `gen` writes for the CRD, so
   it cannot be typechecked in the repo. The LLM ran `wand t --fix` on it
   in a copy that had the module, and brought the manifest back. Cause:
   missing language or stdlib feature (no way to typecheck a script
   against a module that does not exist yet).
2. **No way to run a command in another directory.** `gen` writes under
   the working directory, and wand has no `cd` for a command, so the copy
   is reached with `sh -c 'cd "$1" && ...'`. Cause: missing language or
   stdlib feature.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `tools/e2e.wand` | 1 | 1 (manifest) | 0 |
| `testdata/widget.wand` | 1, in a copy | 2 (an import, manifest) | 0 |

## 2026-09-27 — CI: wait for a CRD's schema

**Task.** The first CI run failed in `tools/e2e.wand`: the CRD was
Established, but `/openapi/v3/apis/example.com/v1alpha1` was "not found"
for a moment longer. Added `wait_for_schema!`, which reads `/openapi/v3`
once a second until the path is listed. wand 0.89.0.

### Easy

- **`wand t --fix` wrote everything the retry loop needed.** Three imports
  (`String`, `Result`, `Clock`) and `Clock` in the manifest, since the loop
  waits. `wand t` was clean after that, on the first attempt.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `tools/e2e.wand` | 1 | 4 (three imports, manifest) | 0 |

## 2026-09-27 — A weekly run on the newest Kubernetes versions

**Task.** Added `.github/workflows/kubernetes.yml`: once a week, for the
three newest Kubernetes minors of the newest kind release, gen writes every
group-version, and the modules and `tools/e2e.wand` are checked; on the
newest one, `upgrade` and `gen` write k8s/ again, and a change becomes a
branch and a pull request (or an issue). Renamed `testdata/widget.wand` to
`testdata/widget.wand.in` so `wand t .` does not read it.

### Easy

- **No wand code changed for the new job.** `gen`, `upgrade` and
  `tools/e2e.wand` already did each step; the workflow only calls them.

### Hard

1. **A script that cannot typecheck in place broke `wand t .`.**
   `upgrade` typechecks the whole repo, and `testdata/widget.wand` imports
   a module that only exists in the e2e copy, so every report would have
   listed it as broken. Renamed it to `.in`. Cause: missing language or
   stdlib feature (no way to mark a file that `wand t .` should skip).

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `tools/e2e.wand` | 1 | 0 | 1 |

## 2026-09-27 — gen removes the modules it did not write

**Task.** The first weekly run failed on Kubernetes v1.35.8 and v1.36.4:
"k8s/storagemigration/v1.wand: type error: unknown type
'meta_v1.GroupResource'". `gen` wrote the modules for the older version
and left the v1.37 `storagemigration/v1.wand` in place, which named a type
the new `meta/v1` does not hold. Added `remove_stale!` to `cli.wand`: gen
and upgrade remove each module with gen's first line that this run did not
write. wand 0.89.0.

### Easy

- **The fix typechecked on the first attempt,** and a local test (one
  stale module with gen's header, one hand-written file without it)
  removed the first and kept the second.
- **The diagnostic named the cause.** "unknown type
  'meta_v1.GroupResource' in field 'resource' of
  'StorageVersionMigrationSpec'" pointed at the module and the type, so
  the stale module was clear from the first failed run.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `cli.wand` | 1 | 0 | 0 |

## 2026-09-27 — The weekly run: one example, one version

**Task.** The second weekly run passed the typecheck on Kubernetes v1.35
and v1.36, and failed at `examples/deployment.wand`: "DeploymentSpec and
Option DeploymentSpec are not the same type". `Deployment.spec` is
required in v1.37 and optional in v1.36. The example is written for the
version k8s/ came from, so `tools/e2e.wand --no-example` leaves it out, and
the weekly run uses that. wand 0.89.0.

### Easy

- **The diagnostic showed the schema change at once.** It named the two
  types, and the fix was not in wand code at all but in which test runs
  where.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `tools/e2e.wand` | 1 | 0 | 0 |

## 2026-09-27 — The ergonomic layer: App → Deployment and Service

**Task.** Wrote `app.wand` (111 lines): `App(name, image, port, replicas,
env, cpu, memory)`, and `deployment`, `service`, `labels`, `manifest` and
`apply!`. The objects are records of plimsoll's own `k8s/` types, so a
script changes one with a record update. Also `test_app.wand` (12 tests),
`examples/app.wand` (the server validates both objects), step 2 of
`tools/e2e.wand`, and a README section. wand 0.89.0, Kubernetes v1.37.0 on
kind.

### Easy

- **The builders typechecked on the first attempt after two fixes.** The
  shapes came from the generated types, and the field names in `k8s/` are
  the names in the Kubernetes docs.
- **A record update of a nested record.** `apps.Deployment(d, spec =
  apps.DeploymentSpec(d.spec, minReadySeconds = Some 5))` was right on the
  first attempt, in the test and in the example.
- **A handler tested `apply!` with no cluster.** `Shell!run (_, input) k ->
  k input` gave back the two JSON texts that the two kubectl commands got.
- **`examples/app.wand`, `test_app.wand` and the e2e step** typechecked on
  the first attempt.

### Hard

- **A field default that is not a literal.** The LLM wrote
  `env: Map String = Map.empty`. E-TYPE: "the default for field 'env' of
  'App' has to be a value written out: a literal, or a constructor applied
  to literals. It is read with nothing in scope, so it says the same thing
  at every construction that leaves the field out". The fix, `{}`, came
  from the syntax card, not from the message: the message says "a literal"
  and does not name the empty map literal. Cause: habit from another
  language (a default is an expression).
- **A punned field in the first place is a record update.** The LLM wrote
  `core.EnvVar(name, value = Some v)` with `name` bound by the lambda.
  E-TYPE: "expected EnvVar, got String -- 'name' here is the record being
  updated, not a field. Write 'EnvVar(value = ..., name)' to pun it". The
  message named the cause and the fix. Cause: the LLM's own mistake; the
  syntax card shows both forms.
- **The manifest.** The LLM added `uses {Shell(kubectl)}` by hand, after
  V-USES2 had suggested `uses {Shell}`. A-USES1: "the manifest allows
  'kubectl', which no command here runs; it could be \"uses {Shell}\"". The
  command word is in `plimsoll.wand`, so this file names none. `wand t
  --fix` wrote the line. Cause: the LLM's own mistake; it wrote an effect
  by hand, which Part B says not to do.
- **`Map.to_list` order.** A test expected the env sorted by name and
  failed: "expected Some(Some([EnvVar("LOG", ...), EnvVar("MODE", ...)])),
  got Some(Some([EnvVar("MODE", ...), EnvVar("LOG", ...)]))". `wand d
  Map.to_list` says "in the order the keys were added". The LLM assumed a
  sorted map without reading the doc. The order as written is the better
  behavior, since `$(NAME)` names only an earlier variable, so the test
  changed, not the code. Cause: habit from another language (OCaml's
  `Map` is sorted).
- **One wrong test, found before the run.** The first `manifest` test
  passed `JSON.of_list` to `plimsoll.manifest` and would have compared a
  nested list. The LLM saw it by reading and wrote the expected text
  directly. Cause: the LLM's own mistake.
- **The formatter split one `env = if ... else Some (List.map ...)` field
  over five lines.** The LLM moved it into a function `env_vars` with a
  `match`, which reads better. Not a fault of `wand f`.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `app.wand` | 5 | 1 (the manifest) | 2 (the default, the pun) |
| `test_app.wand` | 1 | 0 | 0 |
| `examples/app.wand` | 1 | 0 | 0 |
| `tools/e2e.wand` | 1 | 0 | 0 |

One test failed on its first run (the env order). `wand s`: 81 tests pass.

## 2026-09-28 — A rehearsal runs check!

**Task.** `wand --dry-run` withheld `plimsoll.check!`, so both examples
stopped at "json_parse: Blank input data". `check!` now runs its
server-side dry run with `Shell.inspect_with!`, which wand 0.90.0 adds for
this, and a rehearsal runs it. `wand.pkg` needs wand 0.90.0.

### Hard

- **No stdin for a command that only reads.** The LLM tried
  `x |> Shell.inspect! $*(kubectl apply --dry-run=server -f -)`. E-TYPE:
  "String and 'a -> 'b are not the same type". The message did not say
  that `inspect!` takes no input; the LLM found that in `wand d
  Shell.inspect!`. Cause: missing language or stdlib feature. Fixed in wand
  0.90.0 (`Shell.inspect_with!`).
- **V-SHELL3 did not know a kubectl dry run.** `Shell.inspect!
  $*(kubectl apply --server-side --dry-run=server -o json -f -)` gave
  V-SHELL3: "'kubectl apply' changes things, and Shell.inspect! runs it in
  a rehearsal as well as in a real run; run it with $(...) so that
  --dry-run withholds it". The command stores nothing. Cause: missing
  language or stdlib feature. Fixed in wand 0.90.0.
- **`wand f` is not a fixed point on `plimsoll.wand`.** It moves a
  multi-line `match` arm body onto the arrow line and puts the `|>` at the
  level of the arm. The meaning stays the same, but the layout is worse.
  The LLM did not keep the change. Cause: a formatter fault, for wand.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `plimsoll.wand` | 1 | 0 | 0 |

## 2026-09-28 — wand f fixed points, and wand 0.90.1

**Task.** `plimsoll.wand` and `examples/deployment.wand` are now `wand f`
fixed points, formatted with wand 0.90.1. `wand.pkg` needs 0.90.1.

### Hard

- **The LLM did not run `wand f` on the hand-written files.** The checks
  in CLAUDE.md run it on `k8s/` only, and the LLM ran no more than that.
  With 0.90.1, eight hand-written files are not fixed points. Cause: the
  LLM's own mistake.
- **`wand p interface --check` failed on the section that `wand p
  release` had just written**, with "the interface section of wand.pkg
  does not match the code" and every field of a long record listed as
  removed. The LLM ran the check after the tag was pushed. Cause: a wand
  bug, fixed in 0.90.1.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `plimsoll.wand` | 1 | 0 | 0 |

## 2026-09-28 — Every source a wand f fixed point

**Task.** All hand-written files are formatted with wand 0.90.2, and CI
and the checks in CLAUDE.md now hold them to `wand f`, as they held
`k8s/`. `wand.pkg` needs 0.90.2. 10 files changed.

### Hard

- **Layouts that `wand f` wrote worse.** Formatting the six remaining
  files with 0.90.1 showed four wand formatter faults: a `&&` chain joined
  onto one line of 115 columns, a value after `else`, `name =` or `->`
  that broke with its second line at the indent of the head, a
  construction through a module measured from the wrong column, and a
  one-armed `if` whose block opened below `then`. The LLM found them by
  reading the diff and measuring each new line past the margin of 92, not
  from a check. Cause: formatter faults, fixed in wand 0.90.2.
- **A layout that is not a fault.** `(fn acc p -> if ...` with `else`
  below at the lambda's indent looked like the same fault. The comment in
  the formatter says it is the rule, so the LLM left it. Cause: none; the
  formatter's comments answered it.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| all 10 | 1 | 0 | 0 |

## 2026-09-28 — Every enum a sum type, with wand 0.91.0 spellings

**Task.** The 17 enum value sets that stayed `String` are now sum types.
`_gen.wand` gives each value that cannot be a constructor name a name and
keeps the value as its spelling in documents: `None_ "None"`, `All "*"`,
`ClientAuth "client auth"`, `SIGRTMAX1 "SIGRTMAX-1"`. wand 0.91.0 added
those spellings for this. 20 fields in 6 modules changed type. `gen`
writes no "stays String" note now. `wand.pkg` needs 0.91.0.

### Easy

- **The generator change typechecked after one fix.** `spell_values` uses
  the trailing `_` rule that `field_names` already had.
- **The API server took the spellings.** A Service with
  `sessionAffinity = None_` and `ipFamilies = [IPv4_]` was written as
  `"None"` and `"IPv4"`, checked with `check!`, and decoded back.

### Hard

- **A name used before it was defined.** The LLM put the new functions
  above `capital`, which they call. `wand t`: "unbound variable 'capital'".
  The message named the function; the fix was to move the block. Cause:
  the LLM's own mistake.
- **A test helper name taken.** The LLM added a second `text_of` to
  `test_gen.wand`. Parse error: "'text_of' is already defined above.
  Equations for a function must be consecutive". Cause: the LLM's own
  mistake.
- **V-BANG1 on two test helpers** that call `JSON.parse!`: "'spell' can
  raise, but its name does not say so". Renamed to `schema_with!` and
  `spelled_text!`. Cause: the LLM's own mistake.
- **A test that assumed sorted values.** The LLM expected
  `Gadget | Widget_ "Widget"`; `gen` keeps the schema's order. Cause: the
  LLM's own mistake.
- **The LLM claimed a bug in `Args` that was not one.** `Args.parse
  D.decoder` reads a flag by the field's key, and the LLM called it a bug.
  The reference says `T.decoder` reads a document and `T.parser` reads a
  command line. Cause: the LLM did not read the reference before it
  reported.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `_gen.wand` | 2 | 0 | 1 (order) |
| `test_gen.wand` | 3 | 0 | 2 (name, `!` names) |

## 2026-09-28 — A decode error says the types may be old

**Task.** `get!`, `list!`, `get_in!` and `list_in!` add a hint to a decode
error: the cluster may be newer than the types in `k8s/`, run `check`,
then `upgrade`. Enums stay closed, on purpose. README has a section on the
error and its limit with clusters of different versions. 2 tests added.

### Easy

- **The change was one helper.** `Result.map_error` put the hint after the
  decoder's message, and `Result.get!` raised it, as before.
- **The real message was easy to get for the README.** A handler that
  answers `Shell!run` gave the exact text with no cluster.

### Hard

- **The LLM wrote a fix in the README that was not true in every case.**
  It said "run `upgrade`", but `upgrade` takes the types from the oldest
  cluster, so a value from a newer one stays unknown. The LLM saw it on a
  second read and wrote the limit down. Cause: the LLM's own mistake.

### Cost

| File | Attempts until `wand t` was clean | By `wand t --fix` | By hand |
|---|---|---|---|
| `plimsoll.wand` | 1 | 0 | 0 |
| `test_plimsoll.wand` | 1 | 0 | 0 |
