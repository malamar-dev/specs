# Session 003

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Current Workflow Pain Points

Before Malamar, what does the typical workflow look like when using multiple AI CLI tools?

### Answer

Two examples illustrate the current manual workflow:

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

The manual "quit, spin up, instruct, repeat" cycle is time-consuming and cannot be automated without a tool like Malamar.

## Question #2: Most Frustrating Part

What's the most frustrating part of this manual workflow?

### Answer

All of these are equally frustrating:
- Time spent switching between tools
- Need to manually write instructions each time
- Context loss between sessions

## Question #3: Why Multiple AI Tools

Why use different AI tools for different steps instead of just one?

### Answer

Each tool has different strengths and weaknesses:

1. **Claude Code** - Powerful, but low usage limits and expensive
2. **Gemini CLI** - Cheaper, more context length, better writing skills, good multilingual support. But weaker at following instructions.
3. **Codex CLI** - Good for long working sessions, but a bit naive
4. **OpenCode** - Enables local LLM via Ollama for compliance projects or when handling customer PII that shouldn't be sent to external services

## Question #4: "Checking Each Other's Work" Benefit

You mentioned wanting tools to "check each other's work." Is this about catching mistakes, diverse perspectives, or covering weaknesses?

### Answer

All of them, but the most important things are:
- Automatically catching mistakes at each step
- Separating concerns
- Reducing long-context hallucination

Importantly, Malamar can work with just one CLI (e.g., Claude Code only). The multi-agent approach (Planner, Implementer, Reviewer, Approver) is valuable even with a single CLI because shorter, focused sessions reduce hallucination.

## Question #5: Long-Context Hallucination

Can you give an example of "long-context hallucination"?

### Answer

Long-context hallucination occurs when an AI tool makes a mistake late in a session that it wouldn't have made if the session was fresh and focused. As conversations grow longer, the AI can lose track of earlier context or make assumptions that compound into errors. Fresh, focused sessions per step help keep the AI grounded.

## Question #6: Recovery and Human Intervention

What happens when something goes wrong mid-way?

### Answer

This is somewhat "YOLO" - users must trust the AI enough to delegate. However, there are intervention mechanisms:
- The "In Review" status allows human involvement via comments
- Email notifications alert when a task moves to "In Review"
- Users can steer agents from phone via Tailscale, away from the working laptop

## Question #7: Agent Disagreement Loops

If agents get stuck in a loop disagreeing with each other, how do you handle it?

### Answer

Let them fight until usage limits run out. But with Malamar, users can intervene by:
1. Manually moving the task to "In Review"
2. Commenting with feedback (e.g., "reviewer, go easy on this one")
3. Moving the task back to "In Progress" to continue with adjusted behavior

The comment becomes an instruction that influences the next pass.

## Question #8: Definition of "Good Enough"

When you come back to a completed task, what does "good enough" look like?

### Answer

Totally depends on agent instructions - Malamar should not care about this. Examples:
- **Critical projects**: Approver commits to a PR, notifies human to review
- **Non-critical projects** (personal blog): Approver pushes directly to main, triggering automatic build and publish

The definition of "done" lives entirely in agent instructions.

## Question #9: Task Parallelism and Workspaces

How many tasks do you typically work on in parallel?

### Answer

Many projects in parallel, but one thing at a time per project. This is why workspaces exist - to focus on one concern that agents help with.

Key points:
- Multiple workspaces can run in parallel
- Within a workspace, only one task is processed at a time
- Task pickup priority is NOT FIFO - it prioritizes the task just processed, trying to complete it before moving to another ("tackle one task until it's In Review, not starting everything but nothing is done")

## Question #10: Workspace Definition

What does a workspace represent - is it a repository or project?

### Answer

A workspace is "one thing I need agents to help me working on" - not necessarily a repository. Example with a Docusaurus blog:
1. A workspace for blog post writing
2. A workspace for UI implementation
3. A workspace for MDX component creation
4. A workspace for updating dependencies across all repositories

The same repo can have multiple workspaces, each with agents tuned for that type of work.

## Question #11: Agent Instructions Per Workspace

Would agent instructions differ significantly between these workspaces?

### Answer

Yes. For example, the Reviewer in the blog writing workspace would have different instructions than the Reviewer in the UI implementation workspace. But again, this is not Malamar's responsibility - just user-defined content.

## Question #12: Target Users

Do your colleagues who might use Malamar also use multiple AI CLIs?

### Answer

Yes, most of them use multiple AI CLIs. The multi-CLI orchestration is core to the problem for the target user base.

## Question #13: Failure Modes

What would make you stop using Malamar after a week?

### Answer

Have been mentally simulating this idea for 2 weeks while working with CLIs - no pitfalls seen. The only concern is AI cost burning faster, but that's acceptable because:
- Already burning cost manually
- Malamar brings more value, autonomy, and reliability
- It's a known tradeoff

## Question #14: Cascading Failures

What about cascading failures - an agent hallucinates, next agent builds on the mistake?

### Answer

100% depends on agent instructions - not Malamar's problem. The next agent must be designed to verify the work of agents before it. If the user fails to write proper instructions for this, Malamar can only say sorry. Malamar provides loop mechanics; quality depends on instruction design.

## Question #15: Minimum Viable Version

What's the minimum viable version of Malamar?

### Answer

Must have:
- Ability to manage workspaces, agents, and tasks
- The core loop of the runner

But the focus should be on possibilities now, not trimming features.

## Question #16: The One-Liner

How would you summarize the core problem Malamar solves?

### Answer

**"Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity."**
