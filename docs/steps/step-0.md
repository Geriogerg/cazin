# Step 0 brief — Foundations

## Goal
An empty but fully guarded monorepo: every guardrail from `ENGINEERING.md` is in place and **proven to work** before any business code exists.

## Prerequisites (done by hand, before Claude Code)
- Native Windows. Repo in a short path outside OneDrive (e.g. `C:\dev\casino`).
- `git config --global core.autocrlf false` and `git config --global core.longpaths true`.
- Node 24 LTS, `corepack enable` (pnpm), Git configured, gitleaks Windows binary (`winget install gitleaks`).
- PostgreSQL: Docker Desktop if the machine handles it, otherwise native PostgreSQL for Windows.
- Empty private GitHub repo created.
- Files in place: `ENGINEERING.md`, `CLAUDE.md`, `docs/context.md`, this brief in `docs/steps/step-0.md`.

## Scope
1. **Repo basics**: `.gitignore`, `.gitattributes` (`* text=auto eol=lf`), `.editorconfig`, `.nvmrc` (24), root `package.json` with `engines` and `packageManager` pinned.
2. **Workspaces**: `packages/engine`, `packages/shared`, `apps/server`, `apps/web`. Each with a minimal placeholder and one trivial test. `apps/server` only starts and answers a health check. `apps/web` is the default Vite React page.
3. **TypeScript**: shared base config with `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `forceConsistentCasingInFileNames`.
   All `package.json` scripts and hooks must be cross-platform (Windows locally, Linux in CI): no bash syntax.
4. **ESLint + Prettier** with the project's custom rules:
   - `Math.random` forbidden everywhere;
   - `console.log` forbidden outside scripts;
   - in `packages/engine`: `Date`, `Date.now`, timers, and imports of Node I/O modules forbidden;
   - `number` type forbidden on properties/variables named like amounts (`amount`, `stake`, `balance`, `pot`, `fee`), via a `no-restricted-syntax` selector or a small local rule;
   - `any` and `@ts-ignore` forbidden.
5. **Architecture check** (dependency-cruiser): engine imports nothing outside itself; web never imports server; engine never imports shared runtime code with I/O.
6. **Tests**: Vitest across workspaces, fast-check installed, one example property test in `packages/engine`.
7. **Postgres**: Docker Compose file with two databases (dev, test), usable as-is in CI. If Docker is too heavy locally, the same two databases on a native install, with connection strings in `.env`. `.env.example` committed, `.env` ignored.
8. **Scripts**: `lint`, `format`, `typecheck`, `arch`, `test`, and `check` (runs all of them).
9. **Pre-commit** (husky + lint-staged): format, lint on staged files, gitleaks.
10. **CI** (GitHub Actions) on every push and PR: install (frozen lockfile) → lint → typecheck → arch → test → gitleaks → `pnpm audit` → Semgrep.
11. **Dependabot** configured for npm and GitHub Actions.
12. **Claude Code config** in `.claude/settings.json`:
    - after each edit: format + lint + typecheck hooks;
    - deny `rm -rf`, `git push --force`, and `DROP` / `TRUNCATE`.
13. **Docs**:
    - `docs/adr/` with one ADR per item of "Decisions made" in the context (short: context, decision, consequences);
    - `docs/glossary.md`;
    - empty `docs/specs/`;
    - local `CLAUDE.md` in `packages/engine` (purity rules).
14. Update the "Commands" section of the root `CLAUDE.md` with the real scripts.

## Out of scope
Any game logic, provably fair code, database schema or migrations, Drizzle setup, auth, WebSocket, UI work, Stryker and Playwright configuration (later steps).

## Exit checklist (each item needs evidence in the report)
- [ ] Fresh clone in a new folder → `pnpm install --frozen-lockfile && pnpm check` passes. Paste the full output.
- [ ] **Every custom lint rule is proven**: a fixture file (or a temporary edit) that violates it, the lint output showing the error, then reverted. One output per rule.
- [ ] **dependency-cruiser is proven**: a deliberate forbidden import (engine → server code), its failing output, then reverted.
- [ ] **gitleaks is proven**: an attempted commit with a fake secret is blocked. Paste the output.
- [ ] Strict TypeScript is proven: one deliberate `noUncheckedIndexedAccess` error shown, then reverted.
- [ ] CI run green on GitHub: paste the job summary (list of steps and their status).
- [ ] Both local databases reachable (`psql` or `pg_isready` output).
- [ ] Line endings: `git ls-files --eol` shows no CRLF in the index.
- [ ] Claude Code hooks fire after an edit, and a denied command is refused. Paste the evidence.
- [ ] `git ls-files` shows no `.env` and no secret.
- [ ] List of ADRs created (titles only).
- [ ] Every dependency added, with its version, in one list.

## Report
Use the template in `ENGINEERING.md` section 12. Add the exit checklist with each item ticked or explained.