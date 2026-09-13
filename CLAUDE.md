# AGENTS.md

This file provides guidance to AGENTS when working with code in this repository.

## Repository state

This is a fresh Bun template (created with `bun init`, Bun v1.4.2). There is no entry point yet — `index.ts` was intentionally removed — and `package.json` defines no scripts. Note that `bunx tsc --noEmit` fails with TS18003 ("No inputs were found") until at least one `.ts` file exists; that is expected, not a broken config.

## Commands

- `bun install` — install dependencies
- `bun run <file>` — run a script directly (no `bun run <script>` npm-scripts defined)
- `bun test` — run all tests; `bun test path/to/file.test.ts` for one file, `bun test -t "<name>"` for one test
- `bunx tsc --noEmit` — typecheck (see caveat above)
- `bun build <entry.ts|entry.html|entry.css>` — bundle

No linter or formatter is configured.

## TypeScript strictness (tsconfig.json)

- `strict` and `noUncheckedIndexedAccess` are on — indexing an array/map yields `T | undefined`.
- `verbatimModuleSyntax` is on — type-only imports must use `import type { ... }`.
- `allowImportingTsExtensions` is on — relative imports may include the `.ts` extension.

## Bun over Node.js

Default to using Bun instead of Node.js.

- Use `bun <file>` instead of `node <file>` or `ts-node <file>`
- Use `bun test` instead of `jest` or `vitest`
- Use `bun build <file.html|file.ts|file.css>` instead of `webpack` or `esbuild`
- Use `bun install` instead of `npm install` or `yarn install` or `pnpm install`
- Use `bun run <script>` instead of `npm run <script>` or `yarn run <script>` or `pnpm run <script>`
- Use `bunx <package> <command>` instead of `npx <package> <command>`
- Bun automatically loads .env, so don't use dotenv.

### APIs

- `Bun.serve()` supports WebSockets, HTTPS, and routes. Don't use `express`.
- `bun:sqlite` for SQLite. Don't use `better-sqlite3`.
- `Bun.redis` for Redis. Don't use `ioredis`.
- `Bun.sql` for Postgres. Don't use `pg` or `postgres.js`.
- `WebSocket` is built-in. Don't use `ws`.
- Prefer `Bun.file` over `node:fs`'s readFile/writeFile
- Bun.$`ls` instead of execa.

### Testing

Use `bun test` to run tests.

```ts#index.test.ts
import { test, expect } from "bun:test";

test("hello world", () => {
  expect(1).toBe(1);
});
```

### Frontend

Use HTML imports with `Bun.serve()`. Don't use `vite`. HTML imports fully support React, CSS, Tailwind.

Server:

```ts#index.ts
import index from "./index.html"

Bun.serve({
  routes: {
    "/": index,
    "/api/users/:id": {
      GET: (req) => {
        return new Response(JSON.stringify({ id: req.params.id }));
      },
    },
  },
  // optional websocket support
  websocket: {
    open: (ws) => {
      ws.send("Hello, world!");
    },
    message: (ws, message) => {
      ws.send(message);
    },
    close: (ws) => {
      // handle close
    }
  },
  development: {
    hmr: true,
    console: true,
  }
})
```

HTML files can import .tsx, .jsx or .js files directly and Bun's bundler will transpile & bundle automatically. `<link>` tags can point to stylesheets and Bun's CSS bundler will bundle.

```html#index.html
<html>
  <body>
    <h1>Hello, world!</h1>
    <script type="module" src="./frontend.tsx"></script>
  </body>
</html>
```

With the following `frontend.tsx`:

```tsx#frontend.tsx
import React from "react";
import { createRoot } from "react-dom/client";

// import .css files directly and it works
import './index.css';

const root = createRoot(document.body);

export default function Frontend() {
  return <h1>Hello, world!</h1>;
}

root.render(<Frontend />);
```

Then, run index.ts

```sh
bun --hot ./index.ts
```

For more information, read the Bun API docs in `node_modules/bun-types/docs/**.mdx`.

## Conventions

- Write everything the agent produces in English, regardless of the language the user speaks in the conversation (e.g., Portuguese). This covers code, code comments, documentation and any other files added to the repo, commit messages, GitHub content created via `gh` (issues, issue comments, pull request descriptions, review comments), written plans and design/spec documents produced before or during implementation, and any other generated artifact or written output. Only direct conversational replies in chat may mirror the user's language.
- Always use the long format for command arguments (e.g., `docker compose --file ... --project-name ...`, not `-f`/`-p`) — it is easier to read for someone unfamiliar with the options. This covers shell commands in general, including `RUN` statements in Dockerfiles and shell scripts such as `docker-entrypoint.sh` and `user-install.sh`. Where a tool has no long options (e.g., BusyBox utilities in Alpine-based images), short flags are fine — leave a comment saying so.
- In Docker Compose files, prefer the long syntax too: volumes as `type:`/`source:`/`target:` entries and ports as `target:`/`published:`, not the `- source:target` or `- host:container` shorthand.
- Prefer nested `.gitignore` files (e.g., `tests/.gitignore`) instead of including everything in the root `.gitignore`.
- Keep `README.md` current. Whenever the project is modified, check whether the README still reflects reality — objective, architecture, setup commands, tests — and update it in the same change whenever it does not.
- Lint, formatting and type checking are enforced by two formatters with strict ownership plus `tsc`. Biome owns the TS family (`ts`, `tsx`, `js`, `jsx`, `json`, `jsonc`) via `bun run lint` (check) and `bun run lint:fix` (auto-fix); Prettier owns markdown, YAML, shell and Dockerfile via `bun run format` (write) and `bun run format:check`, using `prettier-plugin-sh` (policy in `.prettierrc`); `bun run typecheck` runs `tsc --noEmit`. The two formatters must never own the same file — `.prettierignore` excludes the Biome-owned extensions so a repo-wide Prettier run never crosses the boundary.
- Git hooks run through Husky and activate themselves: the `prepare` lifecycle script runs on every `bun install` and points `core.hooksPath` at `.husky/_`, so a fresh clone needs zero manual setup. Escape hatches: `git commit --no-verify` (and `git push --no-verify`) skips a hook once, `HUSKY=0` disables Husky entirely. Hook logic is always written in TypeScript executed by Bun. Because Husky runs hook files through `sh` (a Bun shebang would not be honored), every extensionless hook under `.husky/` is a minimal shell shim delegating to its sibling `.ts` implementation — `exec bun "$(dirname -- "$0")/<hook>.ts" "$@"` — which is the only place Biome and `tsc` see the hooks (`tsconfig.json` and `biome.json` include `.husky/` explicitly, since `**` skips dot-directories); new hooks must follow this shape. To run subprocesses, hooks use the Bun Shell — `import { $ } from "bun";` — whenever possible, in preference to `Bun.spawn`. Four hooks exist: `pre-commit` runs lint-staged first — the fixers in `.lintstagedrc.json` (Biome and Prettier on staged files, re-staging whatever changed; blocking verifiers deliberately stay out of lint-staged because it runs glob groups concurrently and a checker racing a writer is not deterministic) — then the gates: ShellCheck on staged shell files, Compose validation when YAML is staged, and the full type check; `pre-push` runs the Docker suite's `Dockerfile` block only (`bun test --only-failures --test-name-pattern Dockerfile`) once per push — the `user-install` half depends on real vendor endpoints and fails intermittently on network hiccups, so the full suite stays a manual `bun test` run; `post-checkout` and `post-merge` rerun `bun install` unconditionally (a no-op install is sub-second, and file checkouts can also restore dependency manifests from another commit).
- Whenever information about a third-party project is needed (APIs, CLI, runtime behavior, releases, internals), research it through the `deepwiki` MCP against that project's GitHub repository (e.g., Bun via `oven-sh/bun`). The `deepwiki` skill in `.agents/skills/deepwiki/` describes the workflow.
- Shell scripts are linted with ShellCheck via `bun run lint:sh`, using the local `shellcheck` binary (see the README prerequisites); `.shellcheckrc` holds the repo-wide ShellCheck policy (bash dialect, enabled optional rules, disabled codes). Formatting goes through Prettier with `prettier-plugin-sh`, which embeds the shfmt engine — no host `shfmt` binary is needed — via `bun run format` / `bun run format:check`, with the shell policy (indent, `switchCaseIndent`, `spaceRedirects`, `binaryNextLine`) living in `.prettierrc`. The pre-commit hook runs ShellCheck on staged shell files; their formatting rides the same lint-staged Prettier task as every other Prettier-owned file.
- In shell scripts, prefer single quotes for literal strings without expansions; switch to double quotes as soon as the string contains a variable or command substitution (ShellCheck's SC2016 flags the mistake). prettier-plugin-sh preserves quote style, so this is an authoring convention enforced by review, not by tooling.