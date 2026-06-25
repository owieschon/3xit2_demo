# 3xit2 — a trust layer for an unattended coding agent

[![prove](https://github.com/owieschon/3xit2.demo/actions/workflows/prove.yml/badge.svg)](https://github.com/owieschon/3xit2.demo/actions/workflows/prove.yml)

You start an agent on a task, walk away, and come back to "done." But "done" is
the model's opinion. It may have weakened the test that was failing, run a
subset of the suite, or never exercised the code path it claims to have fixed —
and it can reach past the task on its way there: push the branch, rewrite
history, install a dependency, or edit the very checks that are supposed to
grade it. The first kind of failure is silent until something breaks
downstream. The second is one careless command away.

3xit2 drops into a repo and makes the first failure *mechanically verifiable*
and the second *hard to reach by accident*. It adds three things and nothing
else — no service, no daemon, no package to install. The hard dependency list
is `git`, `bash`, `python3`; all 24 of its Python files import the standard
library only.

## The three components

**1. Boundary guards** — Claude Code PreToolUse hooks
(`.claude/hooks/guard_bash.py`, `.claude/hooks/guard_files.py`) that block a
fixed set of operations before they run. The bash guard refuses `push` and
force-push (including via `bash -c`, env-var prefixes, `git -c
core.hooksPath=`, `git -c alias.*`, and `GIT_CONFIG_*` injection), `merge`,
history rewrites (`rebase`/`filter-branch`/`update-ref`), `reset --hard`,
`commit --amend`, force-deleting branches, dependency adds (`npm/pip/cargo/...
add`), DB migrations, and HEAD-moving ops when there is unpushed work. The file
guard makes the trust layer itself read-only to the agent: any write to
`rails/verifier/`, `.claude/hooks/`, `.claude/settings.json`, the adversarial
eval, or a test listed in `rails/verifier/load_bearing.txt` is refused
(symlink- and case-fold-resolved, so `RAILS/VERIFIER/...` on macOS doesn't slip
through). The agent can add coverage in new files; it can't touch the ones that
judge it. Both guards fail *closed* — unparseable input blocks.

**2. The verifier** (`rails/verifier/verify.sh`) — what actually decides
"done." When the agent claims a task complete, the verifier re-derives the
answer from the current tree instead of trusting the claim. It:

- re-runs the **full test suite** and checks the count against a frozen
  baseline — a run that exits 0 but collects fewer tests than the baseline is a
  false green and fails (`full_suite`);
- confirms the named **load-bearing tests** were actually collected by name, and
  not merely collected but skipped/xfailed (`load_bearing`,
  `exercised_assertions`);
- **greps the changed code onto the live path** — each declared pattern must
  match a line in a file the dispatch actually changed, so a fix to dead code or
  a decoy grep at unrelated shipped code fails (`live_path`);
- proves **demonstrated red→green** for each obligation: revert the fix, prove
  the test goes red, restore it, prove it goes green again. A test that still
  passes with the fix removed proves nothing (`demonstrated_red`);
- checks the agent didn't silently mutate the tests that grade it
  (`oracle_integrity`), and that the **trust layer itself is still proven** —
  if the governor's fingerprint differs from the one stamped at the last eval
  pass, it refuses to certify anything until the eval is re-run
  (`governor_proven`).

It writes `rails/evidence/<id>/verdict.json` stamped with a hash of the working
tree. `--update-baseline` (and lowering the baseline with `--allow-shrink`) is
human-only — the bash guard blocks the agent from invoking it.

**3. The Stop gate** (`.claude/hooks/gate_stop.py`) — a Stop hook that refuses
to end a session while a dispatch is active unless the verifier produced a fresh
`PASS` whose stamped tree hash matches the *current* tree. Stale green is not
green: change a byte after the verifier ran and the PASS no longer counts. The
only other way out is for the agent to formally declare itself blocked
(`rails/handoff/<id>.BLOCKED.md`, stating what it needs from the human). This
gate fails *open* by deliberate design — it gets its input from Claude Code, not
the agent, and a hook that fails closed would brick the session rather than
deny one command. The asymmetry is documented in the code.

## The judge proves itself

Writing a checker is easy; knowing it actually fires is the whole game. So the
trust layer ships its own adversarial eval — **50 cases (48 core + 2 kit-dev)**
under `rails/adversarial/cases/`. Each builds a disposable sandbox, injects one
real violation — a weakened check, an off-live-path fix, a vacuous test, a
forged verdict, a stale green at the Stop gate, a `git -c` push vector, a
case-variant path, an untracked-content swap — and asserts that the specific
guard or verifier check catches it, *and* that clean work is not falsely
flagged.

```bash
bash rails/adversarial/run_eval.sh     # builds 50 sandboxes; give it a minute
```

It passes with zero failures on a clean checkout. CI (`.github/workflows/prove.yml`)
re-runs it on every push and PR with `RAILS_NO_STAMP=1`, and additionally runs
`verify.sh` end-to-end against a fixture dispatch — so CI *proves* the layer but
is forbidden to *stamp* it. Marking the trust layer proven stays a local,
human-released act. The agent being judged cannot edit the judge, freeze its own
proof obligations, or write its own PASS.

## Run it

```bash
./install.sh /path/to/your/repo     # never clobbers existing files
```

Then edit `rails/config.json` (test command, count regex, branch), seed the
baseline from a known-good run, and run the eval to stamp the registry:

```bash
bash rails/verifier/verify.sh BOOTSTRAP --update-baseline   # seed baseline
bash rails/adversarial/run_eval.sh                          # prove + stamp
```

Restart Claude Code in the repo to pick up `.claude/settings.json`. The full
daily loop — drop a spec in `rails/dispatches/inbox/`, `/dispatch`, approve,
walk away, read the handoff — is in [`docs/OPERATING.md`](docs/OPERATING.md).

## Honest about what this is

This is a working prototype and a **trust layer, not a security sandbox**. It
holds against a cooperative agent that cuts corners under pressure; it is
defeatable by one actively trying to escape — the bash guard is a pattern
matcher, and no regex over a shell string can stop an agent that writes a file
through a language interpreter. That class is architectural; the optional
[`isolate/`](isolate/) companion closes it by running the loop in a container
with the trust layer mounted read-only and the network off. Every hole reproduced
in a disposable sandbox — what's closed, what's hardened, and the one kept
open by design — is written up in [`AUDIT.md`](AUDIT.md). It's a real
self-audit, not a disclaimer.

This is the public, sanitized version of a real working system. The whole repo
exists to make one word — "done" — mean something the agent can't fake.
