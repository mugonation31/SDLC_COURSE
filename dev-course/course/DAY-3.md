# Day 3: Build

## Overview

Today you write the code — but not the way you might be used to. You'll build the `build` **skill** and its **agent**, which implement the feature **test-first** against the executable contract (the acceptance tests): drive the acceptance tests red → green, then make the **whole** suite green locally before handing off. The contract is the definition of done, and you **never edit it** — if the tests seem wrong, that's a question, not an edit.

**What you'll build:**

- `build` **skill** — TDD the contract red → green, then green the full suite; fixes root causes, never touches the contract
- `build` **agent** — wields that skill hands-off: approach in → working code + full suite green, appended to the worklog

**Time estimate:** ~2 hours

> **Why test-first here.** The acceptance tests already exist — they're the contract that came with the ticket. That's a gift: you're not guessing what "done" means — you're making a precise, executable spec go green. Build *to the test*, not to your assumption of it.

---

## Part 1: Where you are (~10 min)

Build runs on Day 2's design. Before you start, confirm:

- [ ] Your `design` skill + agent are working
- [ ] Your carry-through worklog has both `## 1 · Receive & Plan` and `## 2 · Design`
- [ ] The acceptance tests for your story exist and are currently **RED** (the feature isn't built yet) — run `npm run test:acceptance` to see them fail

The worklog thread: Build **reads `## 2 · Design`** (the approach) and **appends `## 3 · Build`**.

The dev chain — you're at step 3:

```
Receive & Plan → Design → Build → Push → Rework
                          (today)
```

---

## Part 2: Build the `build` Skill (~55 min)

### What this skill does

Writing a feature in unfamiliar code normally means jumping between the tests, the code, and the docs, hoping you covered every case. This skill makes the loop disciplined: **read the contract → run it red → write the minimum to go green → green the *full* suite → fix root causes**, looping on itself until the whole tree is green. A green contract isn't done; a green *suite* is.

### Step 1: Open the skill creator

```
/skill-creator
```

### Step 2: Paste this prompt

Copy and paste the following exactly:

```
Create a skill called "build" that implements a feature test-first against an
executable contract (the acceptance tests), then makes the FULL test suite
green locally before handing off to Push.

The skill should:
- Read the worklog file `<TICKET-KEY>-worklog.md` at the repo root and use its
  "## 2 · Design" section (approach + impact map) as its input. If it's missing, stop
  and say Design hasn't run.
- Read the contract first — the acceptance tests (e.g. tests/acceptance/*.test.js) —
  and extract the exact surface they require: function names, signatures, return
  shapes, error messages/regexes, and edge cases. Build to THAT, not to assumptions.
- Run it red: run the acceptance tests and confirm they fail for the expected reason
  (missing feature), not a setup error.
- Write the minimum code to go green, in the place Design pointed to, matching the
  codebase's existing style. Money is integer pence — never floats.
- Re-run the acceptance tests until every one passes.
- Run the FULL suite (npm test). A green contract is not done — a green SUITE is.
- On any failure: fix the ROOT CAUSE (not the symptom), check the blast radius (what
  else calls the code you touched), and re-run the whole suite. Loop on yourself until
  the entire suite is green.
- Append a "## 3 · Build" section to the worklog beneath Design's section (never touch
  sections above yours), with: what changed (files), contract result, full-suite
  result (N passed / N), and notes on any root causes fixed.

Hard rules:
- NEVER edit the acceptance tests / the contract to make them pass. If a test seems
  wrong, that's a clarification question, not an edit.
- Fix root causes, not symptoms — no narrow special-cases to paper over a failure.
- A green contract but a red unit test means you broke something — own it and fix it.
- Keep the change minimal and on-target; don't refactor unrelated code.
- If the contract is genuinely ambiguous or self-contradictory, stop and raise a
  clarification — do not guess.

The description should say: "Stage 3 of the Dev flow. Implements a story test-first
against its executable contract — drives the acceptance tests red → green, then makes
the FULL suite green locally before handing to Push. Use when the acceptance tests
exist and the code must be written."
```

### Step 3: Review the generated skill

Claude Code writes `.claude/skills/build/SKILL.md`. Open it and check it includes:

- [ ] YAML frontmatter with `name: build` and a clear description
- [ ] The TDD loop: read contract → run red → minimum code → green contract → green FULL suite → fix root causes → loop
- [ ] "Never edit the contract" rule
- [ ] "Fix root causes, not symptoms" rule
- [ ] "Money is integer pence" / match existing style
- [ ] It appends `## 3 · Build` to the worklog

### Step 4: Test it

Run the skill against your carry-through worklog:

```
Use the build skill on <TICKET-KEY>-worklog.md
```

### Step 5: Verify the output

- [ ] It read the contract first and listed the exact surface (functions, error regexes, edge cases)
- [ ] It ran the tests **red** before writing code
- [ ] It wrote the feature where Design pointed — minimal, in-style
- [ ] `npm run test:acceptance` is now **green** and the contract file is **untouched**
- [ ] `npm test` (the full suite) is **green**
- [ ] It appended `## 3 · Build` to the worklog with the suite result

### Exercise: prove it's honestly green

- *"Show me the git diff — confirm you didn't touch the acceptance test file."*
- *"Run the full suite and paste the pass/fail counts."*
- *"Which root cause did you fix, and what else calls that code?"*
- *"Is there any special-case in your code that only exists to pass one test?"*

---

## Part 3: Build the `build` Agent (~30 min)

### What this agent does

The agent runs the whole TDD job hands-off: point it at the worklog and it builds the feature, greens the full suite, and appends its section — pausing for your review before anything is pushed.

### Step 1: Create the agent

```
/agents
```

Choose **Create new agent**.

### Step 2: Paste this prompt

```
Create a subagent called "build".

Purpose: given the worklog's design and the executable contract, produce working code
with the FULL test suite green locally, then hand off to Push. It authors code and
tests; it does not push, and it never edits the contract.

It should:
- Invoke the build skill.
- Read the contract (acceptance tests), run it red, write the minimum code to go green
  where Design pointed, then run the FULL suite and fix root causes (not symptoms),
  checking blast radius, until the whole tree is green.
- Append "## 3 · Build" to <TICKET-KEY>-worklog.md beneath Design's section.
- Return a clean summary (what changed + the full-suite pass/fail counts) and stop —
  it does not push.

Tools it needs: Read, Edit, Write, Grep, Glob, Bash.
Hard rules: never edit the acceptance tests; money is integer pence; keep the change
minimal and in-style; if the contract is ambiguous, raise a clarification instead of
guessing.
Description: "Stage 3 of the Dev flow. Implements a story test-first against its
contract and makes the full suite green locally, then hands off to Push."
```

### Step 3: Review the generated agent

Open `.claude/agents/build.md` and check:

- [ ] `name: build` and a clear `description`
- [ ] Tools: Read, Edit, Write, Grep, Glob, Bash
- [ ] It invokes the `build` skill
- [ ] It never edits the contract; integer pence; minimal change
- [ ] It hands off to Push only when the **full** suite is green (it doesn't push itself)

### Step 4: Run it end-to-end

```
Use the build agent on <TICKET-KEY>
```

### Step 5: Verify

- [ ] The agent built the feature from the worklog
- [ ] The full suite is green and the contract file is untouched (`git diff`)
- [ ] It appended `## 3 · Build` to the worklog
- [ ] It stopped without pushing

---

## Part 4: Hand off to Day 4 (~15 min)

Day 4 (Push) confirms your green build in a clean run and ships it. Lock it in:

- Run the agent on your **carry-through ticket** so the worklog now has sections 1–3.
- Confirm with your own eyes: `npm test` green, and `git diff tests/acceptance/` shows **no changes** to the contract.

A genuinely green, contract-untouched build is Push's input tomorrow.

---

## Day 3 Recap

### What you built

- `build` **skill** — TDD the contract red → green, then green the full suite; fixes root causes, never touches the contract; appends `## 3 · Build`
- `build` **agent** — runs that job hands-off and stops before pushing

### The dev chain (you're at step 3)

```
Receive & Plan → Design → Build → Push → Rework
                          (today)
```

### Key concepts

- **The contract is the definition of done — and you never edit it.** Make it green honestly.
- **A green contract isn't enough; a green *suite* is.** Push exists because narrow runs miss regressions.
- **Fix root causes, not symptoms.** A special-case that passes one test is a future bug.
- **Build to the test, not to your assumption of it.** Read the exact surface first.
- **The worklog is the thread.** Build reads `## 2` and appends `## 3`.

### Checkpoint

Before moving to Day 4, confirm:

- [ ] `build` skill saved in `.claude/skills/`
- [ ] `build` agent saved in `.claude/agents/`
- [ ] You ran the agent on your carry-through ticket; the full suite is green
- [ ] The contract file is untouched (`git diff` is clean on `tests/acceptance/`)
- [ ] The worklog now has `## 1`, `## 2`, **and** `## 3`
- [ ] You understand: **build test-first, green the whole suite, never edit the contract**

*Wrote the code. Greened the suite. Left the contract sacred. → Day 4: Push.*
