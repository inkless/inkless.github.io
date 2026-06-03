+++
title = "I Stopped Babysitting My Coding Agent and Built a Fleet Instead"
author = ["Guangda Zhang"]
date = 2026-06-03T09:00:00-07:00
lastmod = 2026-06-03T09:00:00-07:00
categories = ["engineering"]
tags = ["ai", "claude-code", "agents", "workflow", "tooling"]
draft = true
+++

*How I run 10–20 Claude Code agents in parallel — and the two pieces of plumbing that made it possible.*

A year ago my AI coding setup was one terminal, one agent, one task. I'd type a prompt, watch it work, approve a tool call, watch some more. It was genuinely useful and genuinely a leash. The agent was fast; I was the bottleneck. Every permission prompt, every "what should I do next," every context-gathering pause routed through me.

So I did the obvious thing and opened a second terminal. Then a third. Within a week I had a tmux grid of agents and a new, worse problem: I had no idea what any of them were doing. Which one was blocked on a prompt? Which one finished an hour ago and was idle? Which one quietly went down a wrong path while I was reading another pane?

The interesting realization was that **"can the agent write the code" had stopped being the constraint.** The model is good enough. The constraints were now two scarce resources that belong to *me*:

1. **Context** — every agent starts amnesiac. The knowledge of *how we do things here* lived in my head, and I was re-typing it into every session.
2. **Attention** — one human can only look at one pane at a time. N agents generate N streams of "I need you," and without a way to triage them, parallelism just converts into anxiety.

This post is about the two systems I built to buy back those resources: a **memory bank** for context, and a **tracker + a TUI called triage** for attention. Together they turn "a bunch of terminals" into something that feels more like running a small team.

## Part 1: Context — giving agents a memory that outlives the session

The first thing I built wasn't fancy. It's a git repo full of markdown.

I call it the **memory bank**. It holds RFCs, investigation notes, per-project analysis, and a pile of small "things I learned the hard way" files. Two files at the root do the heavy lifting:

- `index.md` — a categorized table of contents.
- `log.md` — an append-only changelog, newest first.

Every write to the bank updates both. That sounds like bookkeeping overhead, and it is — so I don't do it by hand. A slash command (`/memory-bank`) handles search/read/write from *any* repo I'm working in, and a small script enforces the log format. There's even a hook that **blocks** direct edits to `log.md` and redirects me to that script, because the one thing worse than no log is a log in five inconsistent formats.

Why does this matter for a fleet? Because the memory bank is the shared brain. When I spin up an agent on some project, its prompt points at the relevant memory-bank docs, and it starts the session already knowing the decisions, the gotchas, and the half-finished threads. I'm not re-explaining the architecture or that one browser bug for the fortieth time.

On top of the project docs, I lean on two lighter forms of memory:

- **Persistent per-fact memory.** A directory of tiny markdown files, each holding *one* fact with a little frontmatter — who I am, preferences, corrections I've given before, pointers to dashboards. These load at the start of every session. So when I tell an agent "stop prescribing a phased rollout plan in your audit reports," that correction becomes a file, and I never have to give it twice.
- **Skills.** Reusable, parameterized procedures the agent can invoke by name — "open a PR with our formatting," "monitor this PR's CI and review loop," "run the test-and-fix cycle," "sync these docs to our wiki." A skill is basically a memory of *how to do a recurring task well*, version-controlled and shared across every agent.

The throughline: **anything I'd otherwise have to say twice becomes a file.** Project knowledge → memory-bank docs. Personal corrections → memory files. Repeatable procedures → skills. Each one is a small deposit that every future agent withdraws from for free.

And it's not a one-way read — the agents write to memory too. Each task gets its own working doc, and when an agent is about to run low on context (or hand off to a fresh one), it checkpoints itself into that doc: what's done, what's still open, what bit it along the way. So a task that spans nine hours and three separate agent sessions doesn't start over each time — the next agent reads the handoff and resumes mid-stream. The agents maintain the memory as much as I do.

(A side benefit of "everything is markdown in a git repo": I can *read* it like a website. I wrote a little CLI — [`mdbrowse`](https://github.com/inkless/mdbrowse) — that serves a folder of markdown as a GitHub-styled site with sidebar nav, live reload, and `Cmd+K` search, no GitHub round-trip. The memory bank is both an agent's context source and my personal wiki, same files.)

This is the unglamorous half of the system, and it's the half that actually unlocks parallelism. You cannot run ten agents if each one needs you for a twenty-minute context download.

## Part 2: Attention — a tracker as the fleet's shared dashboard

Once agents could start smart, I could run more of them. Which brought back the "what is everyone doing" problem, at scale.

The fix was to give the fleet a **shared, structured source of truth** about work in flight. Each project gets a tiny tracker — really just append-only event logs (`claim`, `note`, `review`, `merge`, `block`, …) plus a rendered `tracker.md`. Agents never edit these by hand; they go through a single mutator script (`track.py`) that serializes writes with a lock and re-renders the views atomically. (Direct edits are, again, hook-blocked. I have a pattern.)

The payoff is a single auto-generated **cross-project overview** that re-renders on every mutation. A slice of mine on a normal day:

```text
🙋 Needs your attention   — questions an agent raised for me
🟡 In progress (4)        — agent, branch, PR #, age, last note
🟠 In review (8)          — PR, sign-off + review state, age, last note
⚪ Available  (top 2/proj) — next tasks, with dependencies
🟢 Recently merged (24h)  — 15
```

That table is the closest thing the fleet has to a manager. Each in-progress row shows *which* agent owns it, on what branch, with what PR, and — critically — the agent's own last one-line note ("rebased, all CI green, only the change-management sign-off remains"). I can glance at it and know the state of a dozen workstreams in five seconds, without entering a single pane.

Two parts of that board do quiet work for me. The *needs-your-attention* section at the very top is reserved for the one thing I can't derive from anywhere else: a genuine question an agent has raised ("this migration has two valid approaches — which do you want?"). It's usually empty — and that's the point; when something shows up there, it's real, and it sorts above everything. Separately, over in *in review*, a background job checks each open PR's actual state every few minutes — is a sign-off still waiting on me, has anyone even been asked to review yet — and surfaces that right on the row, floating the PRs blocked on *me* to the top. So the board doesn't just show what the agents are doing; it tells me precisely what's waiting on me. And the agents write to it as much as I read from it: when I answer one of those questions, the agent resumes from where it parked, in its own words.

A few rules keep the signal honest, and they were all written after I got burned:

- **One row per agent until it's reviewed.** Splitting one agent across tasks just multiplies mistakes; parallelism comes from *more agents*, not overloaded ones.
- **Don't mark a row "in review" until it's genuinely mergeable** — green CI, every bot comment addressed, gates actually run locally. Premature "ready!" flags train me to ignore the column, and then I miss the real ones.
- **Attach the PR link the moment it opens,** not when it's done, so I can follow along while the agent is still cooking.

And because spinning up a fresh agent on a task is itself repetitive, there's a launcher (`fleet-launch.sh`) that does the whole dance per task: create an isolated **git worktree** (via another small tool, `gw`, that parks worktrees under `~/.worktrees/` so each agent gets a clean checkout and they don't trample each other's `node_modules`/build caches), claim the tracker row, render the prompt from a shared template (which references my global agent-rules — rebase don't merge, never force-push someone else's branch, never skip pre-commit hooks), and open a tmux window running the agent. One command, N agents, each landing in its own sandbox already briefed.

And because everything an agent does is just an event in a log, that data composes *upward* as easily as it renders sideways. For a big cross-team initiative I render a second view from the same logs: not the operational "who's doing what right now," but a stakeholder-facing progress rollup — every task across several projects collapsed into milestones and themes, with a hand-written paragraph of context around the auto-generated tables. It publishes to the wiki the non-engineers actually read, and it regenerates itself from the work the agents are already recording. Same source of truth, two audiences: the operational board for me, the milestone view for everyone else.

## Part 3: triage — knowing which pane needs me *next*

The overview tells me the state of the *work*. It doesn't tell me, right now, which of the fifteen live *sessions* is sitting on a permission prompt versus churning happily versus quietly stuck.

So I wrote a little Rust TUI called **triage**. It joins three sources — the session JSON, the transcript logs, and `tmux list-panes` — into one table, one row per live agent, sorted by *how badly it needs me*. An eight-state attention model (working, idle-short, blocked, …) drives the sort. It's not Claude-specific, either — it tags each row by provider, so Claude Code and Codex sessions show up side by side in the same view. Each row's headline is the agent's own natural-language recap of what it's doing, so I'm reading English, not guessing from a spinner. Hit a key, jump straight to that pane.

The part that changed my life is **autonomous mode**. For sessions that are merely blocked on a routine permission prompt — "can I run this test command?" — a smaller, cheaper model evaluates the request and returns APPROVE / DENY / WAIT, routing the verdict back through the same approval path I'd use by hand. The routine "yes, run the tests" prompts get handled without me. The genuinely novel or risky ones still escalate. I get a desktop notification — and optionally a phone push (self-hosted ntfy) for when my laptop's locked — when something actually needs a human.

That's the difference between *watching* a fleet and *supervising* one. I'm not staring at panes anymore; I'm responding to a prioritized queue, and the bottom of that queue handles itself.

The agents can also talk to *each other* through triage — short coordination messages like "I'm about to edit this file, are you?" or "I just merged the PR you depend on, rebase before your next push." When you've got several agents in the same monorepo, a little peer coordination prevents a lot of merge-conflict rework.

## Part 4: The manager is also an agent

For a while I was the manager — reading the overview, deciding what to launch next, poking the stuck ones, checking whether "done" was really done. But "read the dashboard and delegate" is itself a job, and it turns out an agent can do that job too.

So now I sometimes run a **coordinator**: an agent in manager mode whose entire purpose is to oversee the others. It never writes code or claims a task itself. It reads the tracker and the live triage state, **ground-truths** what the workers report (an agent saying "CI is green" and CI *actually* being green are different things — so there's a reconcile pass that sweeps every in-review task against the real merge state on GitHub and closes the ones that actually landed, instead of trusting the notes), launches new agents onto available work as capacity frees up, nudges the ones that are stuck or drifting, relays coordination between agents whose changes overlap, and hands me a single digest instead of fifteen panes.

That's the part that still surprises me. The leverage didn't come from a smarter *worker* — it came from separating the **roles**. Workers execute one task each, in their own sandbox. A coordinator holds the cross-fleet picture and delegates. I sit above the coordinator, handling the genuine judgment calls it escalates. It's an org chart, and I'm only at the top of it.

## What it actually feels like now

A normal morning: I open the overview, see four agents mid-task and eight PRs in review. I skim the last-notes. One agent flagged a real design question overnight — that one gets me. The rest either merged on green CI (auto-approved by the auditor) or are waiting on a human sign-off I can click through. I queue up the next batch of available tasks with one launcher command, and go do something else.

The mental model that got me here:

> **Treat your own attention and context as the scarce resources, not the model's capability. Then build the boring infrastructure — memory, a tracker, a triage queue — that lets one human spend those resources across many agents instead of one.**

None of the individual pieces are exotic. Markdown files, append-only logs, git worktrees, tmux, a hook here and there, one TUI. The leverage isn't in any single tool — it's in the fact that, together, they let me stop being the thing my agents wait on.

These days I mostly architect — decide what to build, how the pieces fit, which approach to take — and almost never write the code myself. The actual typing, the chasing-CI, the rebasing, the "did that PR ever land" — that's a fleet's job now, and I just keep the dashboard honest.

---

*Some of it is open source — the triage TUI lives at [github.com/inkless/triage](https://github.com/inkless/triage), and the markdown browser at [github.com/inkless/mdbrowse](https://github.com/inkless/mdbrowse). If you're drowning in agent terminals, start with the cheapest piece: write down the thing you keep re-typing.*
