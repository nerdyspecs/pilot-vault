# Pilot Vault — Agent Guidelines

This repository is the product, business, and architectural specification vault for Knot (codebase internal name: Pilot). It is maintained as an Obsidian vault.

---

## 1. Multi-Repository Architecture

- **Planning Vault** (this repo, `nerdyspecs/pilot-vault`): Single source of truth for domain logic, user stories, architecture decision records (ADRs), design tokens, roadmaps, and work logs.
- **Application Codebase** (`nerdyspecs/pilot`, sibling directory `../pilot`): Production Rails 8 application, view components, stylesheet modules, and test suite.
- **Agent Skills** (`cqh57/agents-skills`): Shared AI agent skill library, globally loaded into `~/.claude/skills/` and `~/.gemini/config/skills/`.

---

## 2. Vault Conventions & Invariants

1. **Format**: Notes are written in standard Markdown and linked using Obsidian Wikilinks (e.g. `[[ADR-004 Multi-tenant foundation]]`).
2. **ADRs are Immutable**: Never edit past Architecture Decision Records. If an architectural decision evolves or is reversed, add a dated footnote referencing the change, or draft a new ADR.
3. **Citation Hygiene**: Vault citations (e.g. `ADR-004`, `Sprint 3`, `Design Law #7`) belong **only** in this vault. When guiding or writing code in `../pilot`, never leak citation IDs into application source code or commit messages; state the concrete domain rule instead.
4. **Coherence Protocol**: Always run `./bin/coherence-check` before and after making changes to verify that no broken links or citation leaks exist.

---

## 3. Quick Reading Order

When grounding context for development tasks, read in this order:
1. `01 Context/Builder identity.md`
2. `01 Context/Product overview.md`
3. `01 Context/User stories.md`
4. `01 Context/Tech stack.md`
5. `02 Decisions/Decisions.md` (index of ADRs)
6. `01 Context/Design system.md` (authoritative tokens and visual rules)
7. `03 Modules/` (current module sprint plan and backlog)
8. `05 Log/worklog.md` (latest progress and active decisions)
