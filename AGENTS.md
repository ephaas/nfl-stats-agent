# Conventions for coding agents

If a tool such as OpenSpec adds a managed block to this file, keep these conventions outside it. The tool may rewrite everything inside its block when it updates.

## Source of truth

- `PLAN.md` is the source of truth for goals, decisions, and phase order. Read it before starting work.

## Budget

- The budget is $0: free services only, with no card on file.
- Never suggest anything that needs a card, such as a paid plan or a pay-as-you-go upgrade. If a sign-up asks for a card, stop and tell the user.

## Secrets

- API keys live only in `.env.local`, which is gitignored. Never put them in code, logs, or chat.
- `.env.example` is committed, so it holds variable names only, with empty values.

## Toolchain

- Use npm (not pnpm, yarn, or bun) on Node 22.
- `web/` is Next.js 16 with strict TypeScript. Stay on TypeScript 5.x: the Next 16 template pins `typescript ^5`, and npm's `latest` tag is 7.x.
- `evals/` is Python, managed with uv.

## AI SDK v7

The AI SDK is on v7, which is ESM-only and needs Node 22 or newer. Use the v7 names ([migration guide](https://ai-sdk.dev/docs/migration-guides/migration-guide-7-0)):

- `system` is now `instructions`
- `stepCountIs` is now `isStepCount`
- `onFinish` is now `onEnd`
- `experimental_telemetry` is now `telemetry`

## Eval ground rules

- Tune on the dev set only.
- Never edit gold answers. Write gold problems to `evals/flags.md` for the user to review.
- No question-specific code: review checks the agent code (`web/src/agent/`) for question text or IDs.
- Evals pin one model and disable fallback.
- The test set is frozen. `--split test` runs only with `--final`, which is reserved for the preregistered final run in Phase 6.

## Ask first

- Ask before installing any package (npm, uv, or system-wide) and before deleting any file.
