# golden-hour

## Project Overview

This is a BMad (AI-assisted software development) project in early planning stage. No implementation artifacts exist yet.

## Project Structure

```
_bmad/                    # BMad framework configuration
  config.toml             # Project-level config (read-only, installer-managed)
  config.user.toml        # User-level config (read-only, installer-managed)
  custom/                 # Custom overrides (safe to edit)
_bmad-output/
  planning-artifacts/     # PRDs, architecture docs, epics, stories
  implementation-artifacts/ # Code and implementation outputs
docs/                     # Project knowledge base
CLAUDE.md                 # This file
```

## BMad Workflow

Use BMad skills (invoked via `/skill-name`) to drive structured product development:

1. **Discovery** — `/bmad-product-brief`, `/bmad-prfaq`, `/bmad-domain-research`, `/bmad-market-research`
2. **Requirements** — `/bmad-create-prd`, `/bmad-agent-pm` (talk to John)
3. **Design** — `/bmad-create-ux-design`, `/bmad-agent-ux-designer` (talk to Sally)
4. **Architecture** — `/bmad-create-architecture`, `/bmad-agent-architect` (talk to Winston)
5. **Planning** — `/bmad-create-epics-and-stories`, `/bmad-sprint-planning`
6. **Implementation** — `/bmad-dev-story`, `/bmad-agent-dev` (talk to Amelia)
7. **Review** — `/bmad-code-review`, `/bmad-qa-generate-e2e-tests`

Use `/bmad-help` to get guidance on what to do next.

## Team Agents

| Agent | Name | Role |
|-------|------|------|
| `bmad-agent-analyst` | Mary | Business Analyst |
| `bmad-agent-pm` | John | Product Manager |
| `bmad-agent-ux-designer` | Sally | UX Designer |
| `bmad-agent-architect` | Winston | System Architect |
| `bmad-agent-dev` | Amelia | Senior Software Engineer |
| `bmad-agent-tech-writer` | Paige | Technical Writer |

## Configuration

- **Output language:** English
- **User skill level:** Intermediate
- **Planning artifacts:** `_bmad-output/planning-artifacts/`
- **Implementation artifacts:** `_bmad-output/implementation-artifacts/`
- **Project knowledge:** `docs/`
