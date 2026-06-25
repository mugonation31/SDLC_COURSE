# Day 5: Rework + the CI Gate (Capstone — full end-to-end)

## Overview

Today is the capstone. You'll build the **last agent — Rework** — and then stand up the **CI gate**: a
robot in the cloud that becomes the *independent judge* of "done." With the gate in place, you run the
**whole flow end-to-end, untouched by hand**: a ticket marches from To Do all the way to **Done**, and a
deliberately broken build gets bounced back through Rework — all on its own.

**What you'll build:**

- `rework` **skill + agent** — pulls the gate's failure report from Jira, finds the root cause, self-greens
  the full suite, hands back to Push (or posts a dev-approved clarification and waits).
- **The CI gate** — a GitHub Actions workflow that runs the full regression on every push, then moves the
  Jira ticket to **Done** on green or bounces it back (comment + In Progress) on red.

**Time estimate:** ~2.5 hours

> **Where you are.** This is the capstone — it assumes Days 1–4 are done: all four agents (`receive-and-plan`, `design`, `build`, `push`) work, your carry-through worklog has sections `## 1` through `## 4`, the card is in **In Review** after a green push, and Jira MCP is connected. Today adds the fifth agent (Rework) and the independent gate that finally sets **Done**.

The dev chain — you're at the final step:

```
Receive & Plan → Design → Build → Push → Rework
                                          (today)
```

---

## Part 1: Build the Rework agent (~45 min)

Same skill→agent shape as Days 1–4. Rework is the **cold loop**: when the independent gate fails a *pushed* build and posts the reason to Jira, Rework pulls that report, finds the root cause, re-greens the full suite, and hands back to Push. It's the only agent that both **reads and writes** Jira — it pulls the failure comment and, when the report is unclear, posts a dev-approved clarification.

### What this skill does

A cold failure is one your local runs missed — it only surfaced in the gate's clean environment. Rework turns that into *"why + a patch"*: pull the gate's comment, reproduce, fix the **root cause** (not the symptom), green the whole suite, and hand back to Push — never editing the contract.

### Step 1: Open the skill creator

```
/skill-creator
```

### Step 2: Paste this prompt (the skill)

```
Create a skill called "rework" — stage 5 of the Dev flow, the cold loop. The
independent E2E gate failed a pushed build and posted the reason to Jira; this skill is
the Dev's response. It pulls that report, finds the root cause, re-greens the full
suite, and hands back to Push. It fixes; it does not judge — the gate already judged.

The skill should:
- Pull the latest gate failure comment from the ticket via the Jira MCP — the reason
  and suggested fix.
- Move the ticket back from "In Review" to "In Progress" via the Jira MCP, matched by
  status NAME (never a hard-coded ID), so the board shows it's being reworked.
- Read the worklog file `<TICKET-KEY>-worklog.md` first for the full history of what
  Plan/Design/Build/Push already did.
- Reproduce the failure locally, find the ROOT CAUSE (not the symptom), and decide the
  fix.
- Self-green like Build: apply the fix → run the FULL suite (npm test) → on any failure
  fix the root cause + check blast radius → re-run, looping on yourself until the whole
  suite is green. Never hand Push a red tree.
- Append a "## 5 · Rework" section to the worklog with: the gate failure, the root
  cause (not the symptom), the fix (files changed), and the full-suite result.
- When the report is NOT actionable (vague, can't reproduce, or the fix breaks
  something else): do NOT guess and do NOT push. Draft a clarification question, get
  the DEV's approval, post it to Jira via MCP, and wait for the reply.

Loop-numbering rule (always follow): the stage number is fixed — Rework is always "5".
If a "## 5 · Rework" section already exists (the gate bounced more than once), append a
new one tagged "(pass N)", e.g. "## 5 · Rework (pass 2)".

Hard rules:
- Pull, don't guess — work from the actual gate report via MCP.
- Move the ticket back to In Progress as you pick the failure up; let Push move it
  forward again.
- Fix root causes, not symptoms; check blast radius before declaring green.
- Never hand Push red, and NEVER edit the acceptance tests / the contract.

The description should say: "Stage 5 of the Dev flow (the cold loop). Pulls the E2E
gate's failure report from Jira via MCP, turns it into a root-cause fix, self-greens
the full suite, and hands back to Push — or, if the report is unclear, posts a
dev-approved clarification and waits. Use when the E2E gate fails a pushed build."
```

### Step 3: Review the generated skill

Open `.claude/skills/rework/SKILL.md` and check it includes:

- [ ] `name: rework`, a clear description, and the Jira MCP tools (get issue, **add comment**, transition) in `allowed-tools`
- [ ] The procedure: pull comment → move to In Progress → reproduce → root-cause fix → green full suite → append `## 5 · Rework`
- [ ] The "report not actionable → dev-approved clarification, then wait" path
- [ ] The loop-numbering rule (`## 5 · Rework (pass 2)`)
- [ ] "Never edit the contract" + "fix root causes, not symptoms"

### Step 4: Create the agent

```
/agents
```

Choose **Create new agent**, then paste:

```
Create a subagent called "rework".

Purpose: the cold loop. The independent E2E gate failed a pushed build and posted the
reason to Jira. Pull that report, turn it into "why + a patch", self-green the full
suite, and hand back to Push. Fix; do not judge.

It should:
- Invoke the rework skill.
- Pull the latest gate comment from the ticket via the Jira MCP.
- Move the ticket "In Review" → "In Progress" via MCP (matched by status name) as it
  picks the failure up.
- Reproduce locally, fix the ROOT CAUSE, then self-green like Build: run the full suite
  and loop until the whole tree is green. Never hand Push red. Never edit the contract.
- Append "## 5 · Rework" to <TICKET-KEY>-worklog.md (a new "(pass N)" section if one
  already exists).
- If the report isn't actionable: draft a clarification, get DEV approval, post it to
  Jira, and wait — do not guess, do not push.

Tools it needs: Read, Edit, Write, Grep, Glob, Bash, and the Jira MCP tools (get the
issue/comments, add a comment, transition the ticket).
Description: "Stage 5 of the Dev flow (the cold loop). Pulls the gate's failure report
from Jira, fixes the root cause, self-greens the full suite, and hands back to Push —
or posts a dev-approved clarification and waits."
```

### Step 5: Review the generated agent

Open `.claude/agents/rework.md` and check:

- [ ] `name: rework` and a clear `description`
- [ ] Tools: Read, Edit, Write, Grep, Glob, Bash **and** the Jira MCP tools (incl. add-comment)
- [ ] It invokes the `rework` skill
- [ ] It moves the ticket In Review → In Progress as it starts; never edits the contract; never hands Push red

*(You'll run `rework` for real in Part 4, once the gate has something to bounce.)*

---

## Part 2: What is a "CI gate"? (~15 min, concept)

Until now **you** have played the gate — running the tests by hand and deciding pass/fail. That works, but
the judge is a human. **CI ("Continuous Integration") replaces that human with a robot:** every time code is
pushed, a clean cloud machine runs the full test suite automatically and reports the verdict.

Why this matters for *our* flow: the gate must be **independent** of the developer. A robot that runs the
exact same tests in a clean environment *is* that independent judge — which is why **the robot, not any
agent, is allowed to set the ticket to Done.** (No dev agent marks its own work done — that would make the
gate's independence a lie.)

We use **GitHub Actions** — free, built into GitHub, one config file.

---

## Part 3: Stand up the gate (~45 min)

**3.1 — Put the repo on GitHub** (if not already). The student's clone needs a GitHub remote so Actions can run.

**3.2 — Add the workflow file** `.github/workflows/e2e-gate.yml`. In plain terms it says:
- *When:* on every push.
- *Do:* spin up a clean Ubuntu machine → `npm ci` → `npm test` (the full regression).
- *On green:* call the Jira API to transition the ticket to **Done**.
- *On red:* call the Jira API to **post the failure as a comment** and move the ticket back to **In Progress**.

**3.3 — How the robot knows which ticket:** derive the ticket key from the **branch name** (e.g.
`PROJ-123-discount-code`) or the commit message. Standard pattern — teach one and stick to it.

**3.4 — Secrets (so the robot can log into Jira):** add repo secrets in GitHub Settings → Secrets:
`JIRA_BASE_URL`, `JIRA_EMAIL`, `JIRA_API_TOKEN`, `JIRA_CLOUD_ID`. The workflow reads these — they are never
committed to the repo.

> **Note:** CI authenticates to Jira with an **API token** (REST API), not the MCP server. MCP is how *your
> agents* (Claude Code) reach Jira; the **robot** is a plain script, so it uses Jira's REST API directly.
> Same board, two different doors in.

---

## Part 4: Mimic the gate by hand (~20 min) — the no-CI cold loop

> **This is the reliable path.** It exercises the entire Rework loop **without** standing up
> CI, GitHub, or secrets. Do this first; treat Part 5 (real CI) as the stretch/automation
> upgrade. You (the tutor) play the independent gate; the student plays the Dev.

The whole loop needs just **two artifacts**: a failing gate test and a Jira comment. Both are
already prepared for you — the gate test **ships dormant** in the repo so the suite is green
all the way through Build and Push.

### How the gate failure ships (dormant → active)

- The file `tests/e2e/discount-case.test.js` is present from `day1-start`, but Jest **ignores**
  `tests/e2e/` so `npm test` stays green on Days 1–4. In `package.json`:

  ```json
  "jest": { "testPathIgnorePatterns": ["/node_modules/", "<rootDir>/tests/e2e/"] }
  ```

- The defect it catches: discount lookup is **case-sensitive** (`DISCOUNTS` is keyed UPPERCASE),
  so a customer typing `save10` is wrongly rejected. The 7 ACs only ever use uppercase, so Build
  ships legitimately green and this slips past — exactly what an independent regression catches.

### The gate test (already in the repo)

```js
// tests/e2e/discount-case.test.js  — SDET gate regression, NOT the dev contract
const { createCart, addItem } = require('../../src/cart');
const { applyDiscount, total } = require('../../src/pricing');

function cartWith(productId, qty) {
  const cart = createCart();
  addItem(cart, productId, qty);
  return cart;
}

describe('discount code is case-insensitive (E2E gate regression)', () => {
  test('a lowercase percentage code is accepted just like its uppercase form', () => {
    const cart = cartWith('SKU1', 2);   // subtotal 2000
    applyDiscount(cart, 'save10');      // customer typed lowercase
    expect(total(cart)).toBe(1800);     // 10% off 2000, same as SAVE10
  });

  test('a mixed-case fixed-amount code is accepted', () => {
    const cart = cartWith('SKU1', 2);   // subtotal 2000
    applyDiscount(cart, '5Off');
    expect(total(cart)).toBe(1500);     // 500 off 2000
  });
});
```

### Step 4.1 — Tutor: fire the gate (play the SDET)

1. **Activate the gate test** — remove `"<rootDir>/tests/e2e/"` from `testPathIgnorePatterns`
   in `package.json` (or check out `day5-broken`, which has it activated).
2. **Prove the cold-failure state:** `npm test` → **2 failed (e2e), contract + units still green.**
   This is the point — local Build/contract was green; only the clean regression catches it.
3. **Post the failure report as a Jira comment** on the carry-through ticket (this is what
   `rework` pulls via MCP). Paste this:

   ```
   🔴 E2E GATE: FAILED — build bounced from In Review

   Finding — discount codes are case-sensitive. The checkout flow accepts what a
   customer types, but applyDiscount() looks the code up verbatim against the seeded
   table (keyed UPPERCASE), so a lowercase/mixed-case code is wrongly rejected.

   Reproduce: cart SKU1 x2 (subtotal 2000) → applyDiscount(cart, 'save10')
     Expected: accepted, total = 1800 (same as SAVE10)
     Actual:   throws "Unknown discount code: save10"

   Failing gate test: tests/e2e/discount-case.test.js (2 assertions). Contract suite
   still green — a gap the contract never exercised, not a contract regression.

   Dev: fix the ROOT CAUSE in src/pricing.js (normalise the code before lookup).
   Do NOT edit the acceptance contract or this gate test to make it pass.
   — SDET / E2E gate
   ```
4. **Leave the card In Review.** Rework moves it back to In Progress itself.

### Step 4.2 — Student: run the cold loop

1. Notice the ticket has a new comment (in real life: a Jira/email alert). Read it.
2. Run **`/rework`**. Watch it:
   - pull the comment from Jira via MCP,
   - move the card **In Review → In Progress**,
   - root-cause it to the verbatim lookup and patch `src/pricing.js` (e.g. `code.toUpperCase()`),
   - re-green the **full** suite (`npm test` → all green, contract + gate),
   - append `## 5 · Rework` to the worklog and hand back to **Push**.
3. Run **`/push`** again → card moves **In Progress → In Review**. The loop is closed.

> **The visible proof** — students don't take anyone's word for red/green. They run `npm test`
> in their own terminal and watch Jest print **✕ 2 failed → ✓ all passed**. Run with
> `npm test -- --watch` to see it flip live as Rework edits the file.

---

## Part 5: The full end-to-end run (~30 min) — the payoff

Run the entire flow with **zero manual board-dragging**:

1. **Receive & Plan** `PROJ-123` → card moves **To Do → In Progress**. ✨
2. **Design → Build** → TDD the contract red → green, full suite green locally. *(card stays In Progress)*
3. **Push** → green push → card moves **In Progress → In Review**. ✨
4. **CI gate fires automatically** → clean-env full regression →
   - **PASS →** robot moves card **In Review → Done**. ✅✨
   - **FAIL →** robot comments the reason + moves card back to **In Progress**.
5. **Rework** picks up the gate's comment → finds root cause → self-greens → hands back to **Push** → which
   re-pushes → gate re-runs → **Done**. ↺

**The deliberate-break demo:** reuse the **same case-sensitivity defect from Part 4** — but now the
**robot** is the gate, not you. Activate `tests/e2e/discount-case.test.js` (it passes the contract locally,
fails the clean-env regression in CI), push, and watch the cold loop fire on its own — CI red → robot
comments + bounces to In Progress → Rework → re-push → CI green → robot sets **Done**. *This* is the moment
the whole course clicks. (In Part 4 you posted the comment by hand; here the workflow's red-path does it.)

---

## Done means

All five agents built, the CI gate live, and a ticket driven from **To Do to Done** — including one trip
through the cold Rework loop — with no human moving a single card. The board is a live, honest mirror of the
work, start to finish.
