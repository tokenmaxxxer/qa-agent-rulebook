---
date: 2026-07-25
status: fixed
files:
  - signoff/hooks/capture-verdict.sh
  - signoff/hooks/tests/run-verdict-tests.sh
---

# Five ways `capture-verdict.sh` dies quietly

## Summary

`signoff/hooks/capture-verdict.sh` declares "This hook never blocks" (line 7)
and **breaks that contract along five paths.** The first version of this report
(2026-07-25) named only one of them. The other four surfaced while writing a
regression suite, and **one of them was stopping the hook's entire purpose from
running.**

There is one cause. The script runs under `set -euo pipefail` (line 17) while
three of the pipelines that extract a value **begin with a `grep`/`git` whose
"found nothing" is reported as exit 1.** Finding nothing is an absence, not a
failure — but `pipefail` promotes it to a pipeline failure and `set -e` ends the
script.

| # | condition | symptom | line |
|---|---|---|---|
| ① | a symlink anywhere in `QA_WORKSPACE` | does nothing (0 tokens) | 29 |
| ② | no `origin` remote | **exit 2 — the prompt is blocked** | 42 |
| ③ | prompt names no `item <id>` | exit 1 | 74 |
| ④ | the verdict carries no `priority` word | **exit 1, no token minted** | 153 |
| ⑤ | (always) | a grep error on stderr at every mint | 185 |

**④ is the largest.** An ordinary verdict — `item F-1 confirmed defect` —
mints nothing: line 153 dies before reaching the minting block at line 214. It
only survives when `priority` appears in the same turn. The reason the hook
exists was not running.

## Measured

`signoff/hooks/tests/run-verdict-tests.sh`, before and after the fix.

| case | before | after |
|---|---|---|
| no `origin` | **exit 2** | exit 0 |
| prompt names no item | exit 1 | exit 0 |
| `item F-1 confirmed defect` | exit 1, **0 tokens** | exit 0, `F-1.token` |
| `item F-1 priority now` | exit 0, `F-1.priority.token` | unchanged |
| both in one turn | exit 0, 2 tokens | unchanged |
| `item F-9` (absent from state.md) | exit 0, 0 tokens | unchanged |
| bare assent (`ok`) | exit 1 | exit 0, 0 tokens |
| symlinked workspace | exit 0, **0 tokens** | exit 0, `F-1.token` |
| `QA_SIGNOFF_DISABLE=1` | exit 0, 0 tokens | unchanged |

**Nothing that should not mint newly mints.** That the gate did not loosen is
the more important half of this table.

## Cause

### ②③④ — the author's own fallback is unreachable

```bash
17: set -euo pipefail
…
42: slug=$(git remote get-url origin 2>/dev/null | sed -e … )
43: [ -n "$slug" ] || slug=$(basename "$PWD")
```

Line 43 is precisely the fallback for "there is no remote". But the script has
already died on line 42: `2>/dev/null` silences **stderr only** and leaves the
exit status alone — with no remote, `git remote get-url origin` exits 128.

Line 74 (extracting `item <id>`) and line 153 (extracting the `priority` value)
have the same shape. In all three, an empty result is the **normal outcome**,
and the lines immediately following are written to receive an empty value. They
are simply never reached.

### ① — a logical path compared against a physical one

```bash
29: ws="$(cd "$ws" 2>/dev/null && pwd)"           || exit 0   # logical
60: proj_dir_real="$(cd "$proj_dir" … && pwd -P)" || exit 0   # physical
61: case "$proj_dir_real" in "$ws"|"$ws"/*) ;; *) exit 0 ;; esac
```

The containment check itself is right — resolve first, then check. But only one
side was resolved. If `QA_WORKSPACE` passes through a symlink the two values can
never match, and the hook exits 0 having done nothing. On macOS that is `/tmp`,
`/var`, and every `mktemp -d`.

### ⑤ — `iE` is passed as the argument to `-m`

```bash
185: grep -miE 1 -E '(…)'      # → invalid argument, fails every time
```

It is wrapped in `|| true`, so it falls back silently. The result is that a
token's `phrase:` carries **the whole prompt** instead of the verdict line, and
the credential scan on line 190 runs against the whole prompt too. A wider scan
is the safe direction, but an internal URL somewhere unrelated to the verdict
can now refuse a legitimate mint.

## Fix

```diff
-29: ws="$(cd "$ws" 2>/dev/null && pwd)" || exit 0
+29: ws="$(cd "$ws" 2>/dev/null && pwd -P)" || exit 0

-42:  slug=$(git remote get-url origin 2>/dev/null | sed …)
+42:  slug=$(git remote get-url origin 2>/dev/null | sed … || true)

-74:  raw_item="$(echo "$prompt" | grep -ioE … | head -1 | sed -E …)"
+74:  raw_item="$(echo "$prompt" | grep -ioE … | head -1 | sed -E … || true)"

-153:   priority_value="$(echo "$prompt" | grep -ioE … | tail -1 | … | tr …)"
+153:   priority_value="$(echo "$prompt" | grep -ioE … | tail -1 | … | tr … || true)"

-185:   grep -miE 1 -E \
+185:   grep -m1 -iE \
```

Dropping `set -e` was considered and rejected. It would release all three at
once, but it would also silence *genuinely* unexpected failures. `|| true`
states **"finding nothing here is normal"** at each site — which is what is
actually true there — and it is the idiom line 34 already uses.

## Regression suite

`signoff/hooks/tests/run-verdict-tests.sh` runs the hook as a real subprocess
and asserts on the observed exit code and on **which token files the run left
behind**. It never reads the hook's source text. Same approach as
`qa-cycle/hooks/tests/run-gate-tests.sh`.

Reverting the fix **fails 5 of its 9 cases.** ⑤ is masked by the others, so it
was checked separately: reverting only ⑤ fails one case, caught by the
clean-stderr assertion.

Without `jq`, `python3`, or `git` the hook exits 0 at its own dependency check.
Every case would then **pass for the wrong reason**, so the runner refuses to
check at all and exits 2 instead.

## Why none of this was visible

②③④ all fire **only on an absence.** A repository with a remote, a prompt
naming an item, a turn that carries `priority` — the happy path anyone tries by
hand — shows none of the five.

And the failures do not look like failures. ① and ④ exit 0 or merely leave no
token; ② writes nothing to stderr, so the session log carries no reason. `bench`
surfaced them because it copies targets without a tracker (so `/bug` falls
through to `UNFILED(no tracker)`), and therefore without a remote — **only the
rulebook-enabled arm dies, so the ablation is biased against the rulebook.** In
a reps=3 batch, 2 of the 3 rulebook-enabled runs produced no artifact at all.

*Runner and logs: `bench/run.py` in `tokenmaxxxer/muster`.*
