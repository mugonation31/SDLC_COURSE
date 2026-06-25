# Day 4: Push

## Overview

Today you build the **last local gate before code leaves your machine**. You'll build the `push` **skill** and its **agent**: it takes a build Build says is green, **confirms** it with one clean full-suite run, and either ships it — moving the Jira card to **In Review** on its own — or hands a precise failure report back to Build. Push **verifies only; it never writes code.** This is also where the board starts moving itself again: a green push slides the card from In Progress to In Review.

**What you'll build:**

- `push` **skill** — one clean full-suite run → push on green (and move the card to In Review via MCP) → or a structured *warm report* back to Build on red
- `push` **agent** — wields that skill hands-off: green build in → pushed + card moved, or an actionable failure report out

**Time estimate:** ~2 hours

> **Why a separate confirm step.** "Works on my machine" is where regressions hide. Push re-runs the **whole** suite from a clean state — catching stale local state and anything a narrow run missed — *before* the independent gate ever sees it.

---

## Part 1: Where you are (~10 min)

Push runs on Day 3's green build. Before you start, confirm:

- [ ] Your `build` skill + agent are working; the full suite is green locally
- [ ] Your carry-through worklog has sections `## 1` through `## 3`
- [ ] Jira MCP is still connected (`/mcp` shows **atlassian** connected ✓) — Push moves the card, so it needs Jira

The worklog thread: Push **reads `## 3 · Build`** and **appends `## 4 · Push`**.

The dev chain — you're at step 4:

```
Receive & Plan → Design → Build → Push → Rework
                                  (today)
```

---

## Part 2: Build the `push` Skill (~55 min)

### What this skill does

Push answers one question: **"Is it still green — all of it?"** It runs the full suite once from a clean state as the source of truth. On green, it pushes and moves the Jira card **In Progress → In Review** (handing the build to the independent gate). On red, it does **not** fix anything — it emits a structured *warm report* (test · expected/actual · culprit · repro) and bounces it back to Build. That bounce is the **warm loop**: Push → Build → Push.

> **Two failure loops, so you know where you are.** The **warm** loop is local: Push catches a fail and hands it back to Build to re-green. The **cold** loop (Day 5) is external: the independent gate fails a *pushed* build and bounces it back through Rework. Warm = before it leaves; cold = after.

### Step 1: Open the skill creator

```
/skill-creator
```

### Step 2: Paste this prompt

Copy and paste the following exactly:

```
Create a skill called "push" that is the final local gate before code leaves the
machine. It confirms a green build with one clean full-suite run, then pushes and
moves the Jira ticket to "In Review" — or, on failure, emits a structured warm report
back to Build. It verifies only; it never writes or fixes code.

The skill should:
- Read the worklog file `<TICKET-KEY>-worklog.md` at the repo root for context (Build's
  "## 3 · Build" section); derive the ticket key from the filename for the board move.
- Confirm green from a clean state: run the FULL suite once (npm test) as the source of
  truth. This catches stale local state / clean-env drift that slipped past Build.
- On PASS: push the code, then move the ticket from "In Progress" to "In Review" via
  the Jira MCP. Match the transition by its status NAME ("In Review"), never a
  hard-coded ID — cloned boards reassign IDs. NEVER move the ticket to Done: that
  verdict belongs to the independent E2E gate alone.
- On FAIL: do NOT fix and do NOT move the ticket. Emit a structured warm report and
  hand it back to Build, containing:
  - test — which test(s) failed (file + name),
  - expected / actual — what was asserted vs what happened,
  - culprit — the likely cause / offending change, if identifiable,
  - repro — the exact command to reproduce.
- On return from Build's re-green, run the full suite again (normally one pass; re-loop
  only on a NEW failure).
- Append a "## 4 · Push" section to the worklog with: the clean-run verdict
  (N passed / N), the result (PASSED → pushed, ticket → In Review | FAILED → warm
  report, back to Build), and the warm report if it failed.

Loop-numbering rule (always follow): the stage number is fixed — Push is always "4".
If a "## 4 · Push" section already exists (you're back here after a Rework loop), do
NOT renumber and do NOT overwrite it — append a new section tagged "(pass N)", e.g.
"## 4 · Push (pass 2)".

Hard rules:
- Verify only — never author or fix code. Red means a report back to Build, not an edit.
- Run the WHOLE suite, not just the contract — Push's value is catching what a narrow
  run missed.
- Move the ticket only on a green push — In Progress → In Review, by status name.
- Never mark the ticket Done — that's the independent gate's verdict.
- Never push red.

The description should say: "Stage 4 of the Dev flow. Confirms a green build by running
the FULL suite once in a clean state, then pushes and moves the ticket to In Review —
or, on failure, emits a structured warm report back to Build. Verifies only; never
writes code."
```

### Step 3: Review the generated skill

Claude Code writes `.claude/skills/push/SKILL.md`. Open it and check it includes:

- [ ] YAML frontmatter with `name: push`, a clear description, and the Jira MCP transition tools in `allowed-tools`
- [ ] The procedure: clean full-suite run → push + move to In Review on pass → warm report back to Build on fail
- [ ] "Move the ticket by status name, not ID" + "never push red"
- [ ] "Never mark Done — that's the gate's verdict"
- [ ] The loop-numbering rule (`## 4 · Push (pass 2)` on re-entry)
- [ ] "Verify only — never write or fix code"

### Step 4: Test it

Run the skill on your carry-through ticket (with a green build):

```
Use the push skill on <TICKET-KEY>-worklog.md
```

### Step 5: Verify the output

- [ ] It ran the **full** suite once from a clean state
- [ ] On green: it reported the build pushed, and the Jira card moved **In Progress → In Review** (check the board)
- [ ] It did **not** mark the ticket Done
- [ ] It appended `## 4 · Push` to the worklog with the verdict
- [ ] It wrote **no** code (it only verified)

### Exercise: see the warm loop fire

Deliberately break one small thing to watch Push catch it (then undo it):

- *"Temporarily make one unit test fail, run push, and show me the warm report."* — confirm the report has test · expected/actual · culprit · repro, and that the card did **not** move.
- *"Undo that change, run push again, and confirm green → card moves to In Review."*
- *"Confirm push never edited any source file."*

---

## Part 3: Build the `push` Agent (~30 min)

### What this agent does

The agent runs the confirm-and-ship job hands-off: point it at the ticket, and it re-runs the suite, pushes on green, and moves the card — or bounces an actionable report back to Build — without step-by-step prompting.

### Step 1: Create the agent

```
/agents
```

Choose **Create new agent**.

### Step 2: Paste this prompt

```
Create a subagent called "push".

Purpose: take a build Build says is green, confirm it with one clean full-suite run,
and either push it (moving the ticket to In Review) or hand a precise warm failure
report back to Build. It executes and verifies only — it never writes or fixes code.

It should:
- Invoke the push skill.
- Confirm green from a clean state (npm test).
- On PASS: push, then move the ticket "In Progress" → "In Review" via the Jira MCP,
  matched by status NAME. Never mark it Done.
- On FAIL: do not fix, do not move the ticket — emit a structured warm report
  (test · expected/actual · culprit · repro) back to Build.
- Append "## 4 · Push" to <TICKET-KEY>-worklog.md (a new "(pass N)" section if one
  already exists from an earlier loop).

Tools it needs: Read, Grep, Glob, Bash, and the Jira MCP tools (read the ticket + move
it In Progress → In Review). No code-writing tools.
Hard rules: verify only, never fix code; run the whole suite; move the ticket only on a
green push, by status name; never mark Done; never push red.
Description: "Stage 4 of the Dev flow. Confirms a green build with one clean full-suite
run, then pushes and moves the ticket to In Review — or bounces a warm report back to
Build. Verifies only; never writes code."
```

### Step 3: Review the generated agent

Open `.claude/agents/push.md` and check:

- [ ] `name: push` and a clear `description`
- [ ] Tools: Read, Grep, Glob, Bash **and** the Jira MCP transition tools — but **no** Edit/Write for source code
- [ ] It invokes the `push` skill
- [ ] It moves the ticket only on a green push, by status name; never marks Done
- [ ] It verifies only — it never fixes code

### Step 4: Run it end-to-end

```
Use the push agent on <TICKET-KEY>
```

### Step 5: Verify

- [ ] The agent re-ran the full suite from a clean state
- [ ] On green: the build was pushed and the card moved to **In Review** on its own
- [ ] It appended `## 4 · Push` to the worklog
- [ ] It wrote no code, and never touched the Done column

---

## Part 4: Hand off to Day 5 (~15 min)

After a green push, your card sits in **In Review** — waiting for the *independent* verdict. That verdict is Day 5's job: the **E2E gate**, and the **Rework** loop for when the gate bounces a build back.

- Run the agent on your **carry-through ticket** so the worklog has sections 1–4 and the card is in **In Review**.
- Notice what *didn't* happen: no agent moved the card to **Done**. That's deliberate — Done is the independent gate's call alone, which you'll stand up tomorrow.

---

## Day 4 Recap

### What you built

- `push` **skill** — one clean full-suite run → push + move card to In Review on green; structured warm report back to Build on red; appends `## 4 · Push`
- `push` **agent** — runs that job hands-off, verifies only, moves the board on a green push

### The dev chain (you're at step 4)

```
Receive & Plan → Design → Build → Push → Rework
                                  (done)
```

### Key concepts

- **Push verifies; it never fixes.** Red is a *warm report* back to Build, not an edit.
- **The whole suite, from clean.** That's how Push catches what "works on my machine" hid.
- **The board moves itself.** A green push slides the card In Progress → In Review via MCP, matched by status name.
- **No agent ever sets Done.** That's the independent gate's verdict — anything else makes its independence a lie.
- **Two loops:** warm (Push → Build, local, before it ships) vs cold (gate → Rework, external, after it ships — Day 5).

### Checkpoint

Before moving to Day 5, confirm:

- [ ] `push` skill saved in `.claude/skills/`
- [ ] `push` agent saved in `.claude/agents/`
- [ ] You ran the agent on your carry-through ticket; on green it pushed and the card moved to **In Review**
- [ ] The card is **not** in Done (no agent sets that)
- [ ] The worklog now has `## 1` through `## 4`
- [ ] You understand: **Push confirms from clean, ships on green, moves the board, and never fixes code or sets Done**

*Confirmed from clean. Shipped on green. Moved the board. Left Done to the gate. → Day 5: Rework + the CI Gate.*
