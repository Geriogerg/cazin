# Project context — PvP crypto casino (step 1: interface and games)

Last updated: 6 November 2026.

## Where the project stands
I am building, as a personal project, the interface and games of a future crypto casino where players duel each other. This step covers only what players will see and play.
Out of scope for now: real money, deposits and withdrawals, crypto wallets, identity verification (KYC), licensing, anti-money laundering. I will decide later whether to launch for real, and those topics will come then.
Games are played with play chips that have no value and cannot be converted.

## Who I am and how to work with me
- I am a developer and I know online gambling and crypto. You help me build: architecture, code, game rules, interface, tests.
- Reply in English. Code, identifiers, comments and commit messages in English.
- Be demanding: tell me what is fragile, badly designed or overcomplicated.
- Separate what is certain, what is an assumption and what is your opinion.
- Do not expand on out-of-scope topics unless I ask. But flag any technical choice that would make adding real money or compliance harder later.
- When a decision changes this context, remind me to update it.

## How we work
- I implement in VS Code with Claude Code. Practices are defined in `ENGINEERING.md` (in the repo).
- Claude acts as an advisor: it decides the global architecture and the order of steps, writes each step brief (goal, scope, out of scope, exit checklist), and reviews my step reports.
- A step is validated only on raw evidence (command outputs, tree, git log, code excerpts on critical parts), never on a summary alone.
- Three documents, three roles: this context (project and decisions), `CLAUDE.md` (short rules for the coding agent), `ENGINEERING.md` (how work is done).

## Roadmap
0. Foundations: repo, tooling, CI, guardrails.
1. Provably fair module + coin flip engine.
2. Ledger + duel state machine.
3. WebSocket server.
4. Minimal UI, coin flip playable end to end.
5. Rock-paper-scissors.
6. Blackjack duel.
Visual identity and name run in parallel with step 1.

## Progress
- Step 0: not started.

## The concept
Players face each other in short duels. The platform referees, guarantees fairness and would later take a fee on each pot: it never plays against the players.
What the interface must convey: fast, readable duels; verifiable fairness; a social side (challenge friends, leaderboards, rivalries).
Target players: typical online and crypto casino players, international, mostly on mobile. To refine during the identity step.
Interface language: English only for now.

## The games
First games to build:
- Blackjack duel (flagship): both players get their cards and decide at the same time to hit or stand; closest to 21 wins; both busting is a draw. To decide: ties, shared or separate shoe, whether one opponent card is visible, value of a natural blackjack against a three-card 21.
- Coin flip: the fastest possible duel, pure chance.
- Rock-paper-scissors, best of 3: bluffing, ideal between friends.

Next games:
- Split or steal (prisoner's dilemma): each player secretly chooses to split or steal; a pool fed by double steals rewards cooperation.
- The box and the offer (final name to find): the pot is hidden in boxes, one player opens them, the other acts as banker and makes offers; always played home and away so each player is banker once.
- Rotating-bank tables: one player holds the bank, then passes it on; rotation is mandatory.
- Knockout tournaments at fixed times.

Original game ideas: last one to dare (crash duel), blind auction, shared minefield, morra.
No names or branding taken from TV shows (Golden Balls, Deal or No Deal).

## Technical principles
- The server is authoritative: it computes every outcome; the browser only displays and sends choices.
- Simultaneous moves: the server receives both moves before revealing anything.
- Fairness has two mechanisms, to be stated per game in its spec:
  - games with randomness (coin flip, blackjack): provably fair, each draw combines a server seed, whose hash is published before the game, with both players' seeds; any player can verify a game in one click;
  - games without randomness (rock-paper-scissors): fairness comes from both moves being committed before either is revealed.
- Real time over WebSocket, mobile-first interface.
- Play chips already go through a real balance ledger (double-entry bookkeeping, idempotent operations, stakes locked as soon as a duel is accepted), so real money can be plugged in later without a rewrite. The fee field exists on each pot from the start, set to 0.
- Flow rules: 15 seconds per move; 20 seconds of grace on disconnect, then an automatic move; quitting = losing. The automatic move is defined per game in its spec.
- Each game has an isolated, tested rules engine, plus a simulation of many games to check that no position has an edge.

## Decisions made
Stack:
- TypeScript everywhere: Node server, code shared with the client.
- Front-end: React + Vite, mobile-first SPA / PWA. No Next.js.
- Real time: `ws` with our own typed protocol.
- Database: PostgreSQL, local in Docker Compose. Access through Drizzle, with explicit transactions and row locks.
- Validation: zod schemas shared between client and server.
- Monorepo: pnpm workspaces (`packages/engine`, `packages/shared`, `apps/server`, `apps/web`).
- Tests: Vitest, fast-check (properties), Stryker (mutation), Playwright (end-to-end).
- Quality: ESLint + Prettier with custom rules, dependency-cruiser for architecture.
- Security tooling: gitleaks, Semgrep, `pnpm audit`, Dependabot.
- No Redis for now: single server process.
- Development environment: native Windows (WSL2 too heavy for the machine for now), Node 24 LTS, Postgres through Docker Desktop or a native install, GitHub (private repo). CI runs on Linux: line endings forced to LF, cross-platform scripts only.

Foundations:
- Amounts are integers (`bigint`, smallest unit), never floats; every ledger entry has an `asset` column (`CHIP` for now).
- Ledger and duel event log are append-only: no updates, no deletes; reversals are new entries.
- Every client action carries a client-generated ID; the server deduplicates on it.
- IDs: UUIDv7.
- The server owns all deadlines, stores them as timestamps; timers survive a restart.
- Engines are pure functions `(state, move, rng) → newState`, no I/O.
- Provably fair: HMAC-SHA256 with a nonce; server seed from `crypto.randomBytes`, hash committed before the duel; seeds encrypted at rest, revealed after settlement.
- Every protocol message has a version field.
- CI runs tests and simulations on every push.

## Open decisions
- Auth library (kept behind our own `users` table; guest + email login).
- Seed lifecycle: one server seed per duel, or per player session with a nonce counter.
- Detailed blackjack duel rules.
- Automatic move on disconnect for each game.
- Visual identity and project name.
- Target players, refined.

## Finding an opponent
This is the weak point of duels: with few players, lobbies stay empty. Rough estimate: for 90% of requests to find an opponent within 10 seconds, you need about 118 concurrent players with 18 queues (3 games × 6 stake levels), 60 with 9 queues, 21 with 3 queues.
Build into the interface:
- few stake levels (3 ranges);
- a "surprise duel" button that picks the game at random;
- asynchronous challenges that can be accepted later;
- friend challenges by link and private rooms;
- tournaments at fixed times.
Bots are allowed for testing, but always shown as bots.

## Interface to build
- Home and lobby: "Play now", open challenges, private rooms.
- A screen for each game, with its animations and a quick tutorial.
- Profile: duel history, stats, rivalries with friends.
- Leaderboards and leagues.
- Game verification (provably fair).
- Later, once the games are stable: fee shown on each pot, play limits and breaks.

## Later, out of current scope
If the project moves to real money: licence (Curaçao considered), excluded countries (including France, where online casinos are banned), identity checks, anti-money laundering, custody of funds, stablecoins, fee level, player acquisition. These are covered in the analysis documents already written, to be revisited at that point with a lawyer.