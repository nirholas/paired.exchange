# AGENTS.md

Operating notes for AI coding agents (Claude Code, Codex, Cursor, Copilot and others) working in this repository. Everything here is derived from the files actually in the tree, so trust it over guesses, and update it when the facts change.

## What this repository is

Launch a coin paired with real stocks. Permissionless, fixed supply, liquidity nobody can pull. On Robinhood Chain.  Every coin here is quoted in a tokenized equity rather than a stablecoin or the chain's gas token. A coin paired with NVDA trades against NVDA: its price, its depth, and the fees it pays its creator are all denominated in the stock.

- Homepage: https://paired.exchange
- Source: https://github.com/nirholas/paired.exchange
- Primary language: TypeScript

## Repository layout

- `apps/`
- `contracts/`
- `docs/`
- `packages/`
- `scripts/`
- `tools/`
- `README.md`
- `package.json`
- `Dockerfile`

## Setup

```bash
npm install
```

## Commands

| Task | Command |
|---|---|
| dev | `npm run dev` |
| build | `npm run build` |
| test | `npm test` |
| typecheck | `npm run typecheck` |
| build image | `docker build .` |

Run the test and lint commands above before you consider a change finished. If a command fails on code you did not touch, say so in your report instead of silently skipping it.

## Conventions

- This is a monorepo (`workspaces` in `package.json`); run scripts from the root unless a package README says otherwise.
- `.env` files are gitignored; never commit credentials, and read configuration from environment variables.
- Commit messages follow Conventional Commits (`type(scope): summary`), matching the existing history.
- Read the surrounding code before adding to it, and match its naming, file organisation and error-handling style.
- Keep `README.md` accurate: if a change alters behaviour, commands or configuration, update the docs in the same commit.
- Do not leave TODO comments, stub functions, placeholder data or commented-out code behind. Finish what you start or leave it out.
- Small, focused commits with a subject line that describes the change, not the act of committing.

## Where to raise things

- Bugs and feature requests: https://github.com/nirholas/paired.exchange/issues
- Questions and ideas: https://github.com/nirholas/paired.exchange/discussions
- Security issues: report privately at https://github.com/nirholas/paired.exchange/security/advisories/new, never in a public issue.
