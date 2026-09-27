# CLAUDE.md

plimsoll makes Kubernetes objects typed wand values. It generates wand types
from the cluster's OpenAPI v3 schema. The design is the "plimsoll — Design"
doc. wand's own rules for writing wand are in `../wand/CLAUDE.md`, Part B.
Obey them here.

## Layout

- `plimsoll.wand` — the hand-written types that generated modules use:
  `IntOrString`, `Quantity`. The runtime API comes here later.
- `_gen.wand` — the generator. It is pure: OpenAPI documents in, file texts
  and notes out. It is private to this package.
- `cli.wand` — the command: `wand github.com/wand-lang/plimsoll/cli gen
  [group-version ...]`. Run it here as `wand cli.wand gen`.
- `k8s/` — generated modules. Do not edit them. Run `gen` again.
- `test_*.wand` — tests, run with `wand s`. `testdata/` holds fixtures cut
  from a real schema.
- `examples/` — scripts that use the generated types.
- `docs/llm-authoring.md` — the authoring log. See below.

## Checks

Check by exit code.

```sh
wand t _gen.wand cli.wand plimsoll.wand test_gen.wand   # no findings
wand s                                                  # all tests pass
wand t k8s/*/*.wand                                     # generated modules
for f in k8s/*/*.wand; do cp $f /tmp/x.wand; wand f /tmp/x.wand; cmp $f /tmp/x.wand; done
```

The generated modules must be `wand f` fixed points. If `wand f` changes
one, change the generator, not the file.

`gen` needs a cluster. `wand --dry-run cli.wand gen` rehearses it: the
schema is read with `Shell.inspect!`, which a rehearsal runs, and the files
are reported, not written.

`examples/deployment.wand` is the round trip: it builds a Deployment,
encodes it, has the API server validate it with `--dry-run=server`, and
decodes the object the server would store. Run it after a change to the
generator.

## The authoring log

The log records the ergonomics of an LLM writing wand: what came
naturally, where habits from other languages got in the way, whether a
diagnostic led to the fix, and what the task cost. It is not a bug list.
Fix a wand bug in wand, or file an issue in wand-lang/wand; fix a plimsoll
bug in plimsoll. Neither goes in the log.

At the end of each task (one commit or one PR), before you report the task
done, add one entry at the end of `docs/llm-authoring.md`. Follow the rules
at the top of that file: write it during the task, quote the real
diagnostic and its code, give counts, and do not soften the cause. Never
change an earlier entry.

Write docs and comments in ASD-STE100 Simplified Technical English.
