---
name: silent-degradation-guard
description: 'Detect and fix silent degradation in scripts — when a missing dependency or a failed parse is swallowed and the script keeps running, producing a plausible-looking but fake result. Use when a health-check or validation script reports 0 or all-failed, when default fallbacks make every output identical, or when a missing key silently routes to another engine. Covers five recurring patterns including field-dropping at object-rebuild choke points and unverifiable branches leaking outside the aggregation so the whole checker reports all-pass. Ships the universal self-check: point the input at an empty directory and the exit code must be non-zero; if it returns 0 the checker is fail-open and must not ship.'
license: MIT
compatibility: 'Needs macOS or Linux with bash, grep and python3. Works in Claude Code, Claude.ai, Codex, Cursor and any SKILL.md-compatible agent.'
metadata:
  author: zihao0639-netizen
  version: '1.0.0'
  tags: 'scripting,debugging,validation,fail-open,silent-failure'
---

# Silent Degradation Guard

## Core judgment

**A fake result is far more dangerous than a crash.**

A crash makes you fix it immediately; a fake result gets used for decisions.
Typical symptoms: a health check reports "0/25 passed", every score is the same value,
multiple characters synthesize to one voice, something that should run locally quietly goes to the cloud.

## Three high-frequency patterns

| Pattern | What the fake result looks like |
|---|---|
| `try: import heavy_dep except: skip_check; return` | "0/25 passed" — nothing was actually tested |
| `TABLE.get(x, DEFAULT)` + length fallback | three different inputs parse to the same output |
| engine chain `A or B or C`, A missing config silently goes to B | private data uploaded to the cloud, while the log still says "local" |

## Pattern 4: 「field-dropping at choke points」 when adding a field

**When a contract/protocol gains a new field, changing the definition is not enough — the data must pass
through several "object-rebuild" choke points; anywhere that rebuilds with only some fields, the field dies there.**

Symptoms match the first three: **compiles / does not crash / clean logs / field has no effect**.
The most dangerous part: it **gets misread as "upstream never sent it"** — you go suspect the backend, a colleague, the network.

```
upstream → [choke 1] → [choke 2] → usage site
            ↑ rebuilds with only {a,b,c}, the new d is gone
```

| Language | Typical choke pattern |
|---|---|
| Swift | `list.map { Foo(a: $0.a, b: $0.b) }` (custom struct rebuild) |
| JS/TS | `arr.map(it => ({ a: it.a, b: it.b }))`, `push({ a, b })` (allowlist object literal) |
| Python | `{k: v for k in ["a","b"]}`, `dataclasses.replace` listing only some fields |

**Detection (mandatory when adding a field)**:
```bash
grep -rn "ObjectName(" src/          # Swift/Java constructor sites
grep -rn "{[[:space:]]*[a-z]*:" src/*.js   # JS object literals
```
**List every constructor site on both sides and check one by one.** Never rely on "I remember there is only one".

**Root fix (recommended)**: **give the new field no default value** → any missed constructor site **fails at compile time**.
★ never add a default value "to change fewer places" — that reopens the door you just closed,
and the 5th field will silently drop again. The cost of one `nil` / `None` is far lower than debugging one phantom failure.

**Two secondary rules**:
1. Multiple choke points often sit on **different execution paths** (normal / degraded / concurrent) →
   it shows up as "**sometimes works, sometimes not**", easily mistaken for a flaky issue. → after fixing, **verify every path**, not just the happy path.
2. **each side reconciles against a different table** is a same-family frequent error: write the check as
   **"sets are exactly equal"**, not "contains" — otherwise both sides think they passed and both logs are clean.

## Pattern 5: **「unverifiable」 branches leak outside the aggregation** → the whole check reports "all-pass"

**Shape**: a multi-criteria script where each criterion returns three states (`✅ / ⚠️ / ⚪ "cannot verify"`), and an **aggregate expression** computes the exit code.
If **a criterion's `⚪` branch is not recognized by the aggregate** ⇒ when all criteria are "cannot verify", exit code = **0 = pass**.
This is worse than "one criterion silently degraded": **it is not one criterion broken, the entire checker reports qualified while unable to work.**

**Example** (a disk-hygiene check script, measured):
```python
code = 2 if any(v.get("fatal") for _k, v in marks) else (1 if bad else 0)
#           ↑ only A5/A6's ⚪ branches carry "fatal";
#             A1/A2/A3's 「directory missing」 branch returns ⚪ but 【omits it】⇒ not in the aggregation
```
→ `ROOT` points at an empty directory ⇒ `A1 ⚪(no fatal) · A2 ⚪(no fatal) · A3 ⚪(no fatal)` ⇒ **aggregate exit code = 0**.

**The root cause is often "fixed one spot, missed the rest"**: in the same file the A5/A6 branches of the same kind **already had** `fatal` —
someone did A5/A6 right, **without going back to patch A1~A3**. ⇒ when reviewing such scripts, **line up the same-kind branches side by side**,
and check whether every `⚪` / `except` / "cannot read" branch **is caught by the aggregate**.

### 🔴 Universal self-check (run this first when reviewing any checker/health-checker)

> **Point its `ROOT` (or input directory) at a 【nonexistent / empty】 location ⇒ the exit code must be non-zero.**
> If it returns `0` ⇒ **the checker is fail-open; it must not ship.**

```python
L.ROOT = "/tmp/empty_dir"          # reproduce "input entirely unavailable"
marks = [(n, f()) for n, f in (("A1", L.check_a1), ("A2", L.check_a2), ("A3", L.check_a3))]
bad = [k for k, v in marks if v["verdict"] == "⚠️"]
code = 2 if any(v.get("fatal") for _k, v in marks) else (1 if bad else 0)
assert code != 0, "🔴 fail-open: cannot-verify was treated as all-pass"
```
★ **write it into the contract**: **every `⚪` (cannot verify) branch must make the aggregate exit code non-zero** — either by "flag fatal" or "fall to `3`",
**the one thing not allowed is "not participating in the aggregation"**.

## Fixes (by priority)

1. **Auto-switch to the right environment**: on a missing dependency, `execv` into an interpreter that has it and rerun
   (guard against loops with an env var, switch at most once)
2. **If it cannot switch, fail hard**: non-zero exit code + print two concrete fixes.
   never deliver a "partially usable" result
3. **If a fallback is truly unavoidable → it must leave a trace**: log/output states "degraded, reason X",
   so at least the fake result is visible

## Implementation template

```python
def ensure_env(pkg="some_vision_lib"):
    try:
        __import__(pkg); return
    except Exception:
        pass
    if os.environ.get("_REEXEC"):      # already switched once and still failing → do not switch again
        error_exit(pkg)
    for py in candidate_interpreters:
        if py.is_file() and installed(py, pkg):
            print(f"⚠ missing {pkg}, auto-switching to {py} and rerunning")
            sys.stdout.flush(); sys.stderr.flush()   # ← critical, see below
            os.environ["_REEXEC"] = "1"
            os.execv(str(py), [str(py), os.path.abspath(sys.argv[0])] + sys.argv[1:])
    error_exit(pkg)
```

## Two secondary pitfalls you will step on

1. **`execv` discards the stdout buffer** — without `flush()`, that "switching now" line never prints,
   and the user just sees a mysterious rerun. Always happens with piped output, never when attached to a terminal, so local testing easily misses it.
2. **`grep` saying "not found" ≠ truly not found** — non-ASCII paths get escaped by `core.quotepath` into `\345\275\242...`,
   run `git config core.quotepath false` before verifying, or you will think the file was never committed.

## Acceptance (do not just run the happy path)

| Path | Expectation |
|---|---|
| Correct environment | normal result, no switch message |
| Wrong environment | **message visible** + auto-switch + correct result (not 0) |
| Missing in all environments | non-zero exit code + readable fix |
| Still failing after switch | no repeated switching, error out immediately |

**Key assertion**: different inputs must produce different outputs.
(multiple character voices, multiple input files — after parsing they must all differ, otherwise something was silently defaulted.)
