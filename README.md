# Malamar

> **Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity.**

## About This Repository

This is the **specification repository** for Malamar. It contains product specifications, technical design documents, and the knowledge base used by the Malamar agent.

The actual source code lives at: [malamar-dev/malamar](https://github.com/malamar-dev/malamar) *(placeholder)*

## Documentation

| Document | Description |
|----------|-------------|
| [SPECS.md](./SPECS.md) | Product specifications: what Malamar does and why |
| [TECHNICAL_DESIGN.md](./TECHNICAL_DESIGN.md) | Technical design: how Malamar is implemented |
| [ROADMAP.md](./ROADMAP.md) | Future vision and potential enhancements |
| [AGENTS/](./AGENTS/) | Knowledge base for the Malamar agent |
| [APPENDIX_ONBOARDING_RESOURCES.md](./APPENDIX_ONBOARDING_RESOURCES.md) | Resources created on first startup |

## Specialized Meeting Minutes

Some meeting minute sessions are too detailed or task-specific to be reflected in SPECS.md or TECHNICAL_DESIGN.md. These sessions focus deeply on implementation details for specific development phases. Read them when working on the related area.

| Session | Focus | When to Read |
|---------|-------|--------------|
| [SESSION-012](./MEETING-MINUTES/SESSION-012.md) | Backend repository structure | When setting up the backend codebase, understanding folder organization, module patterns, testing strategy, or development tooling |

## Writing Tips

When updating documentation in this repository:

1. **No inline notes about future features**: Don't add notes like "Feature X is planned for future version" inside SPECS.md or TECHNICAL_DESIGN.md. These documents should only contain what IS in the current version. Future features belong exclusively in ROADMAP.md.

2. **No fragmented annotations**: Avoid adding scattered `> **Note:**` blocks that reference other documents or explain what's missing. Keep documents clean and self-contained.

3. **Remove, don't annotate**: If a feature is removed or deferred, remove it from the document entirely rather than adding a note explaining its absence.

4. **API changes require full updates**: When adding UI features that need backend support (like search), ensure corresponding API endpoints are added to the API documentation.
