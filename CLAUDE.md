# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SDD Workflow Toolkit is a Claude Code skill collection that supports Spec-Driven Development (SDD) workflows. It includes independent code reviews using OpenAI Codex CLI, section-based progress tracking, E2E evidence collection, and Kiro spec integration to eliminate blind spots of same-model self-review.

## Prerequisites

```bash
# Install Codex CLI
npm install -g @openai/codex

# Verify installation
codex auth status
```

## Directory Structure

```
skill/
├── SKILL.md              # Main skill definition (~400 lines, invoked by /sdd-codex-review)
├── prompts/              # Codex review prompt templates
│   ├── requirements-review.md
│   ├── design-review.md
│   ├── tasks-review.md
│   ├── impl-review.md
│   ├── section-impl-review.md  # Section-based review
│   ├── e2e-evidence-prompt.md
│   ├── arch-review.md          # Architecture review (medium/large)
│   └── cross-check-review.md   # Cross-cutting concerns (large)
├── workflows/            # Detailed workflow documentation
│   ├── phase-workflows.md
│   ├── section-detection.md
│   ├── e2e-evidence.md
│   ├── scale-strategies.md
│   └── context-saving.md
├── reference/
│   └── spec-json-format.md
├── evaluations/          # Test scenarios
└── templates/
    └── review-report.md

integration/              # Integration snippets for CLAUDE.md and Kiro commands
examples/                 # Sample spec.json format
```

## Skill Invocation

```bash
# Phase-based reviews
/sdd-codex-review requirements [feature-name]
/sdd-codex-review design [feature-name]
/sdd-codex-review tasks [feature-name]
/sdd-codex-review impl [feature-name]

# Section-based impl review (recommended)
/sdd-codex-review impl-section [feature-name] [section-id]
/sdd-codex-review impl-pending [feature-name]

# E2E evidence collection (for [E2E] tagged sections)
/sdd-codex-review e2e-evidence [feature-name] [section-id]

# Auto-progress mode
/sdd-codex-review auto [feature-name]
```

## Key Architecture Concepts

### Review Flow
1. Read target files from `.kiro/specs/[feature]/`
2. Execute `codex exec --sandbox read-only "[prompt]"` with appropriate template
3. Parse JSON verdict from Codex output (regex for ```json blocks)
4. Auto-apply suggestions if NEEDS_REVISION (max 5 iterations)
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

Pass criteria: `blocking = 0` (any blocking issue = ok: false)

Severity levels:
- **blocking**: Must fix (1+ = ok: false)
- **advisory**: Recommendations only (ok: true possible)

### Section-Based Review (Recommended for impl)

Sections are defined by `##` headings in tasks.md:

```markdown
## Section 1: Core Foundation
### Task 1.1: Define base types
**Creates:** `src/types/base.ts`

## Section 2: Feature Implementation [E2E]
### Task 2.1: Build main component
**Creates:** `src/components/Main.tsx`
**E2E:** Initial display, user interaction
```

Section completion detection:
1. Parse `##` headings from tasks.md
2. Extract files from `**Creates:**`/`**Modifies:**`
3. Check if all files exist
4. Trigger review when: complete AND not reviewed

### Scale-Based Strategies (impl phase)

| Scale | Files | Lines | Strategy |
|-------|-------|-------|----------|
| small | ≤3 | ≤100 | diff only |
| medium | 4-10 | 100-500 | arch → diff |
| large | >10 | >500 | arch → parallel diff (3-5 groups) → cross-check |

### E2E Evidence Collection

Triggered for `[E2E]` tagged sections AFTER Codex review approval:
1. Execute Playwright MCP to capture screenshots
2. Save to `.context/e2e-evidence/[feature]/[section]/`
3. Update `e2e_evidence` in spec.json

E2E failure is NOT blocking - it's for evidence only.

### Context-Saving Mode (Automatic)

1. Saves state to `.context/sdd-review-state.json`
2. Runs Codex in background (`run_in_background=true`)
3. Offers context compression during wait
4. Restores state after completion
5. Cleans up temporary files

### Codex Commands

```bash
# Initial review (read-only sandbox)
codex exec --sandbox read-only "[prompt]"

# Resume previous session (for re-reviews)
codex exec --sandbox read-only resume --last "[feedback]"
codex exec --sandbox read-only resume [SESSION_ID] "[feedback]"

# Implementation review with diff
codex exec --sandbox read-only review --base main "[instructions]"
```

## Kiro Spec Structure Expected

```
.kiro/specs/[feature]/
├── requirements.md   # EARS format requirements
├── design.md         # Architecture and interface design
├── tasks.md          # Implementation tasks (with ## sections)
└── spec.json         # Phase tracking, approvals, section_tracking
```

## spec.json Key Fields

```json
{
  "phase": "requirements-approved | design-approved | ready-for-implementation | impl-approved",
  "approvals": { "requirements": { "approved": true }, ... },
  "section_tracking": {
    "enabled": true,
    "sections": {
      "section-1-core": {
        "status": "complete | in_progress | pending",
        "reviewed": true,
        "e2e_required": false,
        "e2e_evidence": null
      }
    }
  },
  "codex_reviews": {
    "requirements": { "session_id": "...", "final_verdict": "OK" },
    "impl": { "mode": "section", "sections": { ... } }
  }
}
```

## Development Notes

- Prompt templates use `{{VARIABLE}}` placeholders
- Error handling: 1 retry for normal errors, split files on timeout
- Polling: 60s intervals, max 20 attempts
- Max iterations per phase: 5 (then request user intervention)
