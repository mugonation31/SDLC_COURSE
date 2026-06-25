# Day 1: Receive & Plan

## Overview

Today you'll set up your Claude Code environment, connect it to Jira (so the contract comes to you, not the other way round), and build your first skill **and** its agent. By the end of the day, you'll be able to point Claude Code at a ticket key and get back a scoped, codebase-aware task plan — every task traced to an acceptance test, ambiguities flagged before you write a line of code.

**What you'll build:**

- `receive-and-plan` **skill** — pulls a Jira ticket's acceptance criteria + the SDET's tests via MCP and turns them into a scoped, traceable task list
- `receive-and-plan` **agent** — wields that skill as a hands-off worker: ticket key in → plan out

**Time estimate:** ~2 hours

> **Why Claude Code (not the web)?** You're a developer — your skills and agents live in your repo (`.claude/`), version-controlled alongside the code they act on, and Jira reaches you through MCP. (PMs use the claude.ai web version because they don't touch code; we do.)

---

## Part 1: Environment Setup (~30 min)

### Step 1: Sign into Claude

1. Go to claude.ai
2. Sign in with your Claude Pro, Max, Team, or Enterprise account
3. If you don't have an account, sign up for Claude Pro

### Step 2: Set up the Claude Code CLI

1. Install Claude Code:

   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

2. Start it inside your project repo:

   ```bash
   cd your-project
   claude
   ```

3. Sign in by running `/login` in the session, then complete the browser flow with the same Claude account from Step 1.

### Step 3: Connect Jira via MCP

This is how the contract (the PM's AC + the SDET's tests) reaches you without copy-paste — you register Atlassian's MCP server once, then log in through your browser.

1. If you're inside a Claude Code session, exit first (`Ctrl+C` twice, or type `exit`). Then, in your terminal, register the Atlassian server:

   ```bash
   claude mcp add --transport http atlassian https://mcp.atlassian.com/v1/mcp/authv2
   ```

   Claude Code confirms with `Added HTTP MCP server "atlassian"`. Nothing is connected yet — you've just told Claude Code the server exists.

2. Start Claude Code again and open the MCP menu:

   ```bash
   claude
   ```
   ```
   /mcp
   ```

   You'll see **atlassian** listed with a status like **"needs authentication."**

3. Select **atlassian**, then choose **Authenticate**. Your browser opens an Atlassian login page.

4. Log into your Atlassian account, pick your Jira site if asked, and click **Accept**. The page confirms and tells you to return to your terminal.

5. Back in Claude Code, run `/mcp` again. **atlassian** now shows as **connected ✓**.

> This is Atlassian's current official endpoint. You may see older guides use a `.../v1/sse` address with `--transport sse` — that one is being retired; the `http` command above is the right one.

This lets Claude Code read and write Jira tickets *using your permissions* — it only sees what you can see. No elevated access.

### Step 4: Verify your connection

In the Claude Code session, type:

```
List the Jira projects I have access to
```

If Claude returns your project list, you're good to go.

- [ ] Signed into Claude (Step 1)
- [ ] Claude Code installed, running, and authenticated via `/login`
- [ ] Atlassian MCP connected
- [ ] `List the Jira projects I have access to` returns your projects

---

## Part 2: Build the `receive-and-plan` Skill (~50 min)

### What this skill does

Today, picking up a ticket means 30–60 minutes of cold ramp-up: reading the AC, opening the SDET's tests, hunting through unfamiliar code to see what's affected, then breaking it into tasks. This skill collapses that into one move — **ticket key in → scoped, sourced plan out** — with every task traced back to an acceptance test.

This is **step 1 of the dev chain**: Plan → Design → Build → Push → Rework.

### Step 1: Open the skill creator

In your Claude Code session, type:

```
/skill-creator
```

(Or simply: *"Create a new skill for me"* — Claude Code will scaffold the files in `.claude/skills/`.)

### Step 2: Paste this prompt

Copy and paste the following exactly:

```
Create a skill called "receive-and-plan" that pulls a Jira ticket's acceptance
criteria and the SDET's attached acceptance tests, then turns them into a scoped,
codebase-aware task list.

The skill should:
- Accept a Jira ticket key (e.g. PROJ-1234). Ask for one if not provided.
- Use the Jira MCP to fetch the ticket: summary, acceptance criteria, and the
  SDET's attached/linked acceptance tests (the contract).
- Read the acceptance tests as the contract and list every behaviour they assert
  ("when X, then Y").
- Lightly locate the touch points — search the repo just enough to name the file(s)/module(s)
  the work will land in. Do NOT trace the full blast radius; that's Day 2 (Design).
- Move the ticket To Do → In Progress via the Jira MCP as you pick it up. Match the transition
  by its status NAME ("In Progress"), never a hard-coded ID — cloned boards reassign IDs.
- Output:
  1. Task list — small, ordered tasks, each tied to ONE acceptance behaviour.
  2. Scope — what's in scope and what's explicitly out of scope.
  3. Done-gate — the acceptance criteria restated as the definition of done.
  4. Open questions — any acceptance criterion that is ambiguous or untestable,
     flagged for the SDET/PM (do not guess).
- Save the plan into a worklog file at the repo root named `<TICKET-KEY>-worklog.md`
  (derive the key from the ticket, e.g. PROJ-1234-worklog.md; create the file). Write it
  under a heading `## 1 · Receive & Plan`. This worklog is the handoff every later stage
  (Design, Build, Push, Rework) reads and appends its own section to.
- Do NOT write code. Planning only (the worklog is notes, not code).

Guidelines: Every task must trace to a specific acceptance test — no orphan tasks.
Keep tasks small enough to complete and verify individually. If the tests and the
AC text disagree, treat the tests as the source of truth and flag the mismatch.

The description should say: "Pulls a Jira ticket's acceptance criteria and the SDET's
acceptance tests via MCP and turns them into a scoped, codebase-aware task list. Use at
the start of every ticket, when planning work, or when breaking a story into tasks."
```

### Step 3: Review the generated skill

Claude Code writes `.claude/skills/receive-and-plan/SKILL.md`. Open it and check it includes:

- [ ] YAML frontmatter with `name: receive-and-plan` and a clear description
- [ ] A workflow: fetch ticket → move it To Do → In Progress → read tests → scope touch points → output plan
- [ ] The outputs (task list, scope in/out, done-gate, open questions)
- [ ] It writes the plan to `<TICKET-KEY>-worklog.md` under `## 1 · Receive & Plan`
- [ ] "Tasks trace to acceptance tests" guideline
- [ ] "Tests are source of truth over AC text" guideline
- [ ] "Planning only — no code" guideline

If anything's missing, ask Claude to add it. Evaluating the output *is* the skill you're building in yourself.

### Step 4: Test it

Pick a real ticket from your project and run:

```
Use the receive-and-plan skill on PROJ-1234
```

### Step 5: Verify the output

- [ ] It fetched the real ticket via MCP (AC + tests)
- [ ] The ticket moved To Do → In Progress on the board on its own (via MCP)
- [ ] Each task traces back to a specific acceptance behaviour
- [ ] It named the real touch-point file(s) in your repo (light scope — not a full impact map; that's Day 2)
- [ ] Genuinely ambiguous/untestable ACs are flagged as open questions
- [ ] No code was written

### Exercise: tighten the plan

Practice the review loop with follow-ups:

- *"Re-plan focusing only on the error-path / negative tests."*
- *"Which acceptance tests have no matching task yet?"*
- *"Group the tasks by the file they touch."*
- *"Flag any acceptance criterion that has no corresponding test."*

---

## Part 3: Build the `receive-and-plan` Agent (~30 min)

### What this agent does

The skill is the *how-to*. The agent is the *operator* that runs the whole job hands-off, so you can hand it a ticket key and walk away with a plan — no step-by-step prompting.

### Step 1: Create the agent

In your Claude Code session:

```
/agents
```

Choose **Create new agent** (this scaffolds a file in `.claude/agents/`).

### Step 2: Paste this prompt

```
Create a subagent called "receive-and-plan".

Purpose: given a Jira ticket key, produce a scoped, codebase-aware task plan and
stop. It writes NO source code — planning only — though it does create the worklog
and move the ticket's board status.

It should:
- Take a ticket key as input.
- Invoke the receive-and-plan skill.
- Return the task list, scope, done-gate, and open questions in one clean summary.
- Pause for the developer to confirm or adjust before any building begins.

Tools it needs: file read/search (Read, Glob, Grep), Write (for the worklog), and the
Jira MCP tools (read the ticket + move it To Do → In Progress).
Description: "Reads a Jira ticket via MCP and produces a scoped, codebase-aware task
plan. Use at the start of every ticket."
```

### Step 3: Review the generated agent

Open `.claude/agents/receive-and-plan.md` and check:

- [ ] `name: receive-and-plan` and a clear `description`
- [ ] Tools include file read/search, **Write** (for the worklog), **and** the Jira MCP
- [ ] It invokes the `receive-and-plan` skill
- [ ] It writes no source code (planning only) — but does create the worklog and move the board
- [ ] It returns the outputs (task list, scope, done-gate, open questions) and pauses for confirmation

### Step 4: Run it end-to-end

```
Use the receive-and-plan agent to plan PROJ-1234
```

### Step 5: Verify

- [ ] The agent ran the full job from just a ticket key
- [ ] Output matches the skill's outputs (task list, scope, done-gate, open questions)
- [ ] The ticket moved To Do → In Progress, and the worklog was created
- [ ] It stopped and waited — it didn't start building

---

## Part 4: Lock in your plan for Day 2 (~10 min)

Day 2 (Design) builds directly on today's plan — and it reads it from the **worklog** your skill just created.

- Run the agent on the **real ticket you'll carry through the whole course** (pick one now and stick with it).
- Confirm the skill wrote `<TICKET-KEY>-worklog.md` at the repo root with a `## 1 · Receive & Plan` section. That file **is** the handoff — Day 2 appends `## 2 · Design` beneath it, and every later stage reads it.
- The worklog is a working scratchpad, not a deliverable — add `*-worklog.md` to your `.gitignore` so it isn't committed.

This gives Day 2 real input to design against.

---

## Day 1 Recap

### What you built

- `receive-and-plan` **skill** — ticket key → AC + SDET tests (via MCP) → scoped, traceable task plan
- `receive-and-plan` **agent** — runs that job hands-off, writes no code, pauses before building

### The dev chain (you're at step 1)

```
Receive & Plan → Design → Build → Push → Rework
   (today)
```

### Key concepts

- **Skills are the *how-to*; agents are the *operator* that runs them.** Same name, paired.
- **MCP is the pipe to Jira** — the contract comes to you; you don't copy-paste it.
- **The worklog is the handoff.** Receive & Plan creates `<TICKET-KEY>-worklog.md` and writes `## 1 · Receive & Plan`; every later stage reads it and appends its own section.
- **The acceptance tests are the contract.** Every task traces to one; the tests win over prose AC.
- **Plan before you build.** This agent writes no code today — it only plans (its only writes are the worklog and the board move).
- **Always review AI output before acting on it** — reviewing the generated skill/agent is part of the skill.

### Checkpoint

Before moving to Day 2, confirm:

- [ ] Claude Code installed, authenticated, Jira MCP connected
- [ ] `receive-and-plan` skill saved in `.claude/skills/`
- [ ] `receive-and-plan` agent saved in `.claude/agents/`
- [ ] You ran the agent on your chosen ticket and got a traceable plan
- [ ] The plan is saved in `<TICKET-KEY>-worklog.md` (`## 1 · Receive & Plan`) for Day 2
- [ ] You understand: **skill = how-to, agent = operator, MCP = the Jira pipe**

*Learned something. Built something. Kept it. → Day 2: Design.*
