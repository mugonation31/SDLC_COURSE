# Day 5: Rework + the CI Gate (Capstone — full end-to-end)

> **Status: blueprint / outline.** Days 2–4 (Design, Build, Push) must exist first — this day assumes all
> five agents are built and the board already auto-moves To Do → In Progress → In Review (see
> `dev-sdlc-flow-01.md` §4). Flesh out the step-by-step once Days 2–4 are written.

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

---

## Part 1: Build the Rework agent (~45 min)

*(Standard skill→agent build, same shape as Days 1–4. Rework is the only agent that both reads AND writes
Jira: it pulls the gate's failure comment and, when stuck, posts a clarification.)*

- Build the `rework` skill, then the `rework` agent.
- Confirm its MCP tools are wired (pull comment, post comment, transition In Review → In Progress).

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
