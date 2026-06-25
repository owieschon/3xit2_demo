# 3xit2 — a trust layer for an unattended coding agent

[![prove](https://github.com/owieschon/3xit2.demo/actions/workflows/prove.yml/badge.svg)](https://github.com/owieschon/3xit2.demo/actions/workflows/prove.yml)

You walk away from the keyboard. The agent comes back and says "done."

That word is the whole problem. "Done" might mean it weakened a test, ran a
subset, edited the code path nobody calls, or never exercised the thing it
claims to have fixed. By the time you find out, the bad change is already in.
I've spent enough years where a confident wrong answer gets caught downstream —
on a loading dock, in a customer's hands — to not trust an agent's word that
its work is finished. So I don't. I made the word mean something.

3xit2 turns "done" into a mechanical decision the agent doesn't get to make.

## What it actually checks

When the agent claims a task is complete, the verifier re-derives that from
reality instead of taking its word:

- **Re-runs the full suite** and checks the test count against a frozen
  baseline, so a deleted test can't hide.
- **Greps the diff onto the live path** — the changed code has to be reachable,
  not dead.
- **Demonstrates red→green** for each obligation: it reverts the fix, proves the
  test goes red, restores it, proves it goes green. A test that passes with the
  fix removed proves nothing.
- **Stamps the verdict with a working-tree hash**, so a stale PASS can't be
  reused after the tree changes.

Two hooks sit in front of that. `guard_files.py` and `guard_bash.py` are
PreToolUse boundary guards that block push/merge/force-rebase/amend, dependency
adds, and — the cardinal rule — **any write to the trust layer itself**. The
verifier, the hooks, the settings, and the eval are read-only to the loop. A
`Stop` gate refuses to let a session end mid-task without a fresh PASS.

## The part I care about: the judge proves itself

It's easy to write a checker. It's the whole game to know the checker actually
fires. So the trust layer ships with an adversarial eval — **50 cases, 48 core
+ 2 kit-dev** — that builds a disposable sandbox for each one, injects a real
violation (a weakened check, an off-path fix, a forged verdict, a stale green, a
git-config push vector, an untracked-content swap), and proves the right guard
or verifier check catches it without falsely flagging clean work.

```bash
bash rails/adversarial/run_eval.sh
```

Runs on this clean checkout, exit 0, all 50 PASS. It builds 50 sandboxes, so
give it a couple of minutes. CI re-runs it on every push and PR — but with
`RAILS_NO_STAMP=1`, so **CI proves and is forbidden to stamp**. Marking the
trust layer "proven" stays a local, human-released act. The agent being judged
cannot edit the judge, freeze its own manifest, or write its own pass.

This is the claim. The measurement of whether enforcement like this actually
cuts an agent's defect rate is a separate preregistered experiment —
[code_ver / TrustLadder](https://github.com/owieschon).

## Run it

Zero runtime dependencies: `git`, `bash`, `python3`. All 24 Python files are
stdlib only — nothing to install. Drops into a repo:

```bash
./install.sh /path/to/your/repo     # never clobbers existing files
```

Then fill `rails/config.json` (test command, count regex, branch), seed the
baseline, and run the eval to stamp the registry. The full daily workflow —
dispatch, approve, walk away, read the handoff — is in
[`docs/OPERATING.md`](docs/OPERATING.md).

## Honest about the edge

This is a discipline tool for a well-meaning agent that cuts corners under
pressure, not a sandbox against a hostile one. The bash guard is a pattern
matcher: it can't stop an agent that writes files through a language
interpreter. That class is architectural and closed by the optional
[`isolate/`](isolate/) container — trust layer mounted read-only, network off,
a kernel boundary instead of a pattern. The guards fail closed; the Stop gate
fails open by deliberate design. Every hole I reproduced, what's closed and the
one that's open-by-design, is in [`AUDIT.md`](AUDIT.md) — a real self-audit,
re-run in disposable sandboxes, not a disclaimer.

---

This is the public, sanitized version of a real working system. The stakes here
are narrow and specific: an unattended agent that says "done" when it isn't, and
a bad change already merged before anyone looks. The whole repo exists to make
that one word mean something the agent can't fake.
