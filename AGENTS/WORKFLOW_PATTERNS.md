# Workflow Patterns

This guide describes abstract workflow patterns and how to apply them to different domains.

---

## Understanding Workflow Patterns

Malamar is domain-agnostic - the same loop mechanics can orchestrate agents for software development, content writing, data processing, or any other task achievable via AI CLIs. The key is choosing the right pattern for your use case.

---

## Pattern 1: Linear Pipeline

```
Agent A → Agent B → Agent C → Agent D
   ↓         ↓         ↓         ↓
 Create   Execute    Verify    Approve
```

**How it works:**
- Each agent has a distinct phase of work
- Work flows in one direction through the pipeline
- Later agents verify/refine the work of earlier agents
- Loop restarts only when feedback needs to go backward

**Best for:**
- Processes with clear, sequential phases
- Work that benefits from separation of concerns
- Situations where fresh perspective at each stage is valuable

**Agent count:** Typically 3-5 agents

### Example: Software Development

| Agent | Role | Triggers In Review When |
|-------|------|------------------------|
| **Planner** | Analyzes requirements, creates implementation plan | Requirements unclear, needs human decision |
| **Implementer** | Executes the plan, writes code | Blocked by external dependency |
| **Reviewer** | Reviews code quality, provides feedback | Critical security/architecture issue |
| **Approver** | Verifies against requirements, final check | Work complete and ready for human |

**Flow:**
1. Planner creates plan → comments
2. Implementer follows plan → comments with what was done
3. Reviewer checks implementation → comments with feedback or approval
4. If feedback: loop restarts, Implementer addresses it
5. If approved: Approver verifies and requests In Review

---

## Pattern 2: Iterative Refinement

```
     ┌──────────────────┐
     ↓                  │
Agent A → Agent B → Agent C
 Create    Refine    Evaluate
                        │
                        ↓
                   (loop until good)
```

**How it works:**
- One agent creates initial output
- Another agent refines/improves it
- A third agent evaluates quality
- Loop continues until quality threshold is met

**Best for:**
- Creative work that benefits from iteration
- Tasks where "good enough" requires multiple passes
- Situations where quality improves with refinement

**Agent count:** Typically 2-3 agents

### Example: Content Writing

| Agent | Role | Triggers In Review When |
|-------|------|------------------------|
| **Researcher** | Gathers information, creates outline, writes first draft | Topic requires human expertise, sources unclear |
| **Writer** | Expands and refines the draft, improves flow and clarity | N/A (passes to Editor) |
| **Editor** | Reviews for quality, accuracy, style; provides feedback or approves | Content ready for publication |

**Flow:**
1. Researcher creates outline and draft → comments
2. Writer refines the draft → comments with improvements
3. Editor reviews → comments with feedback or approval
4. If feedback: loop restarts, Writer addresses it
5. If approved: Editor requests In Review

---

## Pattern 3: Gatekeeper

```
Agent A → Agent B (Gatekeeper)
   ↓           ↓
 Work      Pass/Fail
              ↓
         (In Review if pass)
```

**How it works:**
- One agent does the main work
- A gatekeeper agent validates the work
- If validation fails, feedback goes back to the worker
- If validation passes, work moves to human review

**Best for:**
- Quality control workflows
- Compliance checking
- Simple approve/reject decisions

**Agent count:** 2 agents

---

## Choosing a Pattern

| If your work... | Consider... |
|-----------------|-------------|
| Has distinct phases (plan, do, check) | Linear Pipeline |
| Requires multiple refinement passes | Iterative Refinement |
| Needs a single quality gate | Gatekeeper |
| Is complex with many aspects | Linear Pipeline with more agents |
| Is creative/generative | Iterative Refinement |

---

## Adapting Patterns

### Adding Agents

You can add specialists to any pattern:

**Linear Pipeline + Specialist:**
```
Planner → Implementer → Security Reviewer → General Reviewer → Approver
```

### Removing Agents

For simpler workflows, reduce agents:

**Minimal Pipeline:**
```
Implementer → Reviewer
```

### Hybrid Patterns

Combine patterns for complex workflows:

**Research + Implementation:**
```
Researcher → Planner → Implementer → Reviewer → Approver
```

---

## Pattern Anti-Patterns

### Too Many Agents

❌ **Problem:** 8 agents for a simple task
- More agents = more loop iterations = higher cost and time
- Agents may have overlapping responsibilities

✅ **Solution:** Start with 2-3 agents, add more only when needed

### Circular Dependencies

❌ **Problem:** Agent A waits for Agent B, Agent B waits for Agent A
- Creates deadlock where both agents skip

✅ **Solution:** Ensure clear ordering - one agent acts first, others respond

### No Clear Termination

❌ **Problem:** Agents keep refining forever without requesting In Review

✅ **Solution:** At least one agent must have clear "request In Review" triggers

---

## Tips for the Malamar Agent

When helping users design workflows:

1. **Start with the goal** - What does "done" look like?
2. **Identify phases** - What distinct steps are needed?
3. **Define handoffs** - How does one agent's output become another's input?
4. **Plan for feedback** - How will later agents send work back to earlier ones?
5. **Set termination** - Which agent requests In Review and when?
