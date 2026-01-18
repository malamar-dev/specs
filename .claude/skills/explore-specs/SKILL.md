---
name: explore-specs
description: A dedicated workflow for Irving Dinh and Claude Code to clarify, explore, and extend the Malamar project specifications.
---

# Explore Specs

A collaborative workflow to clarify, explore, and extend ideas in the project specification through a structured Q&A session.

## Workflow

### 1. Read All Context

- Read `README.md` for project overview and document navigation
- Read `SPECS.md` for product specifications (what and why)
- Read `TECHNICAL_DESIGN.md` for detailed design

### 2. Load Meeting Minutes Context

Spawn the `meeting-minutes-recall` agent (via Task tool with `subagent_type: "meeting-minutes-recall"`) in **proactive mode** to surface relevant past discussions. This offloads context to the subagent instead of loading all meeting minutes directly.

When spawning, tell the agent what topic area you're about to explore so it can surface:
- Key decisions relevant to the topic
- Potential conflicts with past decisions
- Gaps (topics never discussed that might need exploration)

### 3. Ask Questions

Ask ONE question at a time, wait for the user's answer, then continue. Go deep into each topic before switching to another.

**Topic Priority (in order):**

1. If the user mentions a topic at the start, stick with it
2. Identify un-completed topics from previous sessions (by judgment - topics that didn't go deep enough)
3. Identify gaps or unclear areas in SPECS.md and TECHNICAL_DESIGN*.md files
4. Randomly and creatively explore topics

**Question Format:**

Every question must follow this format:

```
---

Question #N: [Question title]

[Detailed question body - can include markdown formatting, code blocks, ASCII diagrams, etc.]

*Rationale: [Why this question matters and why it hasn't been answered already. This justifies whether the question is worth asking.]*

Suggestions:
A. [Option name] (Recommended): [Explanation with specific source citations, e.g., "based on Section 2.3 of SPECS.md", "based on Question #5 of SESSION-003.md", "based on the Button component docs from shadcn/ui"]
B. [Option name]: [Explanation with specific source citations]
C. [Option name]: [Explanation with specific source citations]
D. Other: [Invite the user to provide their own answer]
```

**Question Guidelines:**

- Always include specific section/point references when citing sources (SPECS.md, TECHNICAL_DESIGN*.md files, previous sessions)
- Actively fetch external documentation (React docs, shadcn/ui, etc.) to ensure suggestions are accurate and current
- Only mark a suggestion as "(Recommended)" when there's clear alignment with existing specs or previous decisions
- Always include an "Other" option for custom answers
- Use however many suggestions make sense for the question (no fixed minimum/maximum)

### 4. Detect Session Completion

The session is ready to wrap up when:

- All identified topics have been explored deeply
- The number of questions is around 20 or more (guideline, not hard rule)

When these conditions are met, ask the user: "I think we've covered the main topics. Should we wrap up this session, or is there anything else you'd like to explore?"

If the user says done/that's all/wrap up, proceed to save.

### 5. Save the Session

**Determine the next session number** by checking existing files in `MEETING-MINUTES/`.

**Create `MEETING-MINUTES/SESSION-XXX.md`** with the following format:

```markdown
# Session XXX

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1

[Full question with all markdown formatting, code blocks, ASCII diagrams, etc.]

### Answer

[Full answer with all markdown formatting, code blocks, ASCII diagrams, etc.]

## Question #2

[...]

### Answer

[...]
```

**Answer Guidelines:**

- Rephrase for clarity (spelling, grammar, structure) but never lose information
- When the user gives a brief confirmation (e.g., "correct", "yes") to a detailed question, expand the answer to incorporate the reasoning and context from the question so the answer stands on its own
- Preserve all details, decisions, trade-offs, and rationale from the conversation

**Notify the user** that the session has been saved.