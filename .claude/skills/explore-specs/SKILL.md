---
name: explore-specs
description: Working along with Irving Dinh on clarifying, exploring and extending the specs of Malamar.
---

# Explore Specs

A collaborative workflow to clarify, explore, and extend ideas in the project specification through a Q&A session.

## Workflow

1. **Read all context**
   - Read `README.md` for project overview and document navigation
   - Read `SPECS.md` for product specifications (what and why)
   - Read `TECHNICAL_DESIGN.md` for technical implementation details (how)
   - Read all files in `MEETING-MINUTES/` folder to understand previous discussions

2. **Ask questions one by one**
   - Identify areas that are unclear, incomplete, or worth exploring
   - Ask ONE question at a time, wait for the user's answer
   - Build on previous answers to go deeper or explore related topics
   - Keep the conversation open-ended - explore whatever seems interesting or important

3. **Provide suggested answers and AI perspective**
   - For each question, provide a **suggested answer** based on patterns and decisions from previous sessions
   - Include an **AI perspective** section with:
     - Judgment on how this aligns with established design principles
     - Potential trade-offs or considerations
     - Recommendation if applicable
   - This helps the user quickly confirm or refine, avoiding redundant discussions

4. **Avoid redundant questions**
   - Before asking, check if similar topics were already covered in previous sessions
   - Reference relevant past decisions when building on established patterns
   - If a question feels redundant, skip it or reframe to explore a genuinely new angle

5. **Detect session completion**
   - When questions start to feel exhausted or no new topics are emerging, ask the user: "I think we've covered the main topics. Should we wrap up this session, or is there anything else you'd like to explore?"
   - If the user says done/that's all/wrap up, proceed to save

6. **Save the session**
   - Determine the next session number by checking existing files in `MEETING-MINUTES/`
   - Create `MEETING-MINUTES/SESSION-XXX.md` with the polished Q&A conversation
   - Format: Use topic headers (`## Topic`), Q&A pairs separated by `---`, support structured answers with bullet points/code blocks where appropriate
   - Notify the user that the session has been saved

## Q&A Format During Session

```markdown
**Q: [The question]?**

**Suggested Answer:** Based on [previous decision/pattern], the answer would likely be [X] because [reasoning].

**AI Perspective:** [Judgment on alignment with design principles, trade-offs, recommendation]

---

User responds, AI acknowledges and moves to next question.
```

## Q&A Format for Session File

```markdown
# Session XXX

## Topic Name

Q: The question asked?

A: The user's answer, potentially with:
- Bullet points
- Code blocks
- Multi-line explanations

---

Q: Next question?

A: Next answer.
```

## Key Design Principles (from previous sessions)

Reference these when formulating suggested answers:

1. **User responsibility over guardrails**: Malamar provides mechanics; users define behavior via instructions. Conflicts, quality, and edge cases are user's responsibility.

2. **Accept risk, keep it simple**: No artificial limits (file sizes, timeouts, iteration caps). Let OS handle cleanup. Trust the user.

3. **Silent failures for non-critical paths**: Network issues fetching knowledge base, notification delivery failures - log but don't surface to user.

4. **Reject invalid actions entirely**: For structured actions (reorder_agents, etc.), validate completely or reject with error - no partial application.

5. **Just-in-time snapshots**: Changes take effect for subsequent operations, not currently running ones.

6. **Cascade delete, no orphans**: When deleting entities, cascade all related data.

7. **No authentication for now**: Local-first, Tailscale for remote access.

8. **Domain agnostic**: Malamar is pure orchestration - no opinions on git workflows, code quality, or domain-specific behavior.

## Tips for Good Questions

- Start broad, then drill into specifics based on answers
- Ask about edge cases and error scenarios
- Explore relationships between entities
- Clarify ambiguous terminology
- Ask about technical decisions and their rationale
- Explore what happens when things go wrong
- Ask about future extensibility when relevant
- Check previous sessions before asking to avoid redundancy
