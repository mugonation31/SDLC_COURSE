# Day 2: Design

## Overview

Yesterday you turned a ticket into a scoped plan. Today you answer the next question **before** writing any code: *"What will this touch, and what breaks?"* You'll build the `design` **skill** and its **agent** — the stage that actually reads the codebase, traces every caller of the code you're about to change, and commits to an approach. Receive & Plan scoped *the work*; Design scopes *the risk*.

**What you'll build:**

- `design` **skill** — reads the plan from the worklog + the real code, traces the blast radius, and writes a chosen approach + impact map
- `design` **agent** — wields that skill hands-off: worklog in → approach + impact map appended, paused for your review

**Time estimate:** ~2 hours

> **Why this stage exists.** The most expensive bugs come from *hidden coupling* — you change one function and three callers you forgot about break in a clean environment later. Design's whole job is to surface that coupling on purpose, now, while it's cheap.

---

## Part 1: Where you are (~10 min)

Design builds directly on Day 1's output. Before you start, confirm:

- [ ] Your `receive-and-plan` skill + agent are working
- [ ] You ran the agent on your carry-through ticket, so `<TICKET-KEY>-worklog.md` exists at the repo root with a `## 1 · Receive & Plan` section (task list + scope)
- [ ] Jira MCP is still connected (`/mcp` shows **atlassian** connected ✓)

The worklog is the thread: Receive & Plan created it, and today Design **reads section 1 and appends section 2**. Each stage reads what came before and adds its own.

The dev chain — you're at step 2:

```
Receive & Plan → Design → Build → Push → Rework
                 (today)
```

---

## Part 2: Build the `design` Skill (~55 min)

### What this skill does

Picking up a plan and asking "what does this actually affect?" normally means 30–45 minutes of reading code, grepping for callers, and second-guessing what might break. This skill collapses that into one move — **plan in → approach + impact map out** — with the blast radius traced for real, not guessed.

### Step 1: Open the skill creator

In your Claude Code session, type:

```
/skill-creator
```

(Or simply: *"Create a new skill for me"* — Claude Code scaffolds the files in `.claude/skills/`.)

### Step 2: Paste this prompt

Copy and paste the following exactly:

```
Create a skill called "design" that takes the plan from the worklog and the real
codebase, and produces a chosen approach plus an impact map — answering "what will
this touch, and what breaks?" — before any code is written.

The skill should:
- Read the worklog file `<TICKET-KEY>-worklog.md` at the repo root and use its
  `## 1 · Receive & Plan` section (task list + scope) as its input. If the file or
  that section is missing, stop and say Receive & Plan hasn't run.
- Read the real code at the touch points the plan named — functions, exports, data
  shapes (e.g. money in integer pence), and the conventions already in use around them.
- Trace the blast radius: search the repo for every caller/dependent of the code that
  will change, and ask "what else depends on this?". Hidden coupling is the pain to kill.
- Choose an approach: the smallest change that satisfies the contract (the acceptance
  tests) without breaking callers, in keeping with the existing style. If there are
  real alternatives, name them and say why you picked one.
- Append a "## 2 · Design" section to the worklog, beneath Receive & Plan's section
  (never touch sections above yours), containing:
  1. Approach — where the change lands and how it's shaped.
  2. Change set — the files/functions to change and what the change is.
  3. Blast radius / callers — what could be affected.
  4. Tests in play — which existing tests protect you, which might newly fail.
  5. Risks / unknowns — and how the approach mitigates them.
- Do NOT write production code. Design only — the output is the plan Build executes.

Guidelines: Read before you decide — no approach without having traced the actual
callers. Honour existing conventions and data shapes (integer pence, module
boundaries). The approach must be checkable against the contract; if it can't satisfy
the acceptance tests, rethink it. Surface coupling and risk explicitly — an unflagged
dependency is the failure this stage exists to prevent.

The description should say: "Stage 2 of the Dev flow. Takes the plan from the worklog
and the real codebase and produces a chosen approach plus an impact map, answering
'what will this touch, and what breaks?' Use after Receive & Plan and before Build."
```

### Step 3: Review the generated skill

Claude Code writes `.claude/skills/design/SKILL.md`. Open it and check it includes:

- [ ] YAML frontmatter with `name: design` and a clear description
- [ ] A workflow: read worklog §1 → read the real code → trace callers → choose approach → append §2
- [ ] The five-part impact map (approach, change set, blast radius, tests in play, risks)
- [ ] "Read before you decide — trace the actual callers" guideline
- [ ] "Design, don't build — no production code" guideline
- [ ] "Honour existing conventions / integer pence" guideline

If anything's missing, ask Claude to add it. Reviewing the generated skill *is* part of the skill you're building in yourself.

### Step 4: Test it

Run the skill on your carry-through ticket's worklog:

```
Use the design skill on <TICKET-KEY>-worklog.md
```

### Step 5: Verify the output

- [ ] It read the `## 1 · Receive & Plan` section as its input
- [ ] It actually opened the real code (names real functions/exports, not guesses)
- [ ] It traced callers/dependents — the blast radius names real files
- [ ] It chose a concrete approach (and named alternatives if there were real ones)
- [ ] It appended a `## 2 · Design` section to the worklog, below section 1, leaving section 1 untouched
- [ ] No production code was written

### Exercise: pressure-test the design

Practise the review loop with follow-ups:

- *"Which callers of the code we're changing did you check — show me the grep."*
- *"What's the smallest possible version of this change?"*
- *"Which existing tests could newly fail under this approach, and why?"*
- *"Is there an alternative approach? Why is this one better?"*

---

## Part 3: Build the `design` Agent (~30 min)

### What this agent does

The skill is the *how-to*. The agent is the *operator* that runs the whole job hands-off: point it at the ticket's worklog and walk away with an approach + impact map appended, ready for your review — no step-by-step prompting.

### Step 1: Create the agent

In your Claude Code session:

```
/agents
```

Choose **Create new agent** (this scaffolds a file in `.claude/agents/`).

### Step 2: Paste this prompt

```
Create a subagent called "design".

Purpose: given the worklog's plan and the real codebase, produce a chosen approach
plus an impact map, then stop. It writes NO production code — its only write is the
"## 2 · Design" section it appends to the worklog.

It should:
- Invoke the design skill.
- Read the touch points for real, trace every caller/dependent of the code that will
  change, choose the smallest safe approach, and produce the impact map.
- Append "## 2 · Design" to <TICKET-KEY>-worklog.md beneath the plan (never editing
  sections above it).
- Return the approach + impact map in one clean summary, then pause for the developer
  to confirm or adjust before Build.

Tools it needs: file read/search (Read, Glob, Grep), Bash, and Edit (only to append
the worklog). No production-code editing.
Description: "Reads the worklog plan and the codebase and produces a chosen approach +
impact map. Use after Receive & Plan, before Build."
```

### Step 3: Review the generated agent

Open `.claude/agents/design.md` and check:

- [ ] `name: design` and a clear `description`
- [ ] Tools include file read/search and Bash/Edit (to append the worklog) — but no broad code-writing brief
- [ ] It invokes the `design` skill
- [ ] It writes no production code — its only write is the `## 2 · Design` worklog section
- [ ] It returns the approach + impact map and pauses for confirmation

### Step 4: Run it end-to-end

```
Use the design agent on <TICKET-KEY>
```

### Step 5: Verify

- [ ] The agent ran the full job from the worklog
- [ ] The approach is concrete and the blast radius names real callers
- [ ] It appended `## 2 · Design` to the worklog beneath section 1
- [ ] It stopped and waited — it didn't start building

---

## Part 4: Hand off to Day 3 (~15 min)

Day 3 (Build) reads your `## 2 · Design` section and turns it into code. Lock it in:

- Run the agent on your **carry-through ticket** so the worklog now has sections 1 **and** 2.
- Skim the `## 2 · Design` section yourself — is the approach the smallest safe change? Are the risks real? This is the last cheap moment to change direction before code exists.

That design *is* Build's input tomorrow.

---

## Day 2 Recap

### What you built

- `design` **skill** — worklog plan + real code → traced blast radius → approach + impact map appended as `## 2 · Design`
- `design` **agent** — runs that job hands-off, writes no production code, pauses before building

### The dev chain (you're at step 2)

```
Receive & Plan → Design → Build → Push → Rework
                 (today)
```

### Key concepts

- **Plan scopes the work; Design scopes the risk.** Plan stayed at altitude; Design reads the code for real.
- **Trace, don't guess.** The blast radius comes from grepping actual callers — hidden coupling is the bug this stage prevents.
- **The worklog is the thread.** Design reads `## 1` and appends `## 2`; it never edits sections above its own.
- **Design, don't build.** The output is the plan Build executes — no production code today.
- **Smallest safe change wins.** The approach must satisfy the contract without breaking callers.

### Checkpoint

Before moving to Day 3, confirm:

- [ ] `design` skill saved in `.claude/skills/`
- [ ] `design` agent saved in `.claude/agents/`
- [ ] You ran the agent on your carry-through ticket and got an approach + impact map
- [ ] The worklog now has `## 1 · Receive & Plan` **and** `## 2 · Design`
- [ ] You understand: **Design reads the code, traces the blast radius, and commits to an approach — before any code**

*Scoped the risk. Chose the approach. Kept it in the worklog. → Day 3: Build.*
