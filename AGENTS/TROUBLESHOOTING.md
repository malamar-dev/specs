# Troubleshooting

This guide helps diagnose and fix common issues with Malamar workflows.

---

## Agents Stuck in Infinite Loop

**Symptom:** Agents keep commenting back and forth without ever reaching "In Review".

### Cause 1: Conflicting Feedback

Agents disagree and keep giving each other feedback that the other rejects.

**Solution:**
1. Cancel the task to stop the loop
2. Add a user comment to break the deadlock (e.g., "Reviewer, accept the current implementation")
3. Adjust agent instructions to be less strict or more deferential

### Cause 2: Missing Skip Conditions

An agent comments "I have nothing to do" or similar, which triggers a loop restart.

**Solution:**
1. Check agent instructions for proper skip conditions
2. Remove any instructions telling agents to comment when they have nothing to do
3. Agents should SKIP (not comment) when they have nothing to contribute

### Cause 3: No Termination Condition

No agent ever requests "In Review".

**Solution:**
1. Ensure at least one agent (typically the last one) has clear "request In Review" triggers
2. Define what "done" looks like for the workflow

---

## CLI Not Detected / Unhealthy

**Symptom:** CLI shows as "Unhealthy" in settings, or agents fail with CLI errors.

### Cause 1: Binary Not in PATH

The CLI executable isn't found.

**Solution:**
1. Verify the CLI is installed: run `which claude` (or gemini, codex, opencode) in terminal
2. If installed but not found, add the binary path to CLI Settings
3. Restart Malamar after PATH changes

### Cause 2: Missing Credentials

The CLI requires API keys or authentication.

**Solution:**
1. Set up credentials as the CLI requires (environment variables, config files)
2. Add necessary environment variables in CLI Settings
3. Click "Refresh CLI Status" to re-test

### Cause 3: Rate Limiting

The CLI is temporarily rate-limited by the provider.

**Solution:**
1. Wait and try again later
2. The health check runs every 5 minutes and will update automatically
3. Consider using a different CLI for some agents to distribute load

---

## Task Never Completes (All Agents Skip)

**Symptom:** Task goes directly to "In Review" without any agent doing work.

### Cause 1: All Skip Conditions Met

Every agent's skip conditions are satisfied from the start.

**Solution:**
1. Check agent instructions - are skip conditions too broad?
2. Ensure at least one agent has conditions to act on a fresh task
3. The first agent (often Planner) should have minimal skip conditions for new tasks

### Cause 2: Missing Context

Agents can't proceed because the task lacks necessary information.

**Solution:**
1. Add more detail to the task description
2. Include repository URLs, file paths, or other context agents need
3. Check workspace instruction for missing project context

### Cause 3: Wrong Agent Order

Agents are in the wrong order, causing later agents to skip waiting for earlier ones.

**Solution:**
1. Review agent order in workspace settings
2. Ensure agents that produce output come before agents that consume it
3. Example: Planner should come before Implementer

---

## Agent Not Following Instructions

**Symptom:** Agent behavior doesn't match what the instruction specifies.

### Cause 1: Instruction Too Vague

The instruction doesn't give clear enough guidance.

**Solution:**
1. Add specific examples of expected output
2. Define explicit triggers for each action (skip, comment, In Review)
3. Structure the instruction with clear sections

### Cause 2: Conflicting Directives

The instruction contains contradictory guidance.

**Solution:**
1. Review instruction for conflicts
2. Prioritize directives (e.g., "Most importantly, always...")
3. Remove or clarify ambiguous sections

### Cause 3: Context Overload

Too much context makes the agent lose focus on key instructions.

**Solution:**
1. Put the most important instructions at the beginning
2. Use clear headers and formatting
3. Keep instructions focused - move reference material to comments if needed

---

## Unexpected "In Review" Transitions

**Symptom:** Task moves to "In Review" earlier than expected.

### Cause 1: Overly Sensitive Triggers

An agent's "In Review" triggers are too easily satisfied.

**Solution:**
1. Review which agent requested In Review (check activity log)
2. Tighten the In Review conditions for that agent
3. Make conditions more specific (e.g., "only when blocked by external dependency" not "when unsure")

### Cause 2: Error Handling

An error occurred and triggered In Review through the error flow.

**Solution:**
1. Check task comments for system error messages
2. Address the underlying error (CLI issue, malformed output, etc.)
3. Errors should NOT trigger In Review - if they are, check if an agent is mishandling errors

---

## Comments Not Appearing

**Symptom:** Agent ran but no comment was added.

### Cause 1: Agent Returned Skip

The agent decided to skip rather than comment.

**Solution:**
1. Check activity log - does it show "agent_started" and "agent_finished" without "comment_added"?
2. Review agent's skip vs comment triggers
3. The agent's skip conditions may be too broad

### Cause 2: Malformed Output

The agent's response couldn't be parsed.

**Solution:**
1. Check for system error comments about malformed output
2. This usually means the CLI produced invalid JSON
3. Check CLI health and retry

---

## Working Directory Issues

**Symptom:** Agents can't find files or make changes in the wrong location.

### Cause 1: Static Path Doesn't Exist

The configured directory path is invalid.

**Solution:**
1. Verify the path exists: `ls /path/to/directory`
2. Check for typos in workspace settings
3. Ensure the path is absolute, not relative

### Cause 2: Permission Problems

Malamar can't read or write to the directory.

**Solution:**
1. Check directory permissions
2. Ensure the user running Malamar has access
3. Consider using a different directory with proper permissions

### Cause 3: Wrong Mode Selected

Using temp folder when static is needed, or vice versa.

**Solution:**
1. Review working directory mode in workspace settings
2. Static: for existing projects, persistent work
3. Temp Folder: for isolation, clean slate per task

---

## Diagnostic Steps

When troubleshooting any issue:

1. **Check Activity Log**
   - What events occurred?
   - Which agents ran?
   - Were there errors?

2. **Check Comments**
   - Look for system comments with error details
   - Review agent comments for clues about their decisions

3. **Check CLI Status**
   - Are all assigned CLIs healthy?
   - Refresh CLI status to get current state

4. **Review Agent Instructions**
   - Do skip/comment/In Review triggers make sense?
   - Are there conflicting directives?

5. **Simplify and Test**
   - Try with fewer agents
   - Test with a simple task
   - Add one complexity at a time

---

## Tips for the Malamar Agent

When helping users troubleshoot:

1. **Ask for symptoms** - What exactly is happening vs what they expect?
2. **Check the basics first** - CLI health, agent order, skip conditions
3. **Look at the activity log** - Concrete evidence of what happened
4. **Suggest incremental fixes** - One change at a time to isolate the cause
5. **Offer to review instructions** - Often the issue is in the instruction text
