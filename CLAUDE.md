# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SDD Codex Review Plugin is a Claude Code skill that enables independent code review using OpenAI Codex CLI. It reviews artifacts from the Kiro Spec-Driven Development workflow (requirements, design, tasks, implementation) to eliminate blind spots of same-model self-review.

## Directory Structure

```
skill/
├── SKILL.md              # Main skill definition (invoked by /sdd-codex-review)
├── prompts/              # Codex review prompt templates
│   ├── requirements-review.md
│   ├── design-review.md
│   ├── tasks-review.md
│   └── impl-review.md
└── templates/
    └── review-report.md  # Report format template

integration/              # Integration snippets for CLAUDE.md and Kiro commands
examples/                 # Sample spec.json format
```

## Skill Invocation

The skill is triggered via:
```bash
/sdd-codex-review [phase] [feature-name]
```

Phases: `requirements`, `design`, `tasks`, `impl`, `auto`

## Key Architecture Concepts

### Review Flow
1. Read target files from `.kiro/specs/[feature]/`
2. Execute `codex exec --full-auto "[prompt]"` with appropriate prompt template
3. Parse JSON verdict from Codex output
4. Auto-apply suggestions if NEEDS_REVISION (max 6 cycles)
5. Update `spec.json` with review results and session ID

### Critical Constraints

**Simulation Prohibited**: This skill MUST execute actual Codex CLI - never simulate review results. Session ID must be included in all reports.

### Review Verdicts by Phase

| Phase | PASS | FAIL |
|-------|------|------|
| requirements | OK | NEEDS_REVISION |
| design | GO | NO_GO |
| tasks | APPROVED | NEEDS_REVISION |
| impl | APPROVED | NEEDS_REVISION |

Pass criteria: critical = 0 AND medium <= 2

### Context-Saving Mode

The skill automatically:
1. Saves state to `.context/sdd-review-state.json`
2. Runs Codex in background (`run_in_background=true`)
3. Offers context compression during wait
4. Restores state after completion
5. Cleans up temporary files

### Codex Commands

```bash
# Initial review
codex exec --full-auto "[prompt]"

# Resume previous session (for re-reviews)
codex exec --full-auto resume --last "[feedback]"

# Implementation review with diff
codex exec --full-auto review --base main "[instructions]"
```

## Kiro Spec Structure Expected

```
.kiro/specs/[feature]/
├── requirements.md   # EARS format requirements
├── design.md         # Architecture and interface design
├── tasks.md          # Implementation tasks
└── spec.json         # Phase tracking and approval status
```

## Development Notes

- Prompt templates use `{{VARIABLE}}` placeholders
- spec.json tracks `phase`, `approvals`, and `codex_reviews` history
- Error handling: 3 retries with exponential backoff (5s, 15s, 45s)
- JSON extraction from Codex output uses regex for ```json blocks
