# silent-degradation-guard

Catch the fake results your scripts produce when they silently degrade.

A skill for Claude Code, WorkBuddy, Codex and any SKILL.md-compatible agent.

## Install

```bash
npx skills add zihao0639-netizen/silent-degradation-guard
```

or via SkillHQ:

```bash
skillhq install silent-degradation-guard
```

## What it does

**A fake result is far more dangerous than a crash.** A crash gets fixed immediately; a
fake result gets used for decisions.

This skill catalogs five recurring ways scripts produce plausible-looking lies:

1. `try: import heavy except: skip` — "0/25 passed" when nothing was tested
2. `TABLE.get(x, DEFAULT)` + length fallback — three inputs, one output
3. engine chain `A or B or C` — private data quietly uploaded while the log says "local"
4. **field-dropping at choke points** — adding a contract field dies at an object-rebuild
   site that only copies some fields; compiles, no crash, clean logs, field has no effect
5. **「unverifiable」 branches outside the aggregation** — every criterion "cannot verify",
   the aggregate exit code is 0, and the whole checker reports all-pass

## Why it exists

Each pattern was found in production. Pattern 5 is the nastiest: it is not one criterion
broken, it is the **entire checker reporting qualified while unable to work** — usually
because someone fixed the same-kind branches in one place (A5/A6) and never went back to
patch the others (A1~A3).

The skill ships the universal self-check that catches all of them:

> **Point the checker's input at an empty directory. The exit code must be non-zero.
> If it returns 0, the checker is fail-open and must not ship.**

Plus the fix ladder (auto-switch interpreter → fail hard → visible degradation), an
implementation template, and acceptance criteria that refuse the happy path.

## Contents

```
skills/silent-degradation-guard/
└── SKILL.md    # the full methodology (single self-contained file)
```

## License

MIT
