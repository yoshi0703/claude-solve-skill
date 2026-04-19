# Code Review Rules

## Review Timing

1. After plan creation, before implementation
2. After test creation, before implementation
3. After each implementation step, before next step
4. After all steps complete, before PR (Codex final gate)
5. Before git push

## Sub-agents

- **Loop review**: `.claude/agents/opus-code-review` — Uses Claude Opus (cheap, iterative)
- **Final gate**: `.claude/agents/codex-code-review` — Uses Codex CLI (external LLM)
- **Verification**: `.claude/agents/iai-verifier` — Adversarial testing (PASS/FAIL/PARTIAL)

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
- Authentication checks on server-side functions
- Input validation on user-facing endpoints

## Project-Specific Check Items

<!--
Add your project-specific review rules here.
These are also used by iai-verifier for adversarial testing.

Examples:

### React + TypeScript
- React Hooks must only be called at the top level of components
- No `font-light` (weight 300) — custom font doesn't have this weight
- Server Components by default, "use client" only when necessary

### Python + Django
- All views must use `@login_required` or explicit permission checks
- Database queries must use `.select_related()` to avoid N+1
- No `print()` statements — use `logging.getLogger(__name__)`

### Go
- All exported functions must have godoc comments
- Context must be the first parameter of all functions
- No `panic()` in library code — return errors instead

### Swift iOS
- UI updates must be on the main thread (`@MainActor`)
- No force unwrapping (`!`) — use `guard let` or `if let`
- Accessibility labels on all interactive elements
-->
