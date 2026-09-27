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
