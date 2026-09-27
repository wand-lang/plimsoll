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
