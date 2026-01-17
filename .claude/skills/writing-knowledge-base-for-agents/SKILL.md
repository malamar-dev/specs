---
name: writing-knowledge-base-for-agents
description: Reference document for writing and updating the AGENTS/ knowledge base that the Malamar agent uses to help users.
invocable: true
---

# Writing Knowledge Base for Agents

A comprehensive reference for writing and updating content in the `AGENTS/` folder. This knowledge base is consumed by the Malamar agent to help users create agents, write instructions, troubleshoot issues, and understand how to use Malamar effectively.

## Target Audience

The content in `AGENTS/` is written for **the Malamar agent (AI)**, not human users directly.

The Malamar agent reads this knowledge base when users ask questions like:
- "Help me create a code reviewer agent"
- "My agents keep disagreeing with each other, what should I do?"
- "How should I structure my workflow for blog writing?"
- "The Implementer keeps skipping, why?"

The Malamar agent needs to **understand Malamar deeply** to give good answers and generate effective agent instructions.

## Content Principles

### 1. Standalone and Self-Contained

The `AGENTS/` knowledge base must be **self-contained**. The Malamar agent should not need to read `SPECS.md` or `TECHNICAL_DESIGN.md` to understand the content.

- **Do**: Explain concepts fully within the AGENTS/ files
- **Don't**: Say "see SPECS.md for details" or assume prior knowledge

**Rationale**: SPECS.md and TECHNICAL_DESIGN.md are for humans building Malamar. They contain implementation details (schemas, API endpoints, polling intervals) that are noise for the Malamar agent trying to help a user write instructions.

### 2. Practical with Examples

Lead with practical explanations and concrete examples. Include edge cases that users are likely to encounter or ask about.

- **Do**: "When Reviewer comments feedback, the loop restarts. Implementer will see the comment in the next pass and can respond."
- **Don't**: "Agents run sequentially." (too abstract)

**Level of detail**:
- Practical explanations with cause-and-effect
- Concrete examples the Malamar agent can reference and adapt
- Edge cases that prevent real user problems
- Skip implementation details (database schemas, polling intervals)

### 3. Structured Format

Use consistent formatting that the AI can parse effectively:

**Headers**: Clear hierarchy for quick navigation
```markdown
## Main Topic
### Subtopic
```

**Examples**: In code blocks or quoted blocks
```markdown
Example instruction:
> You are the Reviewer. Your responsibility is to...
```

**Actionable patterns**: "When X, do Y" format
```markdown
**When to use `skip`:**
- The task doesn't require your expertise
- Another agent should handle it
- The task is already complete
```

**Anti-patterns**: Explicitly called out
```markdown
**Don't do this:**
- Never comment "I have nothing to do" - use `skip` instead
- Never request status changes other than "in_review"
```

### 4. Entry Point Structure

`AGENTS/README.md` serves as the entry point with:
1. **Orientation** - What is this knowledge base, what's the Malamar agent's role
2. **Core context** - Essential concepts needed before diving deeper
3. **Index** - Links to detailed files with brief descriptions

Keep README.md focused as an index; put depth in the linked files.

## Workflow

Follow this workflow for both **creating new files** and **updating existing files** in `AGENTS/`.

### Phase 1: Prepare

Before writing anything, read the latest source documents to ensure you have current knowledge:

1. Read `README.md` for project overview
2. Read `SPECS.md` for product specifications
3. Read `TECHNICAL_DESIGN.md` for technical details
4. Read all files in `MEETING-MINUTES/` for recent decisions
5. Read existing `AGENTS/` content (if updating or adding related content)

**Important**: Always do this, even for small updates. The specs may have changed since the AGENTS/ content was last written.

### Phase 2: Write/Update

Create or modify the content following the Content Principles above:

- Write for the Malamar agent as the audience
- Make content standalone and self-contained
- Include practical examples
- Use structured formatting
- Call out anti-patterns explicitly

### Phase 3: Verify

After writing, spawn a subagent to fact-check the **entire file** - even if you only changed one line or one character.

**Fact-check scope**:
1. Verify against source docs (SPECS.md, TECHNICAL_DESIGN.md, MEETING-MINUTES/)
2. Verify against other AGENTS/ files for cross-file consistency

**Fact-check behavior**:
- Identify discrepancies clearly: "Line 42 says X, but SPECS.md line 128 says Y"
- Suggest corrected text that aligns with the source
- User decides whether to accept, modify, or intentionally keep the difference

**Why fact-check the entire file?** A small change might not break anything directly, but other parts of the file may contain stale information that needs updating based on spec changes.

### Phase 4: Finalize

Confirm changes with the user before committing:

- Summarize what was added/changed
- Note any corrections from the fact-check
- Get explicit approval before saving

## Cross-File Consistency

The `AGENTS/` knowledge base may have multiple files. When updating one file, ensure consistency with others:

- If you change how a concept is described in one file, check if other files reference that concept
- The fact-check subagent should catch inconsistencies, but be mindful while writing
- When in doubt, grep for related terms across AGENTS/ files

## Detecting Obsolete Knowledge

Malamar's specs evolve through discussions in `MEETING-MINUTES/`. Content in `AGENTS/` can become outdated.

**How to handle**:
- Always read latest docs before writing (Phase 1)
- The fact-check process (Phase 3) catches discrepancies
- When you notice outdated content while working on something else, note it for a future update

**Signs of obsolete knowledge**:
- References to features that have been renamed or removed
- Action types or status values that don't match current specs
- Workflow descriptions that don't match current loop behavior
- Missing coverage of new features (like Chat)

## What NOT to Include

The `AGENTS/` knowledge base should **not** include:

- Database schemas or table structures
- API endpoint definitions
- Internal implementation details (polling intervals, cleanup job schedules)
- Configuration options (environment variables, CLI flags)
- UI implementation details

These belong in `SPECS.md` and `TECHNICAL_DESIGN.md`. The `AGENTS/` knowledge base focuses on **how to use Malamar effectively**, not how Malamar is built.

## Summary Checklist

Before finalizing any AGENTS/ content:

- [ ] Read latest README.md, SPECS.md, TECHNICAL_DESIGN.md, MEETING-MINUTES/
- [ ] Content is standalone (no external references required)
- [ ] Content is practical with examples
- [ ] Format uses headers, code blocks, "When X, do Y" patterns
- [ ] Anti-patterns are explicitly called out
- [ ] Spawned subagent to fact-check entire file against source docs
- [ ] Spawned subagent to fact-check against other AGENTS/ files
- [ ] Reviewed and resolved suggested corrections
- [ ] Confirmed changes with user
