# Writing Effective Agent Instructions

This guide helps you write agent instructions that produce reliable, predictable behavior.

---

## Instruction Structure

Every agent instruction should cover these components:

### 1. Role Definition

Start with a clear statement of who the agent is and what they do:

```
You are the Reviewer. Your job is to ensure code quality and catch issues before approval.
```

Keep it brief - one or two sentences. The role definition sets the agent's mindset.

### 2. Skip Triggers

Define when the agent should do nothing and pass to the next agent:

```
## When to SKIP

- No implementation has been done yet (nothing to review)
- You've already reviewed the current changes and the Implementer hasn't responded yet
- The Approver has already approved the work
```

**Why this matters:** Without clear skip conditions, agents may comment unnecessarily ("I have nothing to do") or fail to skip when they should, causing infinite loops.

### 3. Comment Triggers

Define when the agent should add a comment:

```
## When to COMMENT

- You've reviewed changes and found issues (provide specific feedback)
- You've reviewed changes and they look good (give approval for Approver)
- You have suggestions for improvement (even if not blocking)
```

**Why this matters:** Comments are how agents communicate. Agents should comment when they've done meaningful work or have information to share.

### 4. In Review Triggers

Define when the agent should request human attention:

```
## When to request IN REVIEW

- The task requirements are unclear and you need human clarification
- You've identified blockers that require human decision
- You've found a security concern that needs human review
```

**Why this matters:** "In Review" stops the loop and waits for human input. Agents should only request it when genuinely needed.

### 5. Process/Workflow

Describe how the agent should approach their work:

```
## Your Process

1. Read the Planner's plan from the comments
2. Check for any Reviewer feedback that needs addressing first
3. Work through the plan step by step
4. Make actual code changes
5. Summarize what you implemented in your comment
```

### 6. Output Format

Specify how the agent should structure their comments:

```
## Comment Format

Structure your comments like this:

## Code Review

**Status:** [Approved / Changes Requested]

**Issues (must fix):**
- [Specific issue]

**Suggestions (optional):**
- [Non-blocking improvements]
```

---

## Addressing Other Agents

Agents can direct feedback to specific agents by name:

✅ **Good:**
```
Implementer, the catch block in fetchUser() swallows the error silently. Please either re-throw it or log it with context.
```

✅ **Good:**
```
Planner, step 3 of the plan is unclear. What should happen if the user is not authenticated?
```

❌ **Bad:**
```
The error handling needs to be fixed.
```
(Who should fix it? This is ambiguous.)

---

## Common Mistakes

### Mistake 1: No Skip Conditions

❌ **Problem:**
```
You are the Reviewer. Review all code changes and provide feedback.
```

Without skip conditions, the Reviewer will try to review even when there's nothing new, potentially commenting "Nothing new to review" and triggering an unnecessary loop restart.

✅ **Solution:** Always define when to skip.

### Mistake 2: Vague Triggers

❌ **Problem:**
```
## When to request IN REVIEW
- When you're not sure what to do
```

This is too vague. The agent might request In Review for minor uncertainties that could be handled by asking another agent.

✅ **Solution:** Be specific about what warrants human attention.

### Mistake 3: Overlapping Responsibilities

❌ **Problem:**
Planner instruction: "Create a plan and implement quick fixes yourself"
Implementer instruction: "Implement the plan"

Now both agents might implement the same thing, or the Implementer might skip because the Planner already did it.

✅ **Solution:** Keep responsibilities distinct. Each agent should have a clear, non-overlapping role.

### Mistake 4: No Output Format

❌ **Problem:**
```
Review the code and provide feedback.
```

Without a specified format, agents produce inconsistent outputs that are harder for other agents (and humans) to parse.

✅ **Solution:** Define the expected comment structure.

### Mistake 5: Telling Agents to Comment "Nothing to Do"

❌ **Problem:**
```
If you have nothing to do, comment "I have nothing to do at this time."
```

This comment triggers a loop restart, potentially causing infinite loops where agents keep saying they have nothing to do.

✅ **Solution:** If there's nothing to do, skip. Never comment that there's nothing to do.

---

## Good vs Bad Examples

### Role Definition

❌ **Bad:**
```
You help with code.
```

✅ **Good:**
```
You are the Implementer. Your job is to execute the plan and write working code. You translate the Planner's specifications into actual implementations.
```

### Skip Triggers

❌ **Bad:**
```
Skip when appropriate.
```

✅ **Good:**
```
## When to SKIP

- No plan exists yet (wait for the Planner)
- The Reviewer has outstanding feedback you haven't addressed yet
- All planned work is already complete and approved
```

### Feedback

❌ **Bad:**
```
The code has problems.
```

✅ **Good:**
```
Implementer, the validateEmail function on line 45 doesn't handle the case where the email is null. Please add a null check before the regex test.
```

---

## Template

Here's a template for creating agent instructions:

```
You are the [AGENT NAME]. Your job is to [PRIMARY RESPONSIBILITY].

## When to SKIP

- [Condition 1]
- [Condition 2]
- [Condition 3]

## When to COMMENT

- [Condition 1]
- [Condition 2]

## When to request IN REVIEW

- [Condition 1 - should be rare and specific]
- [Condition 2]

## Your Process

1. [Step 1]
2. [Step 2]
3. [Step 3]

## Comment Format

[Specify the expected structure of comments]

## Important

- [Key guideline 1]
- [Key guideline 2]
```

---

## Tips for the Malamar Agent

When helping users write instructions:

1. **Ask about their workflow** - What does success look like? What agents exist?
2. **Identify handoff points** - When does one agent's work trigger another's?
3. **Define clear boundaries** - Each agent should have distinct, non-overlapping responsibilities
4. **Start simple** - Begin with basic triggers and refine based on actual behavior
5. **Include examples** - Show the agent what good output looks like
