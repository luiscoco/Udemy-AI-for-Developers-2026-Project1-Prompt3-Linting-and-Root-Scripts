# Prompt 3 — ESLint 9 Flat Config & Root Scripts

This README walks through what was done in this exercise and why, step by step,
so you can reproduce (or explain) it yourself.

## Running the app (Windows Terminal)

From the repo root, in Windows Terminal (PowerShell):

```powershell
npm install
npm run dev
```

> **Note for students:** `apps/frontend` (Vite) and `apps/backend` (Fastify)
> currently only contain a `package.json` and `tsconfig.json` — there's no
> actual Vite or Fastify source code wired up yet. So right now `npm run dev`
> just runs the placeholder from [Prompt 3's `package.json`](package.json)
> (`echo "no dev server configured yet"`). Once real frontend/backend dev
> scripts exist in `apps/frontend` and `apps/backend`, the root `dev` script
> should be updated (e.g. with a tool like `turbo` or `npm-run-all`/`concurrently`)
> to run both of them together — that's a good next exercise.

## 1. Look at the repo before touching anything

Before adding any tooling, the existing structure was checked:

- Root `package.json` declares an npm **workspaces** monorepo (`apps/*`, `packages/*`).
- `apps/backend`, `apps/frontend`, and `packages/contract` are separate workspace
  packages, each with their own `package.json`.
- `tsconfig.base.json` already exists with strict TypeScript settings.
- No ESLint config or dependency existed yet.

Why start here: a flat config's `files`/`ignores` globs and the root scripts only
make sense once you know how the workspace is laid out.

## 2. Install ESLint 9 and the TypeScript plugin

```bash
npm install -D eslint@^9 typescript-eslint@^8
```

`typescript-eslint` (the unscoped package) bundles the parser, the TS plugin, and
ready-made **flat config** presets (`recommended`, etc.) in one dependency, which
is the standard way to get TypeScript linting under ESLint 9's flat config system.

## 3. Add `eslint.config.js` at the repo root

ESLint 9 uses **flat config** instead of `.eslintrc`. The config:

- Applies `@eslint/js` recommended rules + `typescript-eslint` recommended rules,
  scoped to `**/*.{ts,tsx}` only.
- Ignores `node_modules`, `dist`, `coverage`, `.turbo`, and `**/*.gen.ts`
  (generated OpenAPI types — see below).
- Configures `@typescript-eslint/no-unused-vars` to **allow unused names that
  start with `_`** (e.g. `_unusedArg`), which is a common convention for
  intentionally-ignored function parameters or destructured values.

## 4. Add root scripts to `package.json`

```json
"scripts": {
  "lint": "eslint .",
  "test": "echo \"no tests configured yet\" && exit 0",
  "build": "echo \"no build configured yet\" && exit 0",
  "dev": "echo \"no dev server configured yet\" && exit 0"
}
```

`test`, `build`, and `dev` are placeholders because this repo doesn't have a test
runner, bundler, or dev server wired up yet — they exist so CI/scripts calling
`npm run <script>` don't fail with "missing script", and can be swapped for real
commands later.

## 5. Verify the config actually works

Rather than trust the config blindly, it was exercised with throwaway files
(created, tested, then deleted):

- A `.ts` file with an unused variable named `_unused` → **no error** (allowed).
- A `.ts` file with an unused variable named `bad` → **error** (correctly caught).
- A `.gen.ts` file containing deliberately broken syntax → **skipped entirely**
  by ESLint, proving the ignore pattern works.

```bash
npm run lint
```

## Why generated files (`*.gen.ts`) are excluded from linting

Files like OpenAPI-generated TypeScript types aren't source code you own — they're
a **build artifact** derived from a spec (an OpenAPI/contract schema). That means:

- The real source of truth is the spec, not the generated file.
- Any manual edit or lint-driven "fix" to the generated file gets silently
  overwritten the next time someone re-runs the generator — the fix disappears
  and the file drifts out of sync with the spec it's supposed to represent.
- Linting generated output also means fighting the generator's own formatting
  choices, which you don't control and shouldn't need to police.

**The correct fix for a bad generated type is: fix the spec or generator input,
regenerate, and review the diff.** That's a job for a build/CI regeneration
check, not for ESLint — which is why `**/*.gen.ts` is in the `ignores` list
instead of being special-cased with rule overrides.
