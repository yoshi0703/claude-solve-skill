# Code Review Rules

## Review Timing

1. After feature implementation, before commit
2. After server-side function changes, before deploy
3. Before git push

## Sub-agents

- **Local (CLI)**: `.claude/agents/codex-code-review` — Uses Codex CLI (external LLM)
- **Loop review**: `.claude/agents/opus-code-review` — Uses Claude Opus

## Severity Levels

- **P0 (Must fix)**: Security vulnerabilities, data integrity issues, crash-inducing bugs
- **P1 (Should fix)**: Performance issues, type safety, UX violations, accessibility
- **P2 (Nice to fix)**: Code style, naming conventions, comments

## Default Check Items

These are the baseline checks applied to all projects. Add your project-specific checks below.

- No hardcoded API keys or secrets
- No leftover `console.log` debug statements
- Proper error handling for async operations
- No `as any` type assertions without justification
- No security vulnerabilities (XSS, injection, etc.)

## Project-Specific Check Items

<!-- Add your project-specific review rules here. Examples: -->
<!-- - React Hooks must only be called at the top level of components -->
<!-- - Database queries must use parameterized queries -->
<!-- - All API endpoints must have authentication checks -->
<!-- - Custom font usage restrictions -->
<!-- - Prohibited phrases or content rules -->
