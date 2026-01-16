# Session 003

## Problem Statement Focus

This session focused on understanding the core problems Malamar solves, to build a solid foundation for later discussions about features, specs, and technical design.

## Current Workflow Pain Points

Q: Before Malamar, what does your typical workflow look like when using multiple AI CLI tools?

A: Two examples:

**Example 1 - Implementing code:**
1. Spin up Claude Code, instruct it to discover ideas and write a plan file
2. Quit Claude Code, spin up Gemini CLI (Gemini 3 Flash - more context, cheaper) to implement the plan
3. Quit Gemini CLI, spin up Codex CLI to review Git changes and write a feedback file
4. Quit Codex CLI, spin up Claude Code to verify feedback and update the plan
5. Loop until result is good enough

**Example 2 - Writing blog posts:**
1. Spin up Gemini CLI, share idea, instruct it to learn from related posts and write a draft
2. Quit Gemini CLI, spin up Codex CLI for proofreading, spellcheck, and web fact-checking
3. Quit Codex CLI, spin up Claude Code to convert draft to final post with Mermaid/D3.js diagrams

The manual "quit, spin up, instruct, repeat" cycle is time-consuming and cannot be automated.

---

Q: What's the most frustrating part of this manual workflow?

A: All of these:
- Time spent switching between tools
- Need to manually write instructions each time
- Context loss between sessions

---

## Why Multiple AI Tools

Q: Why use different AI tools for different steps instead of just one?

A:
1. **Claude Code** - Powerful, but low usage limits and expensive
2. **Gemini CLI** - Cheaper, more context length, better writing skills, good multilingual support. But weaker at following instructions.
3. **Codex CLI** - Good for long working sessions, but a bit naive
4. **OpenCode** - Enables local LLM via Ollama for compliance projects or when handling customer PII that shouldn't be sent to external services

---

## The "Checking Each Other's Work" Benefit

Q: You mentioned wanting tools to "check each other's work." Is this about catching mistakes, diverse perspectives, or covering weaknesses?

A: All of them, but the most important things are:
- Automatically catching mistakes at each step
- Separating concerns
- Reducing long-context hallucination

Importantly, Malamar can work with just one CLI (e.g., Claude Code only). The multi-agent approach (Planner, Implementer, Reviewer, Approver) is valuable even with a single CLI because shorter, focused sessions reduce hallucination.

---

Q: Can you give an example of "long-context hallucination"?

A: When an AI tool makes a mistake late in a session that it wouldn't have made if the session was fresh and focused. Fresh, focused sessions per step help keep the AI grounded.

---

## Recovery and Human Intervention

Q: What happens when something goes wrong mid-way?

A: This is somewhat "YOLO" - must trust the AI enough to delegate. However:
- The "In Review" status allows human involvement via comments
- Email notifications alert when a task moves to "In Review"
- Can steer agents from phone via Tailscale, away from the working laptop

---

Q: If agents get stuck in a loop disagreeing with each other, how do you handle it?

A: Let them fight until usage limits run out. But with Malamar, can intervene by:
1. Manually moving task to "In Review"
2. Commenting with feedback (e.g., "reviewer, go easy on this one")
3. Moving task back to "In Progress" to continue with adjusted behavior

The comment becomes instruction that influences the next pass.

---

## What "Good Enough" Means

Q: When you come back to a completed task, what does "good enough" look like?

A: Totally depends on agent instructions - Malamar should not care about this. Examples:
- **Critical projects**: Approver commits to a PR, notifies human to review
- **Non-critical projects** (personal blog): Approver pushes directly to main, triggering automatic build and publish

The definition of "done" lives entirely in agent instructions.

---

## Task Parallelism and Workspaces

Q: How many tasks do you typically work on in parallel?

A: Many projects in parallel, but one thing at a time per project. This is why workspaces exist - to focus on one concern that agents help with.

Key points:
- Multiple workspaces can run in parallel
- Within a workspace, only one task is processed at a time
- Task pickup priority is NOT FIFO - it prioritizes the task just processed, trying to complete it before moving to another ("tackle one task until it's In Review, not starting everything but nothing is done")

---

Q: What does a workspace represent - is it a repository or project?

A: A workspace is "one thing I need agents to help me working on" - not necessarily a repository. Example with a Docusaurus blog:
1. A workspace for blog post writing
2. A workspace for UI implementation
3. A workspace for MDX component creation
4. A workspace for updating dependencies across all repositories

Same repo can have multiple workspaces, each with agents tuned for that type of work.

---

Q: Would agent instructions differ significantly between these workspaces?

A: Yes. For example, the Reviewer in the blog writing workspace would have different instructions than the Reviewer in the UI implementation workspace. But again, this is not Malamar's responsibility - just user-defined content.

---

## Target Users

Q: Do your colleagues who might use Malamar also use multiple AI CLIs?

A: Yes, most of them use multiple AI CLIs. The multi-CLI orchestration is core to the problem for the target user base.

---

## Failure Modes

Q: What would make you stop using Malamar after a week?

A: Have been mentally simulating this idea for 2 weeks while working with CLIs - no pitfalls seen. The only concern is AI cost burning faster, but that's acceptable because:
- Already burning cost manually
- Malamar brings more value, autonomy, and reliability
- It's a known tradeoff

---

Q: What about cascading failures - an agent hallucinates, next agent builds on the mistake?

A: 100% depends on agent instructions - not Malamar's problem. The next agent must be designed to verify the work of agents before it. If the user fails to write proper instructions for this, Malamar can only say sorry. Malamar provides loop mechanics; quality depends on instruction design.

---

## Core Requirements

Q: What's the minimum viable version of Malamar?

A: Must have:
- Ability to manage workspaces, agents, and tasks
- The core loop of the runner

But should focus on possibilities now, not trimming features.

---

## The One-Liner

Q: How would you summarize the core problem Malamar solves?

A: **"Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity."**
