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

3. **Detect session completion**
   - When questions start to feel exhausted or no new topics are emerging, ask the user: "I think we've covered the main topics. Should we wrap up this session, or is there anything else you'd like to explore?"
   - If the user says done/that's all/wrap up, proceed to save

4. **Save the session**
   - Determine the next session number by checking existing files in `MEETING-MINUTES/`
   - Create `MEETING-MINUTES/SESSION-XXX.md` with the polished Q&A conversation
   - Format: Use topic headers (`## Topic`), Q&A pairs separated by `---`, support structured answers with bullet points/code blocks where appropriate
   - Notify the user that the session has been saved

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

## Tips for Good Questions

- Start broad, then drill into specifics based on answers
- Ask about edge cases and error scenarios
- Explore relationships between entities
- Clarify ambiguous terminology
- Ask about technical decisions and their rationale
- Explore what happens when things go wrong
- Ask about future extensibility when relevant
