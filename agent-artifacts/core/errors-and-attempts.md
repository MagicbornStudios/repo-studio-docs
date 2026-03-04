---
title: Errors and attempts
created: 2026-02-04
updated: 2026-03-01
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Errors and attempts (do not repeat)

> **For coding agents.** See [Agent artifacts index](../../18-agent-artifacts-index.mdx) for the full list.

Log of known failures and fixes so agents and developers avoid repeating the same mistakes.

---

## Desktop packaged startup/style regression follow-up (2026-03-01)

**Problem**:
- Installed/smoke-launched builds could fail at runtime with missing Next modules (`next`, `styled-jsx`) despite successful packaging.
- Some successful launches showed largely unstyled UI because standalone static assets were not located where the standalone server served them from.
- Runtime probe sometimes could not find smoke-installed binaries when install path lived under temp smoke folders.

**Root cause**:
- Desktop dependency copy path assumed `workspaceRoot/node_modules`, which breaks under pnpm resolution layout.
- Static assets were copied only to top-level `.desktop-build/next/static`, not guaranteed standalone server-relative paths.
- Install-location resolution omitted temp smoke install directories.

**Fix**:
- `packages/repo-studio/src/desktop/build.mjs` now resolves runtime package roots via `require.resolve(..., { paths })` and copies required runtime deps from resolved locations.
- Static copy now includes standalone server-relative targets (`standalone/.next/static` and nested app fallback path), and `verify-standalone.mjs` asserts one exists.
- `install-locations.mjs` now includes `%TEMP%/RepoStudioSilentInstallSmoke` and `%TEMP%/RepoStudioInstallSmoke` in known install candidates.

**Guardrail**:
- Do not assume flat `node_modules` layout for packaged runtime dependencies.
- Desktop verifier must assert server-relative static presence, not only top-level static copy.
- Smoke/runtime probes must resolve binaries across explicit install dir, registry install dir, and known smoke-temp locations.

---

## Desktop custom header/titlebar integration (2026-03-01)

**Problem**:
- Native Windows menubar/title region remained visible and inconsistent with in-app menu UX.
- Repo Studio needed app-owned window controls (minimize/maximize/close) similar to IDE shells.

**Root cause**:
- Desktop main window used default framed BrowserWindow behavior with native title/menu surface.
- Renderer had no bridge APIs for window-control actions/state.

**Fix**:
- Desktop main process now enables custom frame mode on Windows (`frame: false`, hidden menu visibility) and exposes window IPC handlers:
  - `windowState`
  - `windowMinimize`
  - `windowToggleMaximize`
  - `windowClose`
  - `windowStateChanged` event stream.
- Preload bridge now exposes those APIs to renderer.
- Repo Studio root renders an in-app desktop header when custom-frame desktop mode is active:
  - app-owned menubar,
  - minimize/maximize/restore/close controls,
  - draggable region with no-drag control/menu subregions.

**Guardrail**:
- Desktop-only UI controls must be behind runtime bridge detection.
- Keep framed-window assumptions out of renderer; all window control goes through typed IPC bridge.

---

## Desktop reclaim child-process lineage safety (2026-02-28)

**Problem**: Reclaim/cleanup needed to handle child processes (Next server, codex app-server, spawned terminals) without broad process-name matching that could kill unrelated user workloads.

**Root cause**: Process inventory records did not carry parent PID metadata, so reclaim planning could not reason about process lineage and had to rely on command-line markers + known ports.

**Fix**:
- Added `parentPid` collection in process snapshots:
  - Windows: `Win32_Process` now includes `ParentProcessId`.
  - Posix: `ps` snapshot now includes `ppid`.
- Reclaim planning now computes descendant PID sets from verified RepoStudio/Codex roots (tracked PID + safe-port/ownership checks).
- Child processes are reclaimed only when lineage to a verified root exists (`repo-studio-child` reason), not by process-name alone.

**Guardrail**:
- Do not add kill-by-name-only reclaim rules for `node/electron/powershell/bash`.
- Keep lineage + ownership checks as the primary reclaim contract.

---

## Repo Studio folder-as-workspace expectation (2026-02-26)

**Problem**: Repo Studio behavior drifted from expected editor UX. The app exposed workspace/panel actions but did not expose a first-class File menu open-folder workflow, and push-to-GitHub was not surfaced from File menu with auth-aware gating. Several runtime surfaces still resolved data from host workspace assumptions.

**User correction**:
- Repo Studio must behave like VSCode/Cursor: `File > Open Project Folder...` sets active workspace folder.
- Viewport/workspace data should target the opened folder.
- Forge tooling should remain provided by Repo Studio host, not required inside opened folders.
- `File > Push to GitHub` should exist and be enabled only when signed in and the active folder is a git repo.

**Fix**:
- Added File menu actions (`Open Project Folder...`, `Recent Projects`, gated `Push to GitHub`) through `menu-contributions.ts`.
- Added desktop native folder picker IPC bridge and web input-dialog fallback.
- Relaxed project import from git-only to any existing directory; compute `isGitRepo` dynamically in project records/API payloads.
- Hard-cut data root resolution to active project root across repo-studio workspace loaders/routes.
- Split root semantics explicitly:
  - host workspace root for tool binary resolution,
  - active project root for runtime data and command cwd.

**Guardrail**:
- Do not reintroduce host-root fallback for workspace data paths.
- Keep tool resolution host-rooted while executing against active project cwd.
- Keep File menu Open Folder + Push affordances visible and deterministic.

---

## Repo Studio extension layout drift + Story hard requirement (2026-02-26)

**Problem**: Extension panel layout was still partially hardcoded in `workspace-catalog.ts` (`workspaceKind === "story"` branch), and guardrails/tests still assumed Story manifest must exist locally in this repo.

**User correction**:
- Story is optional/community-distributed per project.
- Extensions must live in the active project folder and be installable from a registry source.
- Host should not hardcode Story layout specs when extension specs are available.

**Fix (phase 14/15 cutover)**:
- Added extension layout resolution priority: extension payload layout -> generated extension layout getter -> generic fallback.
- Removed story-specific catalog fallback branch.
- Added extension registry APIs (`/api/repo/extensions/registry`, `/install`, `/remove`) and project-scoped install flow.
- Added built-in Extensions workspace for browse/install/update/remove.
- Added registry examples feed (`examples/studios/*`) as browse-only reference cards.
- Removed Story-specific install prompt from root shell; extension discovery/install is Extensions-workspace-driven.

**Guardrail**:
- Do not reintroduce story-specific layout hardcoding in workspace catalog.
- Do not treat studio example ids as installable extension targets.
- Keep extension execution host-rendered by workspaceKind; no arbitrary extension JS loading.

---

## Repo Studio product hard cut verification pitfalls (2026-02-26)

**Problem**: During the product hard-cut slice (core workspace reduction + global assistant/terminal), `next build` failed with a hook-rule error that was easy to miss because webpack cache warnings flooded logs.

**Root cause**:
- `AssistantPanel` quick-prompt handler was named `useQuickPrompt`, so eslint interpreted it as an invalid hook call inside a callback context.
- Build output contained heavy webpack cache noise (`PackFileCacheStrategy` warnings), which obscured the actionable lint line.

**Fix**:
- Renamed callback to `handleQuickPrompt` and updated button handlers.
- Captured build logs to file and tailed the final section to isolate lint/type failures from cache warnings.

**Guardrail**:
- Do not prefix non-hook callbacks with `use`.
- When Next build logs are noisy, capture output and inspect tail/error sections before changing unrelated code.

---
## Twick undo-redo: localStorage is not defined (Studio SSR) â€” OBSOLETE

**Obsolete (2026-02-24)**: Twick was fully removed from the repo. This entry is kept for historical reference only.

**Problem**: Studio terminal showed `Failed to load undo-redo state from localStorage: [ReferenceError: localStorage is not defined]`. Twick's `UndoRedoProvider` (timeline) used a persistence key and loaded/saved state from localStorage.

**Guardrail (still valid)**: Any vendored or shared code that touches localStorage in a path that can run during SSR or in Node must use `window.localStorage` and check `typeof window !== "undefined" && typeof window.localStorage !== "undefined"` before use.

---

## Repo Studio predev blocked by Codex login state (`pnpm dev:repo-studio`)

**Problem**: Repo Studio dev startup failed before Next server start when Codex CLI was installed but not logged in (`codex login: not-logged-in`). `predev:repo-studio` ran `forge-repo-studio doctor` and exited non-zero.

**Root cause**: Doctor readiness required `codex_chatgpt_login` for default `ok=true`, coupling runtime/dependency health with user auth state.

**Fix (2026-02-22)**:
- Keep Codex CLI installation as a hard gate, but make login non-blocking in default doctor mode.
- Add strict auth mode only when explicitly requested: `forge-repo-studio doctor --require-codex-login`.
- Bundle `@openai/codex` in `@forge/repo-studio` and prefer bundled invocation by default.
- Add UI-first sign-in path through Repo Studio (`POST /api/repo/codex/login`, Codex Setup card actions).

**Guardrail**:
- `pnpm dev:repo-studio` must fail-fast on dependency/runtime breakage, but must not fail solely because Codex login is missing unless strict mode is explicitly enabled.

---

## Repo Studio assistant/runtime split drift + GitHub CLI auth coupling (2026-02-25)

**Problem**: Repo Studio assistant UX regressed into split assistant surfaces and inconsistent model behavior, while GitHub auth in app bar depended on CLI-style status/login semantics.

**Root cause**:
- Separate `loop-assistant` and `codex-assistant` surfaces created duplicated state and routing ambiguity.
- Model selection in Repo Studio was not fully runtime-scoped/API-backed.
- GitHub sign-in path was tied to CLI-style readiness, causing disabled/confusing app-bar behavior.

**Fix**:
- Hard-cut to one assistant panel/workspace id: `assistant`, with runtime switching inside the panel (`forge` or `codex`).
- Added runtime model catalog APIs:
  - `GET /api/repo/models?runtime=forge|codex`
  - `POST /api/repo/models/selection`
- Added codex `model/list` cache+warn fallback (`.repo-studio/codex-model-cache.json`) so codex catalog failures do not hard-block chat.
- Replaced GitHub login flow with OAuth device-flow routes:
  - `POST /api/repo/github/oauth/device/start`
  - `POST /api/repo/github/oauth/device/poll`
  - `POST /api/repo/github/logout`
  - `GET /api/repo/github/status`
- Rewired app bar controls to start/poll OAuth and show connected state from persisted integration status, not CLI presence.

**Guardrail**:
- Repo Studio must keep one assistant panel with runtime-scoped model catalogs.
- GitHub auth UX must remain OAuth/API-driven and never be disabled due missing CLI tooling.

---

## Repo Studio viewport-as-canvas correction (2026-02-26)

**Problem**: Repo Studio treated `viewport` as an explicit dock tab in the main rail. This produced a redundant â€œViewportâ€ tab header and blocked the expected VS Code-style center canvas behavior, especially in Story where the user expected a left explorer and closable page tabs in the center.

**Root cause**:
- `WorkspaceLayout.Main` did not support `hideTabBar`, so only left/right rails could hide outer Dockview headers.
- Planning/Code/Story rendered `viewport` as a normal toggleable main panel tab.
- Story still used a monolithic single-tab panel (`StoryPanel`) instead of explorer + per-page tabs.
- Panel toggles/store state still allowed `viewport` hidden states to leak through menus/settings/persistence paths.

**Fix**:
- Added `hideTabBar` support to `WorkspaceLayout.Main` and applied rail-header hiding for main panels.
- Updated Planning/Code/Story to use `WorkspaceLayout.Main hideTabBar` while keeping `WorkspaceLayout.Panel id=\"viewport\"` for stable layout IDs/codegen.
- Hard-pinned `viewport` visibility:
  - filtered viewport out of View > Layout hide/show items,
  - filtered viewport out of Settings > Panels toggles,
  - blocked hide attempts in store actions and migration sanitization.
- Refactored Story workspace to two surfaces:
  - left `story` panel (`StoryExplorerPanel`) for outline/scope/create/publish,
  - center viewport tabs (`StoryPagePanel`) for per-page editor+reader with one tab per path.
- Added closable inner Story tabs with:
  - empty canvas support (`allowEmpty` + custom empty state),
  - dirty-close interception (`onBeforeCloseTab` confirm/discard),
  - per-loop tab persistence keying (`${layoutId}::loop::${loopId}`),
  - tree-refresh pruning for removed paths while preserving drafts for still-open tabs.

**Guardrail**:
- Treat `viewport` as canvas, not as a user-toggleable panel.
- For viewport workspaces, hide outer main tab headers and keep close behavior on inner viewport tabs only.
- Story must preserve left explorer + center closable page tabs with empty-canvas allowed.

---

## Repo Studio no-legacy contract drift (2026-02-26)

**Problem**: Legacy assistant ids/config keys and fallback surfaces (`loop-assistant`, `codex-assistant`, `defaultEditor`, `assistant.routes.loop`, JSON proposal fallback, `--legacy-ui`) remained in active runtime/codegen/CLI paths, causing repeated regressions and mixed contracts.

**Fix (hard cut)**:
- Removed assistant-id aliases in runtime and codegen outputs; canonical targets are now only `forge` and `codex`.
- Renamed config/settings contract to `defaultTarget`, `targets`, `assistant.routes.forge`, `assistant.prompts.forgeAssistant`.
- Switched repo-studio shell persist key to clean-state v2 with no legacy migration path.
- Removed JSON proposal fallback/migration path and enforced SQLite-only proposal store with explicit `503` (`SQLITE_PROPOSALS_UNAVAILABLE`) API responses.
- Removed obsolete compatibility files/routes (`LoopAssistantWorkspace`, `CodexAssistantWorkspace`, legacy GitHub login route).
- Removed package runtime `--legacy-ui` and package UI docs tab; package UI tabs are planning/env/commands/assistant.

**Guardrail**:
- Do not add alias normalization or fallback paths for removed assistant/config/proposal/CLI contracts.
- Any stale caller/config using removed tokens is invalid and must be updated to canonical values.

---

## Repo Studio: Payload "Is X column created or renamed?" prompt in dev (2026-02-26)

**Problem**: When running Repo Studio in dev, Payload CMS (collections in dev mode) prompted interactively: "Is assistant_target column in repo_proposals table created or renamed from another column?" after renaming a collection field (e.g. `editor_target` â†’ `assistant_target`). This blocked non-interactive dev and required one-off migration scripts.

**Root cause**: Payload sees schema drift (old column gone, new column present) and asks whether to treat it as a rename or a create. We do not want to rely on persistent dev DB data or answer prompts.

**Fix (2026-02-26)**:
- Dev runs `db:reset-dev` before starting Next + Drizzle Studio. That deletes the Repo Studio SQLite file (same path as `payload.config.ts`), so Payload always starts with no DB and creates tables from current collection configâ€”no prompt.
- Removed one-off script `scripts/migrate-assistant-target.mjs` and `db:migrate:assistant-target`. Standalone wipe: `pnpm --filter @forge/repo-studio-app db:reset-dev`.
- Documented in [decisions.md](./decisions.md): Repo Studio dev DB is ephemeral; Payload owns schema; Drizzle is pull-only.

**Guardrail**: Do not add one-off Payload column-migration scripts for dev. If you need to rename a collection field, the dev workflow is: run dev (which wipes and recreates). For production data, use Payload migrations or a separate process.

---

## Repo Studio dev blocked by orphaned ports (`EADDRINUSE` on 3010/related ports)

**Problem**: `pnpm dev:repo-studio` failed with `listen EADDRINUSE` because orphaned dev processes were still holding RepoStudio/Codex ports.

**Root cause**: cleanup was fragmented (`stop` only tracked runtime + unguarded untracked fallback by port), and there was no explicit repo-scoped reclaim workflow for orphan processes.

**Fix (2026-02-23)**:
- Added manual reclaim commands:
  - `forge-repo-studio processes [--scope repo-studio|repo]`
  - `forge-repo-studio reclaim [--scope repo-studio|repo] [--dry-run] [--force]`
- Safe default scope (`repo-studio`) targets only RepoStudio/Codex-owned processes and known ports.
- Force scope (`repo`) requires `--force` for actual kills and supports repo-wide cleanup of repo-owned runtime processes.
- Added ownership guard to `forge-repo-studio stop` fallback: do not kill foreign untracked port owners.
- Added ANSI-rich CLI reporting with machine-safe `--json` and plain fallback (`--plain` / `NO_COLOR`).

**Guardrails**:
- `pnpm dev:repo-studio` stays non-destructive by default.
- Use `reclaim --dry-run` first, then `reclaim` (safe) or `reclaim --scope repo --force` (explicitly destructive).

---

## RepoStudio CSS import dependency drift (`tw-animate-css`, `tailwindcss-animate`)

**Problem**: RepoStudio can fail at build/dev with:

- `Module not found: Can't resolve 'tw-animate-css'`
- `Module not found: Can't resolve 'tailwindcss-animate'`

when `apps/repo-studio/app/globals.css` imports packages not installed in `apps/repo-studio/package.json`.

**Fix (2026-02)**: Repo Studio uses only `tailwindcss-animate` via `@plugin "tailwindcss-animate"`. The `tw-animate-css` package was removed because its CSS export resolution fails with Next.js/webpack (package exports conditional `"style"` not resolved). `tailwindcss-animate` is Tailwind v4 compatible and sufficient for animation utilities.

- `apps/repo-studio/app/globals.css`: no `@import "tw-animate-css"`; only `@plugin "tailwindcss-animate"`.
- `apps/repo-studio/package.json`: no `tw-animate-css` dependency.
- `forge-repo-studio doctor` checks only `tailwindcss-animate`.

**Guardrails now expected**:

- `forge-loop verify-work` runs `pnpm --filter @forge/repo-studio-app build` for RepoStudio file changes.
- `forge-repo-studio doctor` checks resolvability of `tailwindcss-animate`.
- `pnpm dev:repo-studio` runs a doctor precheck before starting Next dev.

**Do not**: Add CSS `@import` to app globals without updating that same app's dependencies and running the app-local build.

---

## Repo Studio unstyled UI from missing Tailwind/PostCSS runtime transform

**Problem**: Repo Studio rendered as browser-default HTML controls (unstyled buttons/inputs/cards). Live CSS payloads contained raw Tailwind directives (`@theme`, `@apply`) and lacked generated utility classes.

**Root cause**: `apps/repo-studio` was missing local Tailwind v4 PostCSS wiring (`postcss.config.*`) and `@tailwindcss/postcss` resolution, so Next served untransformed CSS for app globals.

**Fix (2026-02-23)**:
- Added app-local PostCSS config: `apps/repo-studio/postcss.config.mjs` with `@tailwindcss/postcss`.
- Added app dependencies: `@tailwindcss/postcss`, `postcss`, `postcss-load-config`.
- Extended dependency health contracts (app + package) with:
  - `postcssConfigResolved`
  - `tailwindPostcssResolved`
  - `tailwindPipelineResolved`
- Made doctor/predev fail-fast on style pipeline breakage by gating `depsOk` on those fields.
- Added regression coverage:
  - `packages/repo-studio/src/__tests__/dependency-health.test.mjs` (missing config/dependency cases)
  - `apps/repo-studio/__tests__/styles/tailwind-pipeline.test.mjs` (compile globals.css, assert no raw directives).

**Guardrail**:
- Repo Studio startup must block when Tailwind/PostCSS compiler wiring is broken. Run `pnpm forge-repo-studio doctor` before `pnpm dev:repo-studio`, and revalidate with `pnpm --filter @forge/repo-studio-app test:styles`.

---

## Agent behavior: user correction overrides agent assumptions

**Rule**: When the user explicitly tells an agent that a hypothesis or fix is wrong, the agent must stop believing and acting on it. User correction overrides external documentation, web search results, or prior agent conclusions.

**Do**:
- Note the correction in errors-and-attempts and do not repeat the forbidden approach.
- Push back once with evidence if the agent has strong contrary data; then defer to the user.
- Update docs (AGENTS.md, errors-and-attempts, STATUS) so future agents find the correction.

**Do not**:
- Re-implement a fix the user forbade.
- Assume stack-trace location equals root cause when the user says otherwise.
- Be stubborn: correct once, then stop.

**Example**: Radix composeRefs / ScrollArea replacement was identified as a fix; user forbade it and confirmed it was a false positive. Existing entry: "Radix composeRefs hypothesis (FALSE POSITIVE)".

---

## Universal padding/margin reset â€” CRITICAL: never add to `*`

**Problem**: A universal selector `* { padding: 0; margin: 0 }` (or equivalent in `@layer base` or CSS reset) strips default padding and margin from **all** elements â€” buttons, labels, inputs, badges. This caused days of debugging: buttons and labels looked cramped everywhere, even with explicit utility classes. Tailwind preflight already normalizes; adding our own `*` reset stacks and breaks UI.

**Fix**:
- **Do NOT add** `padding: 0` or `margin: 0` to the universal selector `*` in `globals.css` or any base layer.
- Keep only intended `*` rules (e.g. `border-border`, `outline-ring/50`). Do not add box-model resets to `*`.
- If components still look cramped, add explicit padding via utilities or design tokens (`--control-padding-x`, `--panel-padding`, etc.), or adjust component styles â€” never strip globally.
- Before changing base/globals CSS, read this entry and [styling-and-ui-consistency.md](styling-and-ui-consistency.md) Â§ No universal padding/margin reset.

**Documentation**: See `apps/studio/app/globals.css` comment above `@layer base` and styling-and-ui-consistency rule 26.

---

## Shiki: `yarn` language not in bundle (fumadocs-mdx code blocks)

**Problem**: Code blocks with ` ```yarn ` in MDX docs cause build failure: `ShikiError: Language 'yarn' is not included in this bundle`. Shiki (via fumadocs-mdx) does not ship the Yarn Spinner grammar.

**Fix**: Use ` ```text ` for Yarn Spinner format examples, or another supported language (e.g. `yaml` if the snippet is YAML-like). Do not use `yarn` as a code block language.

---

## MDX agent blocks: use Callout, not details

**Problem**: Raw HTML `<details><summary>â€¦</summary>â€¦</details>` in MDX can cause build error: `Unexpected character ! (U+0021) before name`. Fumadocs/MDX parsers can choke on details content.

**Fix**: Use `<Callout variant="note" title="For coding agents">` instead. Callout is registered in mdx-components and parses correctly.

**Do not**: Use raw `<details>` or `<summary>` for agent blocks; use Callout. See [docs-building Â§ Agent blocks](docs-building.md).

---

## QuickNav inline items: acorn parse error

**Problem**: `<QuickNav items={[{ label: "X", href: "#x" }, ...]} />` with inline array/object in MDX causes "Could not parse expression with acorn" in fumadocs-mdx.

**Fix**: Use `export const quickNavItems = [...];` in the MDX body, then `<QuickNav items={quickNavItems} />`. Or omit QuickNav until needed.

---

## Heading slug syntax: use `[#slug]`, not `{#slug}`

**Problem**: Custom heading IDs like `## Title {#slug}` cause "Could not parse expression with acorn" in fumadocs-mdx. MDX parses `{...}` as a JS expression; `#slug` is invalid in that context.

**Fix**: Use Fumadocs syntax `[#slug]` (square brackets): `## Title [#slug]`. Do not use `{#slug}` (curly braces).

---

## Fumadocs `meta.json` schema mismatch: `pages` must be strings only

**Problem**: Platform docs build failed with `[MDX] invalid data` when `apps/platform/content/docs/meta.json` (or nested `meta.json`) used object entries in `pages` (for example `{ title, url, icon }`). Current Fumadocs schema in this repo accepts string targets only.

**Fix**:

- Keep every `pages` entry as a string path or separator marker (`---Section---`).
- Do not store per-page metadata objects in `pages`; put page metadata in MDX frontmatter instead.
- Run `pnpm docs:platform:doctor` after nav/meta edits to catch schema and target-existence regressions before `build:platform`.

**Do not**:

- Reintroduce object nodes into any docs `meta.json` `pages` array.
- Skip the doctor check when changing docs nav, generated component docs, or aliases.

---

## Docs component flat-route 404 (`/docs/components/<slug>`)

**Problem**: Generated component docs are grouped under category folders (for example `/docs/components/atoms/accordion`), but flat URLs like `/docs/components/accordion` returned 404 when no explicit alias existed.

**Fix**:

- Added router-level category resolution in `apps/platform/src/app/docs/[[...slug]]/page.tsx` to map flat component slugs to canonical generated paths.
- Resolution precedence is deterministic: atoms -> editor -> assistant-ui -> tool-ui.

**Do not**:

- Assume every component doc lives directly under `/docs/components/*`; generated docs are nested by category by design.
- Remove the flat-route resolver without adding equivalent aliases/redirect coverage.

---

## Platform component showcase fallback regression (`ComponentDemo` placeholders)

**Problem**: Component docs pages rendered fallback text (`A custom runtime preview is not registered...`) instead of real component previews when demo IDs were missing from the runtime registry.

**Fix**:

- Make demo ids contract-generated and typed:
  - `scripts/generate-platform-component-docs.mjs` emits `apps/platform/src/components/docs/component-demos/generated-ids.ts`
  - exports `COMPONENT_DEMO_IDS` and `ComponentDemoId`.
- Make runtime strict:
  - `ComponentDemo` accepts `id: ComponentDemoId` (not `string`).
  - Missing renderer throws hard error; no permissive prefix/text fallback rendering.
- Build demo map from generated ids:
  - `apps/platform/src/components/docs/component-demos/index.tsx` resolves from `COMPONENT_DEMO_IDS`.
- Enforce with doctor:
  - `scripts/platform-docs-doctor.mjs` verifies docs `<ComponentDemo id="..."/>` coverage against generated ids and fails if placeholder fallback code patterns are reintroduced.

**Do not**:

- Add/keep placeholder fallback UI for missing component demos.
- Use untyped `string` ids for `ComponentDemo`.
- Skip `pnpm docs:platform:generate` + `pnpm docs:platform:doctor` after component docs changes.

---

## Payload type generation: collection imports must be Node-resolvable

**Problem**: `pnpm payload:types` failed when collection files imported helpers through non-resolvable runtime paths (for example extension-less relative imports or `@/` aliases from files loaded directly by Payload's Node process).

**Fix**:

- In collection files loaded by `payload.config`, use runtime-resolvable imports (relative path + `.ts` extension where needed).
- Avoid `server-only` guards in helper modules that are imported by collections during type generation.
- Keep app-router-only guards in route handlers/components, not in payload-schema dependency paths.

---

## Local development 403s on protected Payload REST routes (unauthenticated session)

**Problem**: Local Studio development hit repeated 403s from protected collection routes (for example `/api/forge-graphs`, `/api/relationships`) because the browser had no authenticated Payload session while access control correctly required authenticated ownership/membership.

**Fix**:

- Added `LocalDevAuthGate` (`apps/studio/components/providers/LocalDevAuthGate.tsx`) and wrapped it early in `AppProviders` before auth-dependent providers.
- Gate flow:
  - `GET /api/me` with credentials.
  - If unauthenticated, `POST /api/users/login` with local admin credentials.
  - Invalidate `authKeys.me()` and render app after session bootstrap.
- Added feature-flag guard `isLocalDevAutoAdminEnabled()` (`apps/studio/lib/feature-flags.ts`) so this only runs in `NODE_ENV=development` on `localhost` / `127.0.0.1` and when `NEXT_PUBLIC_LOCAL_DEV_AUTO_ADMIN !== '0'`.
- Added local env docs in `apps/studio/.env.example` for:
  - `NEXT_PUBLIC_LOCAL_DEV_AUTO_ADMIN`
  - `NEXT_PUBLIC_LOCAL_DEV_AUTO_ADMIN_EMAIL`
  - `NEXT_PUBLIC_LOCAL_DEV_AUTO_ADMIN_PASSWORD`

**Do not**: Relax collection ACLs globally for local development. Keep production access policy unchanged and use a real auth session bootstrap for local-only convenience.

---

## Env drift across Studio/Platform (`.env.example` vs `.env.local` vs runtime)

**Problem**: Manual env maintenance caused onboarding failures and confusing runtime errors:

- `.env.example` drifted from actual required keys.
- Local `.env.local` files were missing or incomplete.
- Runtime routes failed without clear remediation when critical keys were absent.

**Fix**:

- Added manifest-driven env source of truth in `scripts/env/manifest.mjs`.
- Added generated examples:
  - `pnpm env:sync:examples`
  - `pnpm env:sync:examples:check`
- Added local setup: **env portal** (`pnpm env:portal`) and **env bootstrap** (wired into `dev:studio`/`dev:platform`); when keys are missing, portal opens automatically.
- Added CLI fallback: `pnpm env:setup -- --app studio|platform|all`
- Added env doctor (masked matrix compare, optional Vercel pull):
  - `pnpm env:doctor -- --app ... --mode ... [--vercel]`
- Added Vercel sync: `pnpm env:sync:vercel`; portal has "Sync to Vercel" (config in `.env.vercel.local`).
- Centralized runtime env readers:
  - `apps/studio/lib/env.ts`
  - `apps/platform/src/lib/env.ts`

**Do not**:

- Hand-edit generated `.env.example` files.
- Add new env keys without updating `scripts/env/manifest.mjs` and running `pnpm env:sync:examples`.
- Print plaintext secret values in diagnostics.
- Bypass env checks by adding permissive runtime fallbacks outside local dev contexts.

---

## LangGraph chat rollout: missing transport headers should fall back to legacy path

**Problem**: LangGraph orchestration in `/api/assistant-chat` depends on editor metadata headers (`x-forge-ai-domain`, `x-forge-ai-editor-id`, `x-forge-ai-project-id`, `x-forge-ai-viewport-id`). When project/domain headers are absent or invalid, session lookup/checkpoint persistence can fail or produce poor context.

**Fix**:

- Keep `/api/assistant-chat` as a single endpoint with explicit `legacyStreamPath` and `langGraphPath`.
- Gate LangGraph with `AI_LANGGRAPH_ENABLED`.
- In LangGraph path, require valid domain + project header before orchestrating; otherwise return `null` addendum and continue legacy stream behavior.
- Always keep the legacy stream fallback in a catch block to avoid user-facing chat outages during rollout.

**Do not**:

- Hard-fail chat requests when LangGraph metadata is missing.
- Remove the legacy fallback until metadata wiring is guaranteed in all editor surfaces.

---

## Dialogue assistant (Studio) returning standard response instead of Open Router

**Problem**: The Dialogue assistant in Studio (Dialogue editor) can appear to return a standard or canned response instead of streaming real model output from Open Router. The assistant must use the same AI infrastructure (Open Router) and stream real responses; any fallback to a non-Open-Router path or a generic message is unacceptable.

**Trace (review when debugging)**:
- **Client**: `DialogueAssistantPanel` uses `API_ROUTES.ASSISTANT_CHAT` and `transportHeaders` from `DialogueEditor` (`x-forge-ai-domain`, `x-forge-ai-editor-id`, `x-forge-ai-viewport-id`, `x-forge-ai-project-id` when LangGraph enabled). Transport is `AssistantChatTransport` from `@assistant-ui/react-ai-sdk`; requests go to Studioâ€™s `/api/assistant-chat`.
- **Server**: `apps/studio/app/api/assistant-chat/route.ts` uses `getOpenRouterConfig()` (module load; requires `OPENROUTER_API_KEY`), `requireAiRequestAuth` (401 if missing), model from `getPersistedModelIdForProvider(req, 'assistantUi')` and `resolveModelIdFromRegistry(selectedModel, registry)`. If registry is empty or model invalid, fallback may apply. Then `streamAssistantResponse` (OpenAI SDK + `config.baseUrl`) streams. Early returns: 503 if no API key, 401 if unauth, 400 if invalid JSON.

**Things to check when the assistant does not use Open Router**:
- Env: `OPENROUTER_API_KEY` and `OPENROUTER_BASE_URL` (e.g. `pnpm env:setup --app studio`). If the key is missing at module load, the route returns 503 and the client may show a generic error or placeholder.
- Auth: Unauthorized (401) causes a JSON error response; the client might surface it as a â€œstandardâ€ message.
- Model resolution: Empty OpenRouter registry or invalid persisted model ID can trigger fallback; ensure `resolveModelIdFromRegistry` returns a valid model and that the model list loads (`GET /api/model-settings`).
- Do not bypass the Open Router path for chat; do not return a non-stream JSON body for normal chat requests when auth and config are valid.

**Guardrail**:
- When changing **assistant panels** (e.g. Repo Studio chat-only UI, tooltips, shared `Thread` or transport), do **not** change Studioâ€™s Dialogue assistant transport URL, headers, or provider wiring unless the change is explicitly to fix Open Router. Repo Studio uses a different app and route; keep `API_ROUTES.ASSISTANT_CHAT` and `DialogueAssistantPanel` wiring unchanged for Studio.
- After any change that touches shared assistant UI or the assistant-chat route, **verify** that the Studio Dialogue assistant still streams real model responses (e.g. open Dialogue editor, send a message, confirm streamed reply from the selected model).

---

## Dialogue assistant 401 (Unauthorized) with seeded admin and auto-login

**Problem**: Dialogue assistant returned 401 even when Studio seeds an admin and LocalDevAuthGate auto-logs in. The client might show a generic error instead of streaming.

**Root cause**: (1) AssistantChatTransport did not send cookies by default. (2) No fallback when session/API key was missing in local dev.

**Fix (2026-02-23)**:
- In `DialogueAssistantPanel`, set `credentials: 'include'` on `AssistantChatTransport`.
- In `/api/assistant-chat`, when `requireAiRequestAuth` returns null and `isLocalDevAutoAdminEnabled()` is true, look up the seeded admin by `getLocalDevAutoAdminCredentials().email` and create a synthetic `AiRequestAuthContext` so the request proceeds.

**Guardrail**: Do not remove the local-dev bypass without providing another path for headless/local testing.

---

## Studio CSS import failure: `Can't resolve '@forge/shared/styles/editor'`

**Problem**: Studio build/dev failed while evaluating `apps/studio/app/globals.css` because `@forge/shared/styles/editor` resolved to `packages/shared/dist/styles/editor.css`, but that file is generated only when `@forge/shared` build runs.

**Fix**: In Studio globals, import Dockview CSS and shared Dockview overrides directly:

- `@import "dockview/dist/styles/dockview.css";`
- `@import "../../../packages/shared/src/shared/styles/dockview-overrides.css";`

This removes the local prebuild dependency on `packages/shared/dist/styles/editor.css` for Studio app startup.

**Do not**: Reintroduce package CSS imports in Studio that require prebuilt `dist` artifacts unless dev scripts guarantee those artifacts are generated first.

---

## Dialogue assistant runtime `ReferenceError`: `ForgePlanExecuteProvider is not defined`

**Problem**: `apps/studio/components/editors/dialogue/DialogueAssistantPanel.tsx` referenced `ForgePlanExecuteProvider` and `ForgePlanToolUI` in JSX without importing them, causing runtime crash when the panel rendered with `executePlan`.

**Fix**: Import both symbols from `@forge/domain-forge/assistant` in `DialogueAssistantPanel.tsx`:

- `import { ForgePlanExecuteProvider, ForgePlanToolUI } from '@forge/domain-forge/assistant';`

**Do not**: Assume re-export availability implies runtime scope. JSX symbols must be explicitly imported in each file where they are referenced.

---

## Model switcher popover appears detached at far-left in composer

**Problem**: In Studio chat composer, `ModelSwitcher` popover could render detached at the far-left of the page instead of near the trigger.

**Fix**: In `apps/studio/components/model-switcher/ModelSwitcher.tsx`, render popover content through portal for composer variant and set explicit placement:

- `portalled` enabled
- `side="top"` for composer
- `align="end"`
- `collisionPadding={12}`

**Do not**: Rely on non-portal popover placement inside sticky/complex container stacks for composer overlays.

---

## Studio bar alignment regressions should be fixed in `Studio.tsx` composition first

**Problem**: Studio top-bar alignment drift (menubar/project switcher/tabs/actions) can look like a shared tab-group issue, but the regression is often caused by how `StudioApp.Tabs` slots are composed in `apps/studio/components/Studio.tsx`.

**Fix**:

- Treat this as a Studio-only composition fix first: adjust `StudioApp.Tabs.Left/Main/Right` placement in `Studio.tsx`.
- Ensure `WorkspaceTabGroup` enforces real regions: root `w-full`, tablist `flex-1 min-w-0`, actions `shrink-0 ml-auto`; otherwise all controls can visually collapse to the left despite slot naming.
- Keep shared tab-group spacing tweaks intact unless there is a proven shared component defect.
- Prefer menu-level editor-open actions (File -> Editors submenu) over adding more top-bar quick-open buttons when space is tight.

**Do not**: Overcorrect `packages/shared/src/shared/components/workspace/WorkspaceTabGroup.tsx` for a Studio-only layout regression.

---

## Studio settings opened from menubar lacked a symmetric close action

**Problem**: Opening settings from the top menubar could leave users with no obvious "close from menu" path, creating the impression that Settings was stuck open.

**Fix**:

- In `apps/studio/lib/settings/useAppSettingsMenuItems.tsx`, menu behavior now toggles:
  - `Open Settings` when neither settings surface is open.
  - `Close Settings` when the sidebar/sheet is open.
- Close action explicitly sets both `settingsSidebarOpen` and `appSettingsSheetOpen` false to avoid stale UI state.
- In `apps/studio/components/AppProviders.tsx`, desktop settings rail now includes:
  - Header close button (`X`) to close immediately.
  - Click-away backdrop behind the sidebar to close on outside click.
  - Escape key handler while sidebar is open.

**Do not**: Provide one-way "Open Settings" menu actions without a matching close/toggle path in Studio chrome.

---

## Repo Studio UI load: troubleshooting when "hasn't loaded due to errors"

**Context**: User reports Repo Studio UI has not loaded for a while due to errors.

**Investigation steps**:
1. **Build**: `pnpm --filter @forge/repo-studio-app build` â€” if it passes, compile is fine.
2. **Port**: `pnpm dev:repo-studio` uses 3010; `EADDRINUSE` means another process has it. Stop conflicting process or use different port.
3. **Browser console**: If page loads but breaks, capture console errors (hydration, dockview, missing provider).
4. **Dockview CSS**: Repo Studio uses Dockview; ensure `dockview.css` and overrides load. See Studio CSS import failure entry for pattern.
5. **Bootstrap**: `forge-repo-studio run` or `open` may need to be running; config in `.repo-studio/`.

**Gap**: Root cause TBD. Document findings in repo_studio_analysis GAPS Â§ UI Load / Runtime and here when identified.

---

## Repo Studio layout defaults: declarative code IS the default â€” do not ask

**Rule**: The declarative code in the codebase (e.g. `PlanningWorkspace`, `EditorDockLayout` children) **is** the default layout. There is no separate config or manifest that defines "the default." Reset layout restores what's in code. Codegen may produce artifacts from that code (e.g. for reset), but the source of truth is the code.

**Do not**: Ask "should default layout be in code or config?" or "code vs manifest for layout defaults." The answer is always: code. The code the user sees is the default.

**Reference**: repo_studio_analysis DECISIONS-WORKSPACE-PANELS Â§ Layout and Flexibility.

---

## Platform `@forge/ui` adapter pitfall: root import can break Next server build

**Problem**: Replacing platform local atom files with direct re-exports from `@forge/ui` root (for example `export { Button } from '@forge/ui'`) caused `next build` failures in platform with errors like:

- `You're importing a component that needs createContext/useEffect/useState...`

The failure originates from `packages/ui/dist/index.mjs` being treated as a server import path that bundles mixed client-hook exports.

**Fix**:

- Keep platform atom wrappers local (`apps/platform/src/components/ui/*`) unless `@forge/ui` exposes stable subpath exports that preserve client boundaries.
- Do not switch wrappers to `@forge/ui` root re-exports until package export strategy is updated.
- For now, converge by policy + phased cleanup, not by root re-export shortcuts.

**Guardrail**: If platform build errors originate from `packages/ui/dist/index.mjs` after adapter changes, revert wrapper re-export approach and keep local atoms.

---

## Dock slot content must be reactive (Inspector / slot-dependent UI)

**Problem**: Selecting a node in one graph did not show the selection in the Inspector until the user focused another panel (e.g. the other graph). State (selection, activeScope) updated correctly, but the right-panel (slot) content did not re-render because Dockview controls when `SlotPanel` is rendered; when only React context changed (new `right={inspectorContent}` from the editor), Dockview did not re-render the panel.

**Fix**: Use a **slot content store** (Zustand) so slot content drives re-renders independently of Dockview. In `EditorDockLayout.tsx`: (1) Create a module-level store holding `slots: Record<string, React.ReactNode>` and a `version` that bumps on each write. (2) In `DockLayoutRoot`, sync the resolved context value (left, main, right, rightInspector, rightSettings, bottom) into the store in `useLayoutEffect` when it changes. (3) In `SlotPanel`, subscribe to the store by `slotId` (e.g. `useSlotContentStore(s => ({ content: s.slots[slotId], version: s.version }))`) and render the stored content, with context as fallback. Then when the editor passes new `right={inspectorContent}`, the store updates and `SlotPanel` re-renders without requiring focus elsewhere. Ensure editor slot content (e.g. Inspector) is not over-memoizedâ€”dependency arrays should include `activeSelection`, `inspectorSections`, etc. When using a selector that returns an object or array, wrap with **useShallow** to avoid getSnapshot infinite loop (see next entry).

---

## Zustand: getSnapshot infinite loop when selector returns new object/array

**Problem**: Using a Zustand store with a selector that returns a new object or array every time (e.g. `(s) => [s.foo, s.bar] as const` or `(s) => ({ a: s.a })`) causes React to log "The result of getSnapshot should be cached to avoid an infinite loop" and can cause an infinite render loop. useSyncExternalStore treats any new reference as a change.

**Fix**: Use `useShallow` from `zustand/shallow` to wrap the selector so the result is shallow-compared; only when the shallow comparison fails will the component re-render. Alternatively, select only primitives (e.g. `s.slots[id]` plus a separate selector for `s.version` if both are needed, or a single primitive that changes when either changes).

---

## Hydration mismatch: invalid interactive nesting (`button` descendants)

**Problem**: React/Next hydration failed with errors like `In HTML, <button> cannot be a descendant of <button>`. In this repo, two concrete cases were found:

- `apps/studio/components/settings/SettingsPanel.tsx`: toggle row rendered a native `<button>` wrapper containing `Switch` (Radix button).
- `packages/ui/src/components/ui/dropzone.tsx`: dropzone root used `Button` (renders `<button>`) and nested `<input>`.

**Fix**:

- Replace outer native button wrappers with non-button containers when they must contain interactive controls.
- Preserve row-click behavior with container `onClick` and `stopPropagation` on inner controls (Reset/Switch).
- For non-form clickable surfaces (e.g. dropzone), use a styled `div` root (`buttonVariants`) + dropzone root props instead of a button root when an `<input>` child is required.

**Guardrail**:

- Added `scripts/hydration-nesting-doctor.mjs` and root script `pnpm hydration:doctor`.
- Root `lint` now runs `pnpm hydration:doctor` before Studio lint.

---

## Assistant UI tooltip runtime crash in docs previews (`TooltipProvider` missing)

**Problem**: Rendering assistant-ui surfaces in docs component previews (e.g. `CodebaseAgentStrategyEditor`) crashed with:

- `` `Tooltip` must be used within `TooltipProvider` ``

**Cause**: `TooltipIconButton` is used throughout assistant-ui thread controls, and docs previews can render outside the main app provider stack, so no tooltip context exists.

**Fix**:

- Wrap `CodebaseAgentStrategyEditor` with shared `AppProviders` (includes tooltip provider by default).
- Wrap docs `ComponentPreview` body with `AppProviders` to harden all preview examples rendered outside app-shell providers.

References:

- `packages/shared/src/shared/components/assistant-ui/CodebaseAgentStrategyEditor.tsx`
- `apps/studio/components/docs/ComponentPreview.tsx`

---

## Settings inner tabs (AI / Appearance / Panels / Other) not updating until focus on graph

**Problem**: In the Settings panel (right dock panel â†’ Settings tab), switching between the inner tabs (AI, Appearance, Panels, Other) did not update the tab highlight or content until the user focused the **graph panel** (not the library or other areas). The tab state lived inside `AppSettingsPanelContent`; when it updated, only that subtree re-rendered. The right slot content is provided to Dockview via context; `SlotPanel` only re-renders when the context value (the `right` element) changes. That value is created by the layout parent (e.g. DialogueEditor). So when only `AppSettingsPanelContent`'s state changed, the parent did not re-render, the context value did not change, and the slot did not re-renderâ€”so the tab visual did not update until something else (e.g. focusing the graph) caused the layout parent to re-render and refresh the slot.

**Preferred fix**: **Universal Settings Sidebar (right rail).** Use a single Settings Sidebar at app shell level (shadcn `Sidebar` with `side="right"`), containing `AppSettingsPanelContent`. When Settings is the root of its own surface (sidebar or a dedicated dock panel), inner tab state updates re-render correctly. AppShell wraps content in `SidebarProvider` and renders the Settings Sidebar with `activeEditorId`, `activeProjectId`, and `settingsViewportId` from the store; editors set `setSettingsViewportId(viewportId)` when their viewport/scope changes so the Panels tab shows the right context (e.g. narrative vs storylet). "Open Settings" opens the sidebar. The right dock is Inspector-only (single `right` panel). If the right dock showed empty ("nothing is rendering"), the cause was often a saved layout with panel id `right` while the app only provided `right-inspector`/`right-settings` content; reverting to a single `right` panel and moving Settings to the sidebar fixes both.

**Alternative (legacy) fix**: **Lift the inner tab state** to the parent that owns the slot content so that when the user changes the inner tab, that parent re-renders and the slot context value updates. (1) In the editor, add state for the Settings inner category (e.g. `settingsInnerCategory` / `setSettingsInnerCategory`). (2) Pass it to `AppSettingsPanelContent` as a controlled prop (e.g. `controlledCategory={{ appUserCategory, onAppUserCategoryChange }}`). (3) In `AppSettingsPanelContent`, support optional `controlledCategory`; when provided, use it instead of internal state. (4) Include the inner category in the **key** of the right-panel content so the slot content identity changes and Dockview/SlotPanel receive a fresh tree. (5) Keep `key={value}` on the Radix Tabs in SettingsTabs as a secondary aid. When the component is used outside the dock (e.g. in AppSettingsSheet), omit `controlledCategory` so it keeps using internal state.

**Wrong fix (do not repeat):** (1) Fixing the **Inspector/Settings** two-tab strip instead of the **Settings inner tabs**. (2) Adding only `key={value}` to the Radix `Tabs` in SettingsTabsâ€”that alone does not fix it when the slot content is rendered by Dockview and does not re-render on inner state change.

---

## Zustand: select derived value for reactivity, not the getter

**Problem**: Toggling a setting in Settings (e.g. "Show minimap" in Panels) did not update the UI (e.g. graph minimap) until the user switched panels or refocused. The component was subscribing to the store with `useSettingsStore((s) => s.getSettingValue)` and then calling `getSettingValue(key, ids)` in render. The selector returns the same function reference when the store updates, so Zustand does not re-render when the underlying state (e.g. `viewportSettings`) changes.

**Fix**: Select the **derived value** so the component subscribes to the slice that actually changes. For example: `useSettingsStore((s) => (s.getSettingValue('graph.showMiniMap', { editorId, viewportId }) as boolean | undefined) ?? true)`. Then when that key in the store changes, the selector result changes and the component re-renders. Use the same pattern for any setting that must stay in sync across Settings and consumers (e.g. graph viewport settings).

---

## Logging: use Studio logger, no ad-hoc console

**Guidance:** Do not add ad-hoc `console.log` / `console.warn` / `console.error` for model routing or API flows. Use the Studio logger: `getLogger('namespace')` from `apps/studio/lib/logger.ts` with the appropriate level and structured fields so logs can be turned off via `LOG_LEVEL` or inspected in the log file. Client-only code (e.g. model-router store) uses `clientLogger` from `@/lib/logger-client` when logging to the server is enabled.

---

## Marketing Hero: do not use placeholder YouTube IDs

**Problem**: Hero "Watch Demo" previously opened a dialog with a placeholder YouTube embed (generic or test ID), which could confuse users or look unprofessional.

**Fix**: Do not embed real YouTube URLs in Hero until we have a real product demo asset. "Watch Demo" links to `/demo`; the hero product preview block links to `/roadmap` ("See what we're building"). When a real demo video exists, wire it in `HeroBlock` or `HeroVideoDialog` and remove the /demo redirect. See [HeroBlock.tsx](../../apps/marketing/components/sections/HeroBlock.tsx).

---

## Grey buttons and missing menu/icons

**Problem**: Toolbar and menu items appeared flat grey and text-only; the AI intent card had tight padding and looked flush to borders.

**Fix**: (1) Use `outline` (or `default`) for File menu trigger and primary toolbar actions so they are not ghost-only. (2) Add optional `icon` to `WorkspaceMenubarItem` and `WorkspaceFileMenuItem`, and pass icons from DialogueEditor/CharacterEditor for View, State, and File menu items. (3) Add icons to editor creation buttons and Workbench in AppShell/DialogueEditor. (4) Use `p-[var(--panel-padding)]` (and content padding) in cards (e.g. AgentWorkflowPanel). Design docs 01 and 02 updated with icon/padding rules and screenshot reference ([docs/design/01-styling-and-theming.mdx](../../design/01-styling-and-theming.mdx), [02-components.mdx](../../design/02-components.mdx)).

---

## Input icons and badge padding (overlap + cramped tags)

**Problem**: Search inputs and command palettes showed oversized icons that overlapped text; tag/badge pills (e.g. model provider + v2 chips) appeared cramped with no padding.

**Fix**: Use `--icon-size` for input/command icons and calculate left padding with `pl-[calc(var(--control-padding-x)+var(--icon-size)+var(--control-gap))]` to ensure text never collides. Tokenize command/menu/select padding to `--control-*` and `--menu-*`. For tags, rely on `--badge-padding-x/y` in `Badge` and avoid overriding with ad-hoc `px-*`/`h-*`. See [styling-and-ui-consistency.md](./styling-and-ui-consistency.md) and [design/01-styling-and-theming.mdx](../../design/01-styling-and-theming.mdx).

---

## Fumadocs docs runtime on Next 15.5.9 (`useEffectEvent` + provider mismatch)

**Problem**: Studio docs hit runtime failures when using full Fumadocs runtime components:

- `You need to wrap your application inside FrameworkProvider.`
- `useEffectEvent is not a function` from Fumadocs page/client modules.

**Cause**: On this stack (`next@15.5.9`), importing `fumadocs-ui/page` (and related runtime providers) can pull client paths that rely on `useEffectEvent`, which is not available in the Turbopack-resolved runtime path.

**Current working pattern (2026-02-10)**:

- Use **headless Fumadocs source** (`source.getPageTree()` + `source.serializePageTree`) with the shared docs shell (`DocsLayoutShell`) for Studio docs chrome.
- Keep `fumadocs-ui/page` and `fumadocs-ui/layouts/docs` out of Studio docs route files.
- Keep route-scoped stylesheet import (`@import "fumadocs-ui/style.css";`) in `apps/studio/app/docs/docs.css` for MDX component styling.
- Keep token bridge (`--color-fd-*` -> Studio/shadcn semantic tokens) and custom MDX overrides (`fumadocs-ui/mdx` + Forge blocks).

**Guardrail**:

- Added `pnpm docs:runtime:doctor` (wired into `pnpm docs:doctor`) to fail if Studio docs reintroduces runtime Fumadocs layout/page imports.
- If switching to `fumadocs-ui/css/preset.css`, re-run full CSS/build verification first; this repo previously hit Tailwind utility resolution issues on that path.

References:

- `apps/studio/app/docs/layout.tsx`
- `apps/studio/app/docs/[[...slug]]/page.tsx`
- `apps/studio/app/docs/DocsShell.tsx`
- `apps/studio/app/docs/mdx-components.tsx`
- `apps/studio/app/docs/docs.css`
- `scripts/docs-runtime-doctor.mjs`

---

## Legacy custom docs shell felt raw (missing top tabs, weak sidebar, weak TOC)

**Problem**: Even with the stable custom renderer, docs still looked unfinished: no strong top navigation tabs, sparse sidebar hierarchy, and low-contrast TOC/typography.

**Fix**: Upgrade the shared docs shell primitives instead of switching runtimes:

- `DocsLayoutShell`: sticky top tabs, brand header, mobile sidebar trigger, GitHub link, better content width/padding, sidebar inset offset.
- `DocsSidebar`: section icons (Lucide), collapsible folder groups, search input, stronger active/hover styling, and compact hierarchy spacing.
- `RightToc`: stronger "On this page" rail, indentation by heading depth, active hash highlighting.
- Docs article typography tuned in studio/marketing page renderers (headers, spacing, tables, links, blockquotes).

References:

- `packages/shared/src/shared/components/docs/DocsLayoutShell.tsx`
- `packages/shared/src/shared/components/docs/DocsSidebar.tsx`
- `packages/shared/src/shared/components/docs/RightToc.tsx`
- `apps/studio/app/docs/[[...slug]]/page.tsx`
- `apps/marketing/app/(marketing)/docs/[[...slug]]/page.tsx`

---

## Legacy custom docs shell parity regressions (generic labels, weak TOC tracking, sidebar/header overlap)

**Problem**: Docs styling rendered, but quality still regressed vs common docs templates:

- Sidebar/top tabs showed fallback labels like `Doc` / `Section`.
- Right TOC only updated on `hashchange` (not while scrolling).
- Sidebar content was clipped under the sticky header (desktop overlap).

**Cause**:

- Fumadocs page-tree names are `ReactNode`; our label extraction accepted only string/number.
- TOC state depended only on `window.location.hash` events.
- Sidebar is fixed-position (`inset-y-0`) and needed explicit top offset below header in this shell.

**Fix**:

- Added shared label helpers in `packages/shared/src/shared/components/docs/tree-label.ts`:
  - `toPlainText(node, fallback)` handles ReactNode arrays/elements/`dangerouslySetInnerHTML`.
  - `toTitleFromHref(href, fallback)` provides deterministic fallback labels.
- Updated docs shell/sidebar to use the shared helpers and remove generic label fallback dependency:
  - `packages/shared/src/shared/components/docs/DocsLayoutShell.tsx`
  - `packages/shared/src/shared/components/docs/DocsSidebar.tsx`
- Added shared header height var and desktop sidebar offset (`--docs-header-height`) so sidebar starts below header.
- Upgraded TOC to scroll-aware active tracking via `IntersectionObserver`, with an active rail thumb:
  - `packages/shared/src/shared/components/docs/RightToc.tsx`
- Added docs-shell regression tests and a single command guardrail:
  - `apps/studio/__tests__/docs/*`
  - `pnpm docs:doctor` (runs css doctor + docs tests)

---

## Legacy custom docs shell still felt cramped (left overlap, search icon collision, weak tree depth)

**Problem**: After parity fixes, docs still had UX issues in real usage:

- Main docs content/header could still feel overlapped by the fixed left sidebar.
- Sidebar search icon could collide with input text.
- Folder nesting looked flat (section-like) instead of file-tree hierarchy.
- Markdown tables/readability felt weak for long docs pages.

**Cause**:

- The shell relied on inset `margin-left` for sidebar offset; fixed sidebar + inset sizing still produced cramped behavior in some layouts.
- Sidebar search input left padding was too tight for the leading icon.
- Folder headers were styled as uppercase section labels with subtle nesting rails.
- Article typography/table styles were duplicated per app and not strong enough.

**Fix**:

- Move docs-shell left offset to provider padding (`md:pl-[var(--sidebar-width)]`) and keep sidebar fixed under header (`md:top`, `md:bottom-auto`, computed height).
- Improve top tab overflow handling/truncation to reduce header squeeze.
- Increase search input icon spacing (`pl-9`) and keep icon placement stable.
- Strengthen sidebar hierarchy styling:
  - normal-case folder labels,
  - stronger nested rails/indent,
  - branch connectors for nested items.
- Introduce shared docs article class preset:
  - `packages/shared/src/shared/components/docs/doc-content.ts` (`DOCS_ARTICLE_CLASS`),
  - used by both Studio and Marketing docs page renderers.
- Render fallback body content when `page.data.body` is not a function component (prevents missing content on mixed markdown/MDX shapes).

References:

- `packages/shared/src/shared/components/docs/DocsLayoutShell.tsx`
- `packages/shared/src/shared/components/docs/DocsSidebar.tsx`
- `packages/shared/src/shared/components/docs/RightToc.tsx`
- `packages/shared/src/shared/components/docs/doc-content.ts`
- `apps/studio/app/docs/[[...slug]]/page.tsx`
- `apps/marketing/app/(marketing)/docs/[[...slug]]/page.tsx`

---

## Docs article link hover scope + docs rails visibility fallback

**Problem**:

- Hovering the docs main article caused **all links** to underline at once.
- In some desktop-width sessions, docs header tabs / left sidebar / right TOC could appear missing, making the shell feel broken.

**Cause**:

- `DOCS_ARTICLE_CLASS` used `hover:[&_a]:underline`, which scopes underline to the parent hover state, not each anchor.
- Docs rail visibility relied only on utility breakpoint variants; when those conditions regressed in-session, rails stayed hidden.

**Fix**:

- Change link hover selector to per-anchor: `[_a:hover]:underline`.
- Add explicit docs-shell fallback media rules in `apps/studio/app/docs/docs.css`:
  - at `>=48rem`: force desktop sidebar container and top tabs visible,
  - at `>=64rem`: force right TOC visible.
- Add stable hook classes for fallback targeting:
  - `forge-docs-tabs` in `DocsLayoutShell`.
  - `forge-docs-toc` in `RightToc`.

References:

- `packages/shared/src/shared/components/docs/doc-content.ts`
- `packages/shared/src/shared/components/docs/DocsLayoutShell.tsx`
- `packages/shared/src/shared/components/docs/RightToc.tsx`
- `apps/studio/app/docs/docs.css`

---

## Docs sidebar overlay/clipping and TOC hierarchy spacing polish

**Problem**:

- Desktop docs sidebar could overlap content instead of reliably displacing main content.
- Sidebar top section could appear clipped behind the sticky header.
- TOC looked cramped (no clear inner margin rhythm) and hierarchy connector lines sat too close to numbered headings.

**Cause**:

- Desktop displacement/top offset depended on utility variants only; when that path regressed, sidebar stayed fixed but content offset/top sizing did not fully apply.
- TOC container/link rhythm and connector geometry were too tight for numbered headings.

**Fix**:

- Harden desktop docs layout in route CSS (`apps/studio/app/docs/docs.css`) with explicit media-query fallbacks:
  - force provider left padding (`padding-left: var(--sidebar-width)`),
  - force sidebar top/height (`top: var(--docs-header-height)`, `height: calc(100svh - var(--docs-header-height))`, `bottom: auto`),
  - keep desktop rails visible as fallback.
- Increase docs header height token to `4rem` in `DocsLayoutShell` to prevent top clipping.
- Improve TOC spacing/legibility in `RightToc`:
  - add padded rounded TOC container,
  - add more link padding/vertical rhythm,
  - shift and shorten hierarchy connector lines so they donâ€™t crowd numbered text.

References:

- `apps/studio/app/docs/docs.css`
- `packages/shared/src/shared/components/docs/DocsLayoutShell.tsx`
- `packages/shared/src/shared/components/docs/RightToc.tsx`

---

## Better Auth-style TOC spacing parity (explicit gutters + rail geometry)

**Problem**:

- Even after initial TOC polish, links could still look visually flush against the TOC container edge.
- TOC hierarchy rail/connector spacing still looked tighter than the target Better Auth pattern.

**Cause**:

- TOC spacing depended mostly on utility class composition; visual rhythm remained too tight for this docs aesthetic.
- Rail and connector offsets were not tuned as a single geometry system (container padding, rail x-position, link padding, connector length).

**Fix**:

- Added explicit route-scoped TOC classes in `apps/studio/app/docs/docs.css` for deterministic spacing:
  - `.forge-docs-toc-inner` (outer gutters),
  - `.forge-docs-toc-nav` (padded inner card),
  - `.forge-docs-toc-nav::before` (vertical rail geometry),
  - `.forge-docs-toc-link` (consistent link padding),
  - `.forge-docs-toc-thumb` (active indicator aligned to rail).
- Updated `RightToc` markup to use these classes and tuned depth/connector geometry:
  - width `w-72`,
  - depth step `14px`,
  - connectors shifted left and lengthened so numbers/text have breathing room.

References:

- `packages/shared/src/shared/components/docs/RightToc.tsx`
- `apps/studio/app/docs/docs.css`

---

## Docs sidebar directory styling, truncation, and ultra-slim scroll behavior

**Problem**:

- Sidebar structure felt flat and not file-directory-like.
- Long doc titles could visually overflow/crowd the sidebar.
- Sidebar scrollbar looked heavy vs modern docs UIs.

**Cause**:

- Sidebar rows/icons were generic and depth spacing was shallow.
- Truncation depended on partial `truncate` usage without consistent `min-w-0`/overflow constraints on all row variants.
- Default scroll-area scrollbar treatment remained too visible for docs navigation.

**Fix**:

- Updated docs sidebar tree rendering to a directory pattern:
  - folder rows use left chevron + folder open/closed icons,
  - file rows use file-oriented icons (with API/reference pages getting a code-file icon),
  - deeper indentation and clearer branch line rhythm (Cursor-like tree feel).
- Hardened truncation in both folder and file rows:
  - `min-w-0` + `overflow-hidden` containers,
  - `truncate` on labels,
  - `title` attributes for full-name hover reveal.
- Added route-scoped scrollbar styling for docs sidebar:
  - native viewport scrollbar hidden,
  - Radix scroll thumb reduced to an ultra-slim 4px rail that fades in on hover.

References:

- `packages/shared/src/shared/components/docs/DocsSidebar.tsx`
- `apps/studio/app/docs/docs.css`

---

## Legacy custom docs sidebar trigger not opening (no client boundary)

**Problem**: Docs sidebar icon was visible, but clicking it did nothing; the left sidebar and right TOC never appeared to hydrate.

**Cause**: `DocsLayoutShell` (client-only, uses hooks like `usePathname`) was imported directly into a **server** `page.tsx` from the bundled `@forge/shared` entrypoint. Because the bundle does not carry a module-level `"use client"` directive, Next treated the import as server code and skipped client hydration for the docs shell.

**Fix**: Wrap `DocsLayoutShell` in a local client component and use that in the docs pages, so the docs shell is in a client boundary. Example: `apps/studio/app/docs/DocsShell.tsx` and `apps/marketing/app/(marketing)/docs/DocsShell.tsx`, then render `<DocsShell ...>` from the server page. Also ensure Tailwind `@source` includes `packages/shared` in marketing so docs shell classes compile.

---

## Tailwind v4 arbitrary CSS variable values output invalid CSS

**Problem**: Layout and popover sizing broke (docs sidebar width, Radix popover/menu/select sizing, tooltip/menubar/hover-card origin). Utilities like `w-[--sidebar-width]` and `origin-[--radix-*-transform-origin]` generated invalid CSS (`width: --sidebar-width`), so styles were effectively ignored.

**Cause**: Tailwind v4 no longer auto-wraps `--var` in `var(...)` for arbitrary values.

**Fix**: Replace `[...]` values that reference CSS variables with explicit `var(--...)`, e.g. `w-[var(--sidebar-width)]`, `max-h-[var(--radix-select-content-available-height)]`, and `origin-[var(--radix-dropdown-menu-content-transform-origin)]`. Updated shared UI components (`@forge/ui`) plus marketing and shared components that used Radix CSS vars.

---

## Docs styles partially missing (Tailwind v4 partial utility generation)

**Problem**: Docs rendered with only partial styling: some classes worked (e.g. typography/arbitrary values), but layout-critical utilities such as `px-4`, `h-14`, `md:hidden`, `lg:px-10`, `max-w-4xl`, and docs shell offset utilities were missing from compiled CSS.

**Cause**: `apps/studio/app/globals.css` and `apps/marketing/app/globals.css` still used legacy `@tailwind base/components/utilities` directives in a Tailwind v4 setup. In this repo pipeline, that could compile a partial utility set and break docs shell layout. Marketing also had `@import` ordering drift (imports after Tailwind directives), making failures harder to diagnose.

**Fix**:

- Migrate app globals to Tailwind v4 canonical import: `@import "tailwindcss";`.
- Keep all CSS `@import` rules grouped at the top of each globals file.
- Add strict diagnostics script: `scripts/css-doctor.mjs` and root scripts `css:doctor`, `css:doctor:studio`, `css:doctor:marketing`.
- `css:doctor` validates:
  - no legacy `@tailwind` directives in globals;
  - no `@import` ordering violations;
  - generated utility probes for docs layout classes.

References:

- `apps/studio/app/globals.css`
- `apps/marketing/app/globals.css`
- `scripts/css-doctor.mjs`
- `package.json`

---

## Styling: theme variables overridden by globals

**Problem**: Semantic tokens (`--background`, `--foreground`, etc.) were defined in both `packages/shared/src/shared/styles/themes.css` (via `--color-df-*`) and `apps/studio/app/globals.css` (`:root` / `.dark` with raw oklch). Import order caused globals to override the theme.

**Fix**: Removed the duplicate `:root` and `.dark` blocks from `apps/studio/app/globals.css`. Themes.css is the single source for semantic tokens. Set default theme with `data-theme="dark-fantasy"` on `<html>` in `apps/studio/app/layout.tsx`.

---

## Theme/surface tokens (contrast in sidebar and app bar)

**Problem**: Using `bg-background` or `text-foreground` inside a different surface (e.g. sidebar, app bar strip) can produce wrong contrast (e.g. dark text on dark background), because those tokens are global.

**Fix**: Use surface-specific tokens. Inside sidebar use `bg-sidebar` / `text-sidebar-foreground` (and `sidebar-accent` for hover). For popovers use `bg-popover` / `text-popover-foreground`; for cards/panels use `bg-card` / `text-card-foreground`. Do not use `bg-background` or `text-foreground` in non-body surfaces. See [01 - Styling and theming â€” Token system](../../design/01-styling-and-theming.mdx#token-system-layers-and-when-to-use-which).

---

## Toolbar buttons not switching with theme

**Problem**: When switching app theme (e.g. Dark Fantasy to Cyberpunk), some toolbar controls (e.g. project switcher trigger "Demo Project", ghost toolbar buttons) did not update and appeared stuck with a light or default look.

**Cause**: The ghost button variant in `packages/ui` had only hover styles (`hover:bg-accent hover:text-accent-foreground`) and no base `bg-*` or `text-*`, so the default state inherited browser/parent styling, which is not driven by theme CSS variables.

**Fix**: (1) In `packages/ui/src/components/ui/button.tsx`, give the ghost variant an explicit default state: `bg-transparent text-foreground` so the default state is theme-driven. (2) For toolbar triggers, prefer `outline` (or `default`) per design rules; ProjectSwitcher compact trigger was changed to `variant="outline"` to match other app bar buttons. See [styling-and-ui-consistency.md](./styling-and-ui-consistency.md).

---

## Model routing: server-state only (single chat provider)

**Design**: One slot â€” `assistantUiModelId` â€” in in-memory server-state for chat. **No model ID in request**: assistant-chat handler uses only `getAssistantUiModelId()`; do not send `x-forge-model` or body `modelName`. Client updates model only via POST to `/api/model-settings` with `{ provider: 'assistantUi', modelId }`. **ModelSwitcher** with `provider="assistantUi"`. Default from code via `getDefaultChatModelId()`. Use naming `assistantUi` (type `ModelProviderId`). See [03-model-routing-openrouter.mdx](../../architecture/03-model-routing-openrouter.mdx).

---

## Model 429 / 5xx (custom cooldown removed)

**Problem**: Repeated requests to a rate-limited or failing model can exhaust the provider.

**Current fix**: Assistant-chat uses a **single model** from server-state (no fallbacks). Task routes (forge/plan, structured-output) use `DEFAULT_TASK_MODEL`. No app-level health or cooldown. If we reintroduce fallbacks later, use OpenRouter's `models: [primary, ...fallbacks]` in the request body.

---

## Model switcher registry empty (OpenRouter models not loading)

**Problem**: ModelSwitcher showed "Models load from OpenRouter when available. Check API key if empty" even when the API returned a full list. The dropdown never showed the OpenRouter model list.

**Cause**: The model router store's `fetchSettings` called `GET /api/model-settings` (which returns `registry` from `getOpenRouterModels()`) but only updated `activeModelId`, `mode`, `fallbackIds`, `enabledModelIds`, `manualModelId` in state. It never set `registry`, so the client kept the initial `registry: []`.

**Fix**: In `apps/studio/lib/model-router/store.ts`, in `fetchSettings`, set `registry` (and `copilotModelId`, `assistantUiModelId`) from the GET response so the UI receives the OpenRouter list and current slot values.

---

## Settings: single source of truth (tree-as-source)

**Problem:** Previously, adding a new setting required touching both (1) defaults (config/schema) and (2) section definitions (APP_SETTINGS_SECTIONS, etc.). Duplicate definitions led to drift and missing controls.

**Fix (current):** Settings use **tree-as-source**: the React tree (SettingsSection + SettingsField with `default`) is the single source. Codegen (`pnpm settings:generate`) mounts the tree, reads the registry, and writes `lib/settings/generated/defaults.ts`. Config and store use generated defaults; the panel uses only registry sections. **Do not** add keys in a hand-maintained schema or duplicate defaults; add or edit SettingsField in the tree and run codegen. See [settings-tree-as-source-and-codegen.mdx](../../architecture/settings-tree-as-source-and-codegen.mdx) and decisions.md ADR "Settings: tree-as-source and generated contract."

---

## Project switcher at editor level (avoid)

**Context**: Project context is **app-level**. The app-shell store holds `activeProjectId`; `ProjectSwitcher` lives in AppShell (editor tab bar). Dialogue and Character editors sync from app shell into their domain stores.

**Do not**: Add a project switcher or project-selection UI inside an individual editor. Use the app-level project; see [decisions.md](./decisions.md) (Project context at app level).

---

## Double-registering system actions

**Problem**: If multiple editors are mounted at once and each calls `useDomainCopilot`, the same action names (e.g. `forge_createNode`) can be registered twice, leading to undefined behavior.

**Fix**: Only the **active** editor is rendered. App Shell renders only DialogueEditor or VideoEditor, so only one system contract is active at a time.

---

## Payload type generation failing via CLI

**Problem**: Running `payload generate:types` directly can fail with ESM/require errors when loading `payload.config.ts` on Node 24.

**Fix**: Use `pnpm payload:types`, which calls `node scripts/generate-payload-types.mjs`. This script loads collections via `tsx/esm/api`, builds config, and writes to `packages/types/src/payload-types.ts`.

---

## pnpm blocked by PowerShell execution policy

**Problem**: PowerShell can block `pnpm.ps1` with "running scripts is disabled" errors.

**Fix**: Run via `cmd /c pnpm <command>` (or use a shell where script execution is allowed).

---

## Payload dynamicImport warning during build

**Problem**: Next build may warn about `payload/dist/utilities/dynamicImport.js` using an expression for dynamic import.

**Fix**: This warning is expected from Payload and does not block the build. Treat it as informational unless it turns into a hard error.

---

---

## Raw fetch for API routes

**Problem**: Using raw `fetch` for collection or app API bypasses the Payload SDK / generated client and duplicates logic; it breaks type safety and makes it easy to drift from the contract.

**Fix**: For collection CRUD (forge-graphs, video-docs), use the Payload SDK (`lib/api-client/payload-sdk.ts`) via the TanStack Query hooks. For custom endpoints (auth, settings, AI), use the generated services or **manual client modules** in `lib/api-client/` (e.g. elevenlabs, media, workflows), or **vendor SDKs** where appropriate. Do not call `fetch` for `/api/*` from components or stores. **New endpoints:** add a manual client module in `lib/api-client/` or use a vendor SDK. The OpenAPI spec is for **documentation only**; OpenAPI client generators do not support streaming responsesâ€”use manual modules (e.g. workflows.ts) for SSE/streaming endpoints. Do not rely on extending the spec to generate new clients.

## Duplicating collection CRUD in custom routes

**Problem**: Adding custom Next routes that reimplement Payload collection CRUD (e.g. `/api/graphs` that just calls `payload.find`/`create`/`update`) duplicates Payload's built-in REST and is unnecessary.

**Fix**: Use Payload's auto-generated REST (`/api/forge-graphs`, `/api/video-docs`) and the Payload SDK from the client. Add custom routes only for app-specific behavior (e.g. scope-based settings upsert, AI, SSE).

---

## Vendor CLI: npx is not available (ElevenLabs, etc.)

**Problem**: Running `npx @elevenlabs/cli@latest components add audio-player` (or similar) can fail with "Error: npx is not available. Please install Node.js/npm." even when `npx --version` works in the same shell. The CLI often spawns a subprocess that doesn't inherit PATH correctly (e.g. on Windows or with pnpm).

**Fix**: Use **`pnpm dlx @elevenlabs/cli@latest components add <component-name>`** from the directory that has `components.json` (e.g. `cd packages/ui` for shared). If that still fails, install the CLI globally: `npm i -g @elevenlabs/cli`, then run `elevenlabs components add <name>` from that directory.

---

## Twick CSS/JS resolution (build) â€” OBSOLETE

**Obsolete (2026-02-24)**: Twick was fully removed. Kept for historical reference.

---

## Twick: useTimelineContext must be used within a TimelineProvider (VideoEditor) â€” OBSOLETE

**Obsolete (2026-02-24)**: Twick was fully removed. Video editor remains capability-locked; when re-enabled, use a new timeline/editor stack.

---


## Lost panels / Reset layout (DockLayout)

**Problem**: Users can end up with an empty content area (all panels gone). Dockview can persist a broken layout if all panels are closed or moved.

**Fix**: DockLayout uses **Dockview** and exposes a **ref with `resetLayout()`**. Call `ref.current.resetLayout()` to clear persisted layout and restore default panels (e.g. from a "Reset layout" or "Restore default layout" button in the editor toolbar or settings). Layout is stored at `localStorage['dockview-{layoutId}']`; clearing that key and remounting restores defaults.

---

## Panel hide not reclaiming space (View > Layout > Hide Library)

**Problem**: Hiding a panel (e.g. Library) via View > Layout only hid the **content**; the panel slot stayed in the Dockview layout, so the space was not reclaimed and adjacent panels did not expand.

**Fix (Option B)**: DockLayout has an effect that syncs panel visibility to the Dockview API. When `resolvedSlots[slotId]` becomes null (panel content hidden), we call `panel.api.close()` to remove the panel from the layout. When content is restored, we re-add the panel with `api.addPanel()` and the correct position. This dynamically mutates the layout instead of only changing slot content. See `packages/shared/.../EditorDockLayout.tsx` useEffect "Sync panel visibility".

**View menu label sync**: When the user closes a panel via the Dockview tab X (instead of View > Hide), our settings store was not updated, so the View menu still showed "Hide" instead of "Show". Fix: DockLayout now accepts `onPanelClosed?(slotId)`. Editors pass a callback that calls `setPanelVisible(spec.key, false)` for the removed panel. DockLayout subscribes to `api.onDidRemovePanel` and invokes the callback.

---

## DockLayout runtime crash: `referencePanel 'main' does not exist`

**Problem**: Dialogue/Character could crash at runtime from `api.addPanel(...)` with `referencePanel 'main' does not exist` when rehydrating visible side rails after loading a saved layout that did not contain the main anchor panel.

**Root cause**: Visibility-sync logic in `EditorDockLayout` assumed `main` (or first main rail id) always exists and used it as `position.referencePanel` for first left/right/bottom re-add. With persisted layouts or settings where main content was hidden/missing, Dockview rejected that add.

**Fix**:

- Rehydrate visible `mainPanels` first.
- Add an anchor-safe panel helper: if `position.referencePanel` is missing, omit `position` and add at root.
- Resolve side-rail first-panel anchors in order:
  1. first visible main panel id,
  2. configured main id,
  3. first existing panel id in the current layout.
- Keep existing behavior for closing hidden panels and for within-rail tab stacking.

**Result**: Main remains hideable, and layout recovery no longer throws when saved layout state has no main anchor.

---

## Dockview tab props leaking to DOM

**Problem**: Dockview panel header props include `containerApi` and other non-DOM fields. Spreading them onto a `<div>` triggers React warnings or invalid DOM props.

**Fix**: In `DockviewSlotTab`, destructure `containerApi` (and any Dockview-only props like `tabLocation`) before spreading the rest onto the DOM element.

---

## FlexLayout panels not showing (only overlays visible)

**Problem**: After the Dockviewâ†’FlexLayout swap, only overlay UI (e.g. app bar, modals) was visible; no Library/Main/Inspector/Workbench panels.

**Fix**: FlexLayout was removed; DockLayout is back to **Dockview**. If panels disappear, reset the Dockview layout (`resetLayout()` or clear `localStorage['dockview-{layoutId}']`) rather than reintroducing FlexLayout.

---

## MDX build error: missing frontmatter

**Problem**: `pnpm --filter @forge/studio build` failed with `[MDX] invalid frontmatter ... expected string, received undefined` for a doc under `docs/` (e.g. `docs/architecture/README.md`, `docs/how-to/27-posthog-llm-analytics.mdx`, or any `.md`/`.mdx` in the docs collection).

**Fix**: The requirement applies to **all** `.md` and `.mdx` under `docs/`: fumadocs-mdx requires YAML frontmatter with at least **`title`** (string). Add frontmatter to the failing file. Example:
```
---
title: Plan - Editor layout main content not showing
created: 2026-02-07
updated: 2026-02-08
---
```
For the ongoing convention (new or moved docs), see [standard-practices](standard-practices.md) Â§ Docs (MDX build).

---

## Build: uncaughtException "data argument must be string or Buffer... Received undefined"

**Problem**: `pnpm --filter @forge/studio build` failed during "Creating an optimized production build" with `TypeError: The "data" argument must be of type string or an instance of Buffer, TypedArray, or DataView. Received undefined` (code `ERR_INVALID_ARG_TYPE`). No stack trace in output; typically from `Buffer.from(undefined)`, `fs.writeFile(path, undefined)`, or `crypto.Hash.update(undefined)`.

**Fixes applied**: (1) **api-client request.ts** â€” `base64(str)` now guards: if `str` is null/undefined or not a string, return empty-base64 instead of calling `Buffer.from(str)`. (2) **generate-openapi.mjs** â€” never pass undefined to `writeFileSync`: use `spec != null ? JSON.stringify(spec, null, 2) : '{}'`. (3) **next.config.ts SanitizeUndefinedAssetSourcePlugin** â€” guard `sanitizeUndefinedSources(compilation, assets)` so if `assets` is null/undefined we return immediately (avoids `Object.entries(assets)` throwing when processAssets calls the hook with undefined at some stages). Re-run build after changes; if the error persists, the source may be inside a dependency (e.g. hashing in webpack).

---

## Catalog route: where type not assignable to Payload Where

**Problem**: `pnpm --filter @forge/studio build` failed with type error in `apps/studio/app/api/catalog/route.ts` (line 22): `Type 'Record<string, unknown>' is not assignable to type 'Where'`. Payload's `find()` expects `where` to be of type `Where`; a variable typed as `Record<string, unknown>` is too loose.

**Fix**: Build the `where` object inline in the `payload.find()` call (e.g. `where: slug ? { status: { equals: 'published' }, slug: { equals: slug } } : { status: { equals: 'published' } }`) so TypeScript infers the correct type. Avoid typing query objects as `Record<string, unknown>` when passing to Payload.

---

## CSS @import must precede all rules (globals.css parse error)

**Problem**: `pnpm dev` fails with "Parsing CSS source code failed" at `globals.css` (compiled line ~1115): `@import rules must precede all rules aside from @charset and @layer statements`. The offending lines are `@import url('https://fonts.googleapis.com/...')` (Inter, JetBrains Mono, Rubik, Mulish, etc.).

**Root cause (historical)**: A dependency (formerly e.g. `@twick/studio/dist/studio.css`) could contain `@import url(...)` for Google Fonts. Twick was removed 2026-02-24; if a future dependency injects font @import, the same fix applies. When `apps/studio/app/globals.css` is processed by PostCSS/Tailwind, the pipeline order produces a single CSS file where Tailwind's expanded rules appear first (~1100+ lines), then the inlined content from `@import "@twick/studio/dist/studio.css"` appears, so the font `@import` ends up after other rules. CSS requires all `@import` to be at the top.

**Attempted fix (insufficient)**: Moving all `@import` in globals.css to the very top (tw-animate, shared styles, studio) so they are "first" in the source file. This did not fix it because the Tailwind/PostCSS pipeline expands `@tailwind base/components/utilities` and merges dependency CSS in an order that still places dependency-injected `@import url()` after those rules in the final output.

**Fix**: Added a PostCSS plugin that hoists all `@import url(...)` rules to the top of the compiled CSS. See `apps/studio/scripts/postcss-hoist-import-url.cjs` and `apps/studio/postcss.config.mjs` (plugin runs after `@tailwindcss/postcss`). The config resolves the plugin with an absolute path (`path.join(__dirname, 'scripts', 'postcss-hoist-import-url.cjs')`) so Next/Turbopack can load it when PostCSS runs from `.next`. If this error reappears (e.g. after a dependency update), ensure the hoist plugin still runs after Tailwind and that it handles any new @import from the dependency.

---

## "getSnapshot should be cached" / useSyncExternalStore loop

**Problem**: Console error "The result of getSnapshot should be cached to avoid an infinite loop", often followed by "Maximum update depth exceeded". Triggered when a Zustand selector returns a **new object or array reference** every time (e.g. `state.getMergedSettings(...)` which spreads and returns a new object). React's `useSyncExternalStore` requires the snapshot to be referentially stable when state has not changed; a new reference each time is treated as a change and causes re-render ? getSnapshot again ? infinite loop.

**Fix**: Do not select merged or derived objects from the store in components. Use selectors that return **primitives or stable references** only (e.g. `(s) => s.getSettingValue('ai.agentName', ids)` with stable `ids`). Build any object needed in the component with `useMemo` from those primitive values. Optionally, the store can cache merged result per key and return the same reference when underlying data is unchanged. **Fallbacks:** If the selector uses `?? defaultValue` and `defaultValue` is an object or array, use a **module-level constant** (e.g. `EMPTY_RAIL`, `EMPTY_SECTIONS`) so the same reference is returned when the key is missing; otherwise `defaultValue` creates a new ref every call and triggers the loop (see panel-registry.ts, settings-registry.ts).

---

## Model switcher manual mode oscillation (Maximum update depth exceeded)

**Problem**: Selecting **Manual (pick one)** in ModelSwitcher caused the model router `mode` to flip between `auto` and `manual`, creating a rapid `store.setState` loop and React "Maximum update depth exceeded". The bidirectional sync between `ai.model` (settings) and the model-router store let router-driven changes trigger the settings ? router effect, which forced the mode back to auto before settings could update.

**Fix**: Make the settings ? router sync run **only when the app setting changes** (read current router state via `useModelRouterStore.getState()`), and keep the router ? settings sync to reflect router changes. This prevents router-driven updates from re-triggering the settings sync and eliminates the oscillation.

---

## "Maximum update depth exceeded" â€” Radix composeRefs hypothesis (FALSE POSITIVE)

**Context**: Error surfaces at Radix UI `setRef` / `ScrollAreaPrimitive.Root` / `WorkspaceTooltip` in the stack. External reports (Radix #3799) cite composeRefs + React 19 as cause.

**FALSE POSITIVE in this codebase**: The user explicitly confirmed that (a) Radix tooltip was **not** the cause â€” stripping tooltips did not fix the error; (b) Radix ScrollArea / replacing with native overflow is **forbidden** â€” "absolutely not"; (c) the composeRefs hypothesis was wrong for this repo. Many projects use Radix without issue.

**Do not**: Replace Radix ScrollArea, Tooltip, or other primitives with native alternatives based solely on the composeRefs hypothesis. Do not assume stack-trace location equals root cause.

**Actual cause**: Under investigation. See "Maximum update depth â€” panel/slot cascade" for attempted fixes. Tooltip strip and native-title migration remain for other reasons; Radix primitives stay in packages/ui.

---

## Maximum update depth â€” panel registration and slot store cascade

**Problem**: "Maximum update depth exceeded" in WorkspaceInspector / EditorDockPanel / ScrollArea path (Dialogue and Character editors). The panel registration cascade (WorkspaceRail â†’ setRailPanels â†’ EditorLayout â†’ setSlots â†’ SlotPanel) may contribute.

**Attempted fix (reverted)**: Deferring setRailPanels/setSlots via requestAnimationFrame caused graph panels to stop rendering (black/empty). rAF deferral reverted.

**Fix (slotIdsKey + shallow compare)**: Guard the `setSlots` effect in DockLayout so we only call `useSlotContentStore.getState().setSlots(resolvedSlots)` when the slots actually changed. Use a `prevSlotsRef` and compare keys + slot content by reference; skip if identical. Add `slotIdsKey = Object.keys(resolvedSlots).sort().join('|')` for effect stability. Do **not** remove `resolvedSlots` from the effect deps entirelyâ€”that caused main/right/bottom panels to not render while left worked (effect ran before all rails had populated or ran too rarely).

**Anti-pattern (left-only render)**: When the setSlots effect was changed to run only on `slotIdsKey` (and not `resolvedSlots`), the effect could run before all rails had populated, so only left was synced. Always ensure the effect runs with the complete resolvedSlots when structure or content changes.

**Layout JSON migration**: When using rails (`rightPanels` etc.), saved layouts with legacy `right-inspector` / `right-settings` panel ids are incompatible. DockLayout skips loading such layouts and falls through to `buildDefaultLayout`.

**Forbidden fix**: Do **not** replace Radix ScrollArea with native overflow. User explicitly forbade this; the Radix hypothesis is a false positive.

**Current state**: Synchronous setRailPanels and setSlots; setSlots guarded by shallow compare. If error recurs, try: (1) memoize panel content so descriptors have stable refs; (2) batch store updates; (3) audit selectors for unstable refs. Do not blame Radix or replace ScrollArea.

---

## Panel registry recursion fix (UI-first layout)

**Problem**: The imperative panel flow (WorkspaceRail effect â†’ setRailPanels â†’ panel registry store â†’ EditorLayout subscribes â†’ DockLayout) caused store update cascades and "Maximum update depth exceeded".

**Fix**: Replaced with **UI-first declarative API**. Editors compose `EditorDockLayout.Left` / `.Main` / `.Right` / `.Bottom` slot children with `EditorDockLayout.Panel` children. DockLayout collects panels from JSX in render; no store-driven registration, no effects for panel registration. `EditorLayoutProvider` now provides only `{ editorId }`; `WorkspaceRail`, `WorkspaceRailPanel`, and `EditorLayout` are deprecated. `useEditorPanelVisibility` uses `EDITOR_PANEL_SPECS` fallback when `useEditorPanels` returns empty.

**Do not**: Reintroduce imperative setRailPanels or store-driven panel registration for layout.

---

## Wide form dialogs + left-side close button regression

**Problem**: Character/create upsert dialogs rendered too wide (near edge-to-edge) for simple forms, with low visual hierarchy and inconsistent close placement (left-side close icon). This reduced readability and made the UI feel unpolished.

**Cause**: Overlay surfaces used broad width presets and did not enforce a consistent close-position contract. Some dialogs bypassed shadcn form section patterns and lacked tokenized spacing.

**Fix**: Standardize overlay/dialog sizing and close placement:

- `WorkspaceOverlaySurface` size mapping is compact by default (`sm -> max-w-md`, `md -> max-w-xl`, `lg -> max-w-2xl`).
- `full` remains constrained (`min(94vw, 72rem)`) with max-height cap; never edge-to-edge for standard forms.
- Close button is icon-only and top-right for all dialogs.
- Upsert forms use shadcn `Form` primitives plus section layout with tokenized spacing.

References: `packages/shared/src/shared/components/workspace/WorkspaceOverlaySurface.tsx`, `packages/ui/src/components/ui/dialog.tsx`, `apps/studio/components/character/CreateCharacterModal.tsx`, and [styling-and-ui-consistency.md](./styling-and-ui-consistency.md).

---

## Copilot sidebar closes when selecting a model

**Problem**: Selecting a model from the ModelSwitcher inside the Copilot sidebar caused the sidebar to close immediately.

**Cause**: The model picker popover was portaled to `document.body` (Radix Popover default). CopilotKit renders the sidebar as a Dialog; clicking inside a portaled popover is treated as an "outside click" and closes the dialog.

**Fix**: Add a `portalled` option to `@forge/ui/popover` and set `portalled={false}` for ModelSwitcher in the `composer` variant (Copilot sidebar / assistant-ui). Keep the portal for toolbar usage so popovers can escape overflow when not inside a dialog.

---

## Cmd+K assistant popup: no model switcher + stale model ID ("No endpoints found")

**Problem**: Global assistant chat opened from Cmd+K had no in-composer model switcher, so users could not recover when the selected model became invalid. Requests failed with OpenRouter `"No endpoints found for google/gemini-2.0-flash-exp:free"`.

**Cause**: (1) `DialogueAssistantPanel` did not expose composer slot props to `Thread`, so `AssistantChatPopup` could not inject `ModelSwitcher`. (2) Legacy persisted model IDs were still read as-is, and the old default (`google/gemini-2.0-flash-exp:free`) could be unavailable.

**Fix**: (1) Forward `composerLeading` / `composerTrailing` through `DialogueAssistantPanel` to `Thread` and mount `ModelSwitcher provider="assistantUi" variant="composer"` in `AssistantChatPopup`. (2) Move default chat model to `openai/gpt-oss-120b:free`, normalize legacy unavailable IDs in persistence, and resolve selected model against the current OpenRouter registry before use (`resolveModelIdFromRegistry`) in `/api/model-settings`, `/api/assistant-chat`, and Copilot runtime resolver.

---

## MdxBody type error when using body as JSX component

**Problem**: In `apps/studio/app/docs/[[...slug]]/page.tsx`, `body` from `page.data` is narrowed to `body` when `typeof body === 'function'`, but TypeScript infers that as a generic `Function`. JSX requires a construct/call signature (e.g. `React.ComponentType`), so `<MdxBody components={mdxComponents} />` fails type-check.

**Fix**: Cast `body` to a React component type when assigning: `(typeof body === 'function' ? body : null) as React.ComponentType<{ components: typeof mdxComponents }> | null`. Keep the existing render: `{MdxBody ? <MdxBody components={mdxComponents} /> : null}`.

---

## Payload SQLite CANTOPEN (error 14) when opening /admin or creating project

**Problem**: `Error: cannot connect to SQLite: ConnectionFailed("Unable to open connection to local database ./data/payload.db: 14")` when visiting `/admin` or doing any Payload operation. SQLite error 14 is `SQLITE_CANTOPEN` (unable to open the database file).

**Cause**: The default DB URL was `file:./data/payload.db`, which is resolved relative to the **process cwd**. When running `pnpm dev` from the repo root, cwd is the root, so the path pointed at `./data/payload.db` under the root; that directory often doesn't exist, or the path is wrong when Next runs from a different working directory.

**Fix**: In `apps/studio/payload.config.ts`, the default DB path is now resolved relative to the config file: `path.join(dirname, 'data', 'payload.db')`, and the URL is built with `pathToFileURL(...).href` so libsql receives a valid absolute file URL. The config also ensures `apps/studio/data` exists with `fs.mkdirSync(dataDir, { recursive: true })` when `DATABASE_URI` is not set. To use a custom path, set `DATABASE_URI` in `.env` (e.g. `file:./data/payload.db` for a path relative to cwd, or an absolute path).

---

## Payload DB: pathToFileURL not defined + DATABASE_URI only in .env.example

**Problem**: `ReferenceError: pathToFileURL is not defined` at `payload.config.ts` when hitting e.g. `/api/projects`. Payload could not connect to the database.

**Causes**: (1) `pathToFileURL` was used in the config but never imported (only `fileURLToPath` was imported from `'url'`). (2) Agents had only updated `.env.example` with `DATABASE_URI`; the actual secret files (`.env`, `.env.local`) were not updated, so `DATABASE_URI` was empty at runtime. The code then fell back to the default path and tried to call `pathToFileURL(defaultDbPath).href`, which threw.

**Fix**: (1) In `payload.config.ts`, import `pathToFileURL` from `'url'`: `import { fileURLToPath, pathToFileURL } from 'url'`. (2) When adding or changing env vars that affect runtime (e.g. `DATABASE_URI`), **update both** `.env.example` (documentation) **and** the real env files (`.env` or `.env.local`) so the app has a valid value. If you leave `DATABASE_URI` unset on purpose, the default local SQLite path is used and requires the `pathToFileURL` import to build a valid file URL.

---

## BuiltInAgent / OpenRouter SDK incompatibility

**Problem**: Using `@openrouter/ai-sdk-provider` (e.g. `createOpenRouter`) for the CopilotKit agent causes interface mismatch. CopilotKit expects `openai` for the adapter and `@ai-sdk/openai` for BuiltInAgent; the OpenRouter provider uses a different shape. We swapped runtimes multiple times trying to use the OpenRouter SDK.

**Fix**: Do **not** use `@openrouter/ai-sdk-provider` for the CopilotKit route or for `createForgeCopilotRuntime` in shared. Use **OpenAI** (`openai` package) with `baseURL: config.baseUrl` and **createOpenAI** from `@ai-sdk/openai` with the same baseURL. Model fallbacks are implemented via a custom fetch that injects `models: [primary, ...fallbacks]` into the request body (see [openrouter-fetch.ts](../../apps/studio/lib/model-router/openrouter-fetch.ts)). Reference: [03-model-routing-openrouter.mdx](../../architecture/03-model-routing-openrouter.mdx).

---

## Verdaccio `npm adduser` 409 Conflict / access denied

**Problem**: `npm adduser --registry http://localhost:4873` returns `E409 Conflict` with "bad username/password, access denied".

**Cause**: The username already exists in `verdaccio/storage/htpasswd` (local registry auth file) and the password does not match.

**Fix**: Either log in with the existing user:

```bash
npm login --registry http://localhost:4873 --auth-type=legacy
```

Or reset the user by deleting `verdaccio/storage/htpasswd`, restarting Verdaccio, and re-adding the user:

```bash
rm -f verdaccio/storage/htpasswd
pnpm registry:start
npm adduser --registry http://localhost:4873 --auth-type=legacy
```

**Note**: Even with `publish: $all`, npm expects auth for scoped publishes; log in once (or use `npm-cli-login`) before running publish scripts.

---

## Forge publish pipeline failures (DTS + Verdaccio)

**Problem**: `pnpm registry:forge:build` failed in `@forge/shared` DTS build with React `ref` type conflicts and tool-ui `<style jsx>` props, then `@forge/agent-engine` / `@forge/dev-kit` DTS builds failed to resolve `@forge/*` packages. Publishing failed because `npm publish packages/ui` was interpreted as a GitHub repo (`packages/ui`), and Verdaccio required auth.

**Fix**:

- **React types**: enforce a single `@types/react` + `@types/react-dom` via root `pnpm.overrides`.
- **Markdown component props**: strip `ref` before spreading props in `markdown-text.tsx` to avoid cross-react-type conflicts.
- **Tool UI CSS**: replace `<style jsx global>` with a plain `<style>` using `dangerouslySetInnerHTML`.
- **DTS resolution**: add `tsconfig.json` with `moduleResolution: "bundler"` to `packages/agent-engine` and `packages/dev-kit`.
- **Dev-kit exports**: avoid duplicate exports by namespacing UI (`export * as ui from '@forge/ui'`).
- **Publish scripts**: prefix publish paths with `./` so npm treats them as local directories.
- **Verdaccio auth**: add `auth.htpasswd` config and log in once (or use `pnpm dlx npm-cli-login`) before publish.

---

## CopilotKit BuiltInAgent fails on OpenRouter v3 responses

**Problem**: `AI_UnsupportedModelVersionError` when CopilotKit BuiltInAgent runs with OpenRouter models like `google/gemini-2.0-flash-exp:free`. AI SDK 5 responses API only supports spec v2, but some OpenRouter-backed providers return v3.

**Cause**: BuiltInAgent uses the OpenAI **responses** API by default. Models that return v3 responses (Gemini, Claude) are incompatible and crash.

**Fix**:
- Add responsesâ€‘compatibility metadata to the model registry (`supportsResponsesV2`).
- Filter CopilotKit model selection to responsesâ€‘v2 compatible models, with a safe fallback (e.g. `openai/gpt-4o-mini`).
- Keep assistantâ€‘ui chat on the **chat** pipeline so it can use a broader set of models.

---

## createForgeCopilotRuntime import path (server vs client)

**Problem**: `/api/copilotkit` failed with `TypeError: createForgeCopilotRuntime is not a function` when imported from `@forge/shared/copilot/next`.

**Cause**: `@forge/shared/copilot/next` exports **client-only** provider APIs; the runtime factory lives in `@forge/shared/copilot/next/runtime`.

**Fix**: Import `createForgeCopilotRuntime` from `@forge/shared/copilot/next/runtime` in server routes (`apps/studio/app/api/copilotkit/handler.ts` and any consumer API routes).

---

## CopilotKit runtime sync: Agent "default" not found

**Problem**: `useAgent: Agent 'default' not found after runtime sync (runtimeUrl=/api/copilotkit). No agents registered.`

**Cause**: Only `POST /api/copilotkit` existed. CopilotKit syncs agents via `GET /api/copilotkit/info` (and sometimes `/api/copilotkit/agents__unsafe_dev_only`). Without those routes, the client sees no agents.

**Fix**: Add a catch-all route `apps/studio/app/api/copilotkit/[...path]/route.ts` and export `GET` for `/api/copilotkit` so the runtime handler serves `/info` and dev-only endpoints. Share the handler via `apps/studio/app/api/copilotkit/handler.ts`.

**Note:** If the handler throws before returning (e.g. `ReferenceError: log is not defined` in resolveModel), the client will also see "No agents registered" because GET /info never gets a successful response. Ensure the handler does not throw on request (e.g. define logger; validate env at module load).

---

## CopilotKit removed (Phase 7 migration, complete 2026-02-12)

**Status**: CopilotKit fully removed: `@copilotkit/*` deps removed from studio/shared/consumer; shared copilot types stripped of @copilotkit imports; CopilotKitProvider, copilot/next (runtime, provider), DomainCopilotRegistration, use-domain-copilot* deleted; COPILOTKIT_FLAG_KEY removed. Assistant UI is the chat surface. `CopilotKitBypassProvider` supplies only `CopilotSidebarContext` for legacy sidebar state. See [AI migration 00-index](../../ai/migration/00-index.mdx).

**Archived entries below** (BuiltInAgent, createForgeCopilotRuntime, etc.) document historical issues; the code paths no longer exist.

---

## CopilotKit handler: ReferenceError log is not defined

**Problem**: `ReferenceError: log is not defined` at `handler.ts:35` (resolveModel). POST /api/copilotkit returns 500.

**Cause**: resolveModel used `log.info(...)` without importing or defining `log`. Studio uses structured logging via `getLogger` from `@/lib/logger`.

**Fix**: In `apps/studio/app/api/copilotkit/handler.ts`, add `import { getLogger } from '@/lib/logger';` and `const log = getLogger('copilotkit');`. Do not use ad-hoc `console` for API/routing; use the Studio logger per the existing "Logging: use Studio logger" guidance in this file.

---

## Model switcher errors and excess Studio calls on load (planned work)

**Problem**: Model switcher is producing many errors; Studio makes too many calls on app load. Related: registry hydration, settings â‡„ router sync (see "Model switcher registry empty" and "Model switcher manual mode oscillation" in this file).

**Planned work**: (0) **AI agent and model provider plan** â€” discuss/document single source of truth, who provides models, avoid duplicate fetches and sync loops. (1) **Model switcher stability** â€” single source of truth, registry hydrated once, no oscillation. (2) **Reduce Studio calls on load** â€” fewer calls, defer/batch, single init path. See [STATUS Â§ Next](./STATUS.md) and initiative `model-routing-stability` in [task-registry](./task-registry.md). Architecture: [03-model-routing-openrouter.mdx](../../architecture/03-model-routing-openrouter.mdx).

---

## Single model router at Studio (no root duplicate)

**Do not** add a second model router at repo root. The **single** model router (server-state, registry, openrouter-config) lives **only** in `apps/studio`. All AI/model API routes are under `apps/studio/app/api/`. See [03-model-routing-openrouter.mdx](../../architecture/03-model-routing-openrouter.mdx).

---

## Drawer requires DialogTitle (Workbench)

**Problem**: Opening the Dialogue workbench triggered `DialogContent requires a DialogTitle` from Radix/vaul.

**Cause**: `DrawerContent` rendered without a `DrawerTitle`, which Radix requires for accessibility.

**Fix**: Add `<DrawerTitle className="sr-only">Workbench</DrawerTitle>` inside the drawer content (DialogueEditor). Use `DrawerTitle` (not a plain `<div>`) so Radix can detect it.

---

## Settings / Workbench drawer or sheet not opening when clicked

**Problem**: Clicking Settings (app bar or editor toolbar) or Workbench (Dialogue editor) did not open the Sheet or Drawer; panels appeared not to respond.

**Cause**: Stacking order: CopilotKit sidebar and other UI (e.g. Dockview) use z-index 30â€“50. Our Sheet and Drawer used z-50 / z-100 and could render behind another layer or a floating layer could capture clicks before they reached the buttons.

**Fix**: (1) Raise overlay z-index for Sheet, Drawer, and DropdownMenu in `packages/ui`: use `z-[200]` for overlay and content so they sit above CopilotKit and Dockview. (2) In `apps/studio/app/globals.css`, add `.copilotKitSidebar { z-index: 20; }` so the CopilotKit sidebar stays below our overlays. (3) Ensure DropdownMenuContent (editor Settings menu) uses the same z-[200]. See [styling-and-ui-consistency.md](./styling-and-ui-consistency.md) for UI debugging.

---

## Repo Studio Env render loop + runtime deps semantics drift (2026-02-23)

**Problems**:
1. `EnvWorkspace` intermittently crashed with `Maximum update depth exceeded` (Select trigger stack), followed by secondary React unmount errors.
2. `/api/repo/runtime/deps` returned `500` for `desktop.nextStandalonePresent=false` even though doctor treated that condition as diagnostic-only.
3. `forge-repo-studio doctor` output did not clearly guide runtime start/reclaim actions when runtime was stopped or ports were occupied.

**Causes**:
- Target-sync effects depended on non-memoized arrays and reset state with fresh object literals (`setEditedValues({})`) even when already empty.
- Runtime deps route mixed hard readiness and diagnostics in one `ok/status` path.
- Doctor runtime row was warning-only with no explicit quick-action command block.

**Fix**:
- Extracted pure target helpers (`deriveNextSelectedTargetId`, `shouldResetTargetState`) and switched Env workspace sync to memoized target lists + idempotent functional resets.
- Added stale async load guard for target fetch races and normalized empty Select value to `undefined`.
- Moved runtime dependency evaluation to `src/lib/runtime-deps-evaluator.ts`; route now always returns HTTP `200` with additive fields: `desktopRuntimeReady`, `desktopStandaloneReady`, `severity`.
- Upgraded doctor runtime section with deterministic quick actions:
  - stopped (expected): start commands,
  - running: URL + stop command,
  - port listener conflict: reclaim command flow.
- Added optional doctor link suppression flag: `--no-links` (OSC8 links stay rich-mode-only).

**Guardrails**:
- New tests:
  - `apps/repo-studio/__tests__/env/env-target-state.test.mjs`
  - `apps/repo-studio/__tests__/api/runtime-deps-route.test.mjs`
  - `packages/repo-studio/src/__tests__/doctor-runtime-hints.test.mjs`
  - `packages/repo-studio/src/__tests__/terminal-format-links.test.mjs`
- Note: Next route files cannot export extra symbols beyond route handler contracts; shared evaluators must live in separate modules (e.g. `runtime-deps-evaluator.ts`).

---

## Repo Studio workspace overload + assistant context UX mismatch (2026-02-23)

**Problems**:
1. Repo Studio surfaced too many panels at once and behaved unlike a focused workspace app.
2. Codex auth/session controls were embedded as a large setup card inside assistant content.
3. Assistant context still relied on panel-side "Attach To Assistant" actions instead of chat-input mentions.
4. Terminal panel did not provide a true interactive shell workflow.

**Causes**:
- Global visibility model + static tab labeling encouraged panel sprawl instead of workspace-focused presets.
- Codex setup was implemented as in-panel status card instead of app-bar controls.
- Legacy attachment actions remained in code/diff/git/story/review surfaces after mention-model direction was chosen.
- Terminal UI lacked persistent session APIs (start/stream/input/resize/stop).

**Fix**:
- Added workspace preset contracts and per-workspace hidden-panel maps; main tabs now render true workspace tabs with open/close/switch behavior.
- Moved Codex controls to compact app-bar actions (`Sign In`, `Refresh`, `Start/Stop`) with status surfaced via concise UI affordances.
- Adopted mention-first assistant context:
  - per-loop/per-assistant system prompts in settings,
  - `@planning/...` mention parsing/resolution in assistant chat route,
  - removed panel-level attach-context actions and related planning-attachment coupling.
- Replaced terminal panel with interactive `xterm` client backed by terminal session API routes and server-side session manager.
- Added sidebar close hardening (`X` button + `Esc`) and panel overflow fixes (`min-h-0` / `overflow-auto`).

**Guardrails**:
- Added workspace preset regression tests and mention-context parsing tests.
- Added terminal route API tests and style-pipeline compile test coverage.

---

## Repo Studio build hash/runtime terminal hardening follow-up (2026-02-23)

**Problems**:
1. `pnpm --filter @forge/repo-studio-app build` failed on Node 24 with webpack `WasmHash` crash (`TypeError: Cannot read properties of undefined (reading 'length')`).
2. Terminal session start route could return `500` when `node-pty` spawn/init failed in constrained test/runtime contexts.
3. Strict type-checking surfaced implicit-any callbacks in shared docs sidebar (`DocsSidebar.tsx`), blocking app build.

**Causes**:
- Webpack wasm hash path instability under this environmentâ€™s Node 24 runtime.
- PTY startup path threw without fallback, so route-level catch returned `500`.
- Untyped callback parameters in `.some(...)` calls under strict TS settings.

**Fix**:
- Added Repo Studio webpack override in `apps/repo-studio/next.config.ts`:
  - `config.output.hashFunction = 'sha256'`
  - Avoids wasm hash path and restores deterministic build behavior on Node 24.
- Hardened terminal session startup in `apps/repo-studio/src/lib/terminal-session.ts`:
  - Added degraded fallback PTY mode when `node-pty` fails.
  - Start route now returns `200` with fallback metadata/message instead of hard `500`.
  - Terminal UI surfaces fallback status/reason.
- Typed docs sidebar callbacks in `packages/shared/src/shared/components/docs/DocsSidebar.tsx` to remove implicit-any build failures.

**Guardrails**:
- Re-ran API tests to confirm terminal route behavior in fallback scenarios.
- Re-ran full Repo Studio app build to ensure strict type + build pipeline stays green.

---

## Repo Studio workspace-as-layout migration (2026-02-24)

**Problems**:
1. Workspace tabs still behaved like one global Dockview instance with preset-based hide/show state.
2. Panel naming was ambiguous (`*Workspace` used for panel content while tabs also represented workspaces).
3. Legacy persistence state (`repo-studio-main` + global hidden panels) caused migration drift when moving to workspace-scoped layouts.

**Causes**:
- `RepoStudioShell` mounted one large `EditorDockLayout` and conditionally rendered all panels via visibility map (`workspace-presets` + `useRepoPanelVisibility`).
- `EditorDockLayout` still prioritized array rail props over declarative slot-collected panels.
- Store migration logic was tied to preset defaults, not workspace layout definitions.

**Fix**:
- Introduced workspace layout definitions (`workspace-layout-definitions.ts`) and explicit one-layout-per-workspace components under `components/layouts/*Layout.tsx`.
- Refactored shell to mount active workspace layout with per-workspace layout ids (`repo-<workspaceId>`) and workspace-scoped panel specs/visibility.
- Removed preset and legacy visibility helpers (`workspace-presets.ts`, `useRepoPanelVisibility.ts`) and migrated store to version 4 with:
  - legacy layout key migration (`repo-studio-main` -> active workspace layout id),
  - hidden-panel sanitization by workspace panel set,
  - guard to keep at least one main panel visible.
- Normalized naming to `*Layout` (tab-level) + `*Panel` (panel content exports), with temporary aliases for compatibility.
- Updated shared `EditorDockLayout` precedence so slot children override array props; documented array props as deprecated compatibility surface.

**Guardrails**:
- Added shell regression tests:
  - `apps/repo-studio/__tests__/shell/workspace-layout-definitions.test.mjs`
  - `apps/repo-studio/__tests__/shell/workspace-store-migration.test.mjs`
  - `apps/repo-studio/__tests__/shell/workspace-visibility.test.mjs`
- Verified with:
  - `pnpm --filter @forge/repo-studio-app exec node --import tsx --test \"__tests__/shell/*.test.mjs\"`
  - `pnpm --filter @forge/repo-studio-app test:api`
  - `pnpm --filter @forge/repo-studio-app test:styles`
  - `pnpm --filter @forge/repo-studio-app build`
  - `pnpm --filter @forge/studio test`
  - `pnpm --filter @forge/studio build`
  - `pnpm forge-repo-studio doctor`

---

## Semantic hard-cut follow-up: workspace-first contracts + payload typegen resolver (2026-02-24)

**Problems**:
1. Semantic migration left mixed compatibility behavior in settings/assistant contracts (`editor` fallbacks still accepted in some paths).
2. `pnpm payload:types` failed from root generator path resolution:
   - `ERR_MODULE_NOT_FOUND` for `@payloadcms/db-postgres`
   - `ERR_PACKAGE_PATH_NOT_EXPORTED` for Payload internals.
3. Repo Studio build regressed after function signature cleanup (`renderDatabaseDockPanel` call-site mismatch).

**Causes**:
- Payload type generator imported adapters and internals via root-level static imports/export paths that were not valid under workspace package resolution and package `exports` constraints.
- One workspace call site was not updated after panel helper signature simplification.
- Settings API/store still carried `editor -> workspace` compatibility normalization after hard-cut naming decision.

**Fix**:
- Enforced workspace-first settings contracts:
  - removed `scope === 'editor'` mapping in `apps/studio/app/api/settings/route.ts`,
  - removed `editor` normalization branch in `apps/studio/lib/settings/store.ts`.
- Hardened payload type generator (`scripts/generate-payload-types.mjs`):
  - resolve/import from `apps/studio` package context via `createRequire`,
  - lazily import only active DB adapter (postgres/sqlite),
  - resolve Payload package root from installed entry path and load `dist/bin/generateTypes.js` by absolute file path.
- Fixed Repo Studio signature drift:
  - updated `DatabaseWorkspace` to call `renderDatabaseDockPanel()` without stale args.
- Removed legacy rail-prop test path in Studio dock anchor regression test and kept slot-based assertions only.

**Guardrails / verification**:
- `pnpm payload:types`
- `pnpm --filter @forge/shared build`
- `pnpm --filter @forge/studio test`
- `pnpm --filter @forge/studio build`
- `pnpm --filter @forge/repo-studio-app test:api`
- `pnpm --filter @forge/repo-studio-app build`
- `pnpm forge-repo-studio doctor`

---

## AI/chat-first hard-cut: canonical assistant + inline panel composition (2026-02-24)

**Problems**:
1. Assistant runtime wiring drifted into app-local wrappers (`RepoAssistantPanel`, `DialogueAssistantPanel`), causing duplicated behavior and semantic confusion.
2. Repo Studio workspace composition regressed into render helper indirection (`render*DockPanel` via `workspaces/panels.tsx`).
3. Repo Studio request contracts still used `editorTarget`, conflicting with the workspace/assistant semantic model.
4. Consumer reference app lived under `examples/consumer` with local assistant API coupling instead of companion-runtime routing.

**Causes**:
- No hard guardrails enforced canonical shared assistant runtime surface.
- Workspace panel trees were abstracted behind helper functions instead of declared where each workspace layout is defined.
- Contract rename was incomplete across route/session/proposal/service/schema layers.
- Consumer example architecture lagged behind AI/chat-first companion mode decisions.

**Fix**:
- Added shared canonical assistant runtime component:
  - `packages/shared/src/shared/components/assistant-ui/AssistantPanel.tsx`
  - migrated Studio and Repo Studio to use this shared panel directly.
- Removed app-local assistant wrappers:
  - deleted `apps/repo-studio/src/components/RepoAssistantPanel.tsx`
  - deleted `apps/studio/components/workspaces/dialogue/DialogueAssistantPanel.tsx`
- Hard-cut Repo Studio contracts to `assistantTarget`:
  - assistant chat route, codex session, proposal contracts/repository, story publish services/routes, payload collection field.
- Inlined `WorkspaceLayout.Panel` JSX in every Repo Studio `*Workspace.tsx` and deleted `apps/repo-studio/src/components/workspaces/panels.tsx`.
- Added shared companion runtime primitives:
  - `companion-runtime-store`, `CompanionRuntimeSwitch`, `useCompanionAssistantUrl`.
  - migrated Studio to shared switch/url hook usage.
- Added companion CORS support:
  - `OPTIONS` + scoped CORS headers for `/api/repo/health` and `/api/assistant-chat`.
- Removed `examples/consumer` and added `apps/consumer-studio` chat-only dev-kit app.

**Guardrails / verification**:
- Added scripts:
  - `scripts/guard-assistant-canonical.mjs`
  - `scripts/guard-workspace-semantics.mjs`
- Wired guard scripts into root `lint` and CI workflow.
- Updated AGENTS guidance to codify chat-first + shared assistant canonical policy.

---

## Repo Studio extensions-first hard cut: payload bootstrap coupling in route tests (2026-02-26)

**Problems**:
1. Repo Studio API test suite failed on multiple routes with Payload bootstrap crash:
   - `TypeError: Cannot destructure property 'loadEnvConfig' of 'import_env.default' as it is undefined`
   - surfaced in `test:api` after adding extension discovery and active-project root routing.
2. Extension route test (`/api/repo/extensions`) failed even though extension discovery itself should be filesystem-only.

**Causes**:
- `apps/repo-studio/src/lib/project-root.ts` imported `payload-client` at module scope.
- Routes that only needed `resolveActiveProjectRoot()` still loaded Payload runtime transitively during module evaluation.

**Fix**:
- Switched `project-root.ts` to lazy payload loading:
  - added `getRepoStudioPayloadLazy()` with dynamic import,
  - moved payload access behind async project persistence/list functions only (`saveProject`, `listRepoProjects`, `setActiveRepoProject`),
  - kept root resolution/browse paths payload-free.
- Fixed Next lint blocker introduced by dynamic import helper by renaming local binding from `module` to `payloadClientModule` (`@next/next/no-assign-module-variable`).

**Guardrails / verification**:
- `pnpm --filter @forge/repo-studio-app run test:api` (pass)
- `node --import tsx --test "apps/repo-studio/__tests__/shell/*.test.mjs"` (pass)
- `pnpm --filter @forge/repo-studio-app build` (pass; warnings remain)
- `pnpm guard:workspace-semantics` (pass)
- `pnpm hydration:doctor` (pass)

---

## Repo Studio extraction cut: hydration doctor root drift after legacy move (2026-02-27)

**Problems**:
1. `pnpm hydration:doctor` failed with `rg: apps/studio: The system cannot find the file specified` after moving Studio sources to `legacy/studio-apps/studio`.
2. Verification closeout for extraction slice could not pass root guardrail set while hydration doctor still assumed old app paths.

**Causes**:
- `scripts/hydration-nesting-doctor.mjs` hardcoded `ROOT_GLOBS = ["apps/studio", "apps/platform", ...]`.
- After legacy move, `apps/studio` no longer exists.

**Fix**:
- Updated hydration doctor scan roots to active targets:
  - `apps/repo-studio`
  - `apps/platform`
  - `packages/shared`
  - `packages/ui`
- Added existence filtering before running `rg`, so missing roots do not fail the command.

**Guardrails / verification**:
- `pnpm hydration:doctor` (pass)
- `pnpm guard:workspace-semantics` (pass)
- `node scripts/build.mjs` in `vendor/repo-studio-extensions` (pass)

---

## Repo Studio desktop silent-installer investigation (2026-02-28)

**Problem**:
1. Silent installer validation remained ambiguous: sometimes the custom `/D=` target looked empty, other times it contained a partial install, but the process still had not exited.
2. Installed silent-build launches were failing health checks even though the current `win-unpacked` executable could boot successfully.

**Findings**:
- Fresh `win-unpacked/RepoStudio.exe` passed local launch smoke and served `GET /api/repo/health` with `200`.
- `RepoStudio Silent Setup 0.1.5.exe` could run longer than early smoke thresholds; killing it mid-run produced partial installs.
- Partial installs were missing required runtime files under `resources/next/standalone/node_modules/next`, causing startup failures (`Cannot find module 'next'`).

**Fix**:
- Added structured probes:
  - `packages/repo-studio/src/desktop/smoke-install.mjs`
  - `packages/repo-studio/src/desktop/smoke-launch.mjs`
- Hardened installer completion logic in `smoke-install.mjs`:
  - required-file completion checks (including `Uninstall RepoStudio.exe` and `next/dist/server/next.js`),
  - install progress snapshot tracking,
  - explicit `timedOut` vs `stalled` reasons.
- Updated `desktop:smoke:silent` to run install + launch (`--kind silent --timeout-ms 420000 --launch`).
- Set `compression: "store"` for silent NSIS builds in `packages/repo-studio/electron-builder.silent.json` to reduce extraction bottlenecks.
- Kept scratch cleanup between package stages (`desktop:clean:scratch`) to avoid stale/corrupt `.nsis.7z` carryover.

**Verification**:
- `pnpm --filter @forge/repo-studio run desktop:package:win` (pass).
- `pnpm --filter @forge/repo-studio run desktop:smoke:silent` (pass):
  - installer `exitCode: 0`,
  - `installReady: true`,
  - launched installed app reached `/api/repo/health` with `200`.

**Guardrail**:
- Do not treat a timed-out silent installer as a completed install.
- Validate installer completion via required runtime files before launch assertions.

---

## Repo Studio desktop packaging cut: Windows builder blockers and first artifact output (2026-02-27)

**Problems**:
1. Electron packaging initially failed with electron-builder constraint: `Package "electron" is only allowed in "devDependencies"`.
2. Windows code-sign helper extraction failed with symlink privilege errors while unpacking `winCodeSign` cache archives.
3. Next standalone build still fails in this environment with `EPERM` symlink errors, triggering fallback desktop build mode.

**Causes**:
- `packages/repo-studio/package.json` had `electron` under `dependencies`.
- Electron-builder default Windows path tried to extract helper artifacts requiring symlink privileges unavailable in this user context.
- Next standalone tracing under pnpm store attempted symlink operations requiring elevated privileges.

**Fix**:
- Moved `electron` to `devDependencies` in `packages/repo-studio/package.json`.
- Updated `packages/repo-studio/electron-builder.json` with `win.signAndEditExecutable = false` to avoid blocking winCodeSign extraction path in this environment.
- Re-ran packaging to successful artifact output:
  - `packages/repo-studio/dist/desktop/RepoStudio Setup 0.1.1.exe`
  - `packages/repo-studio/dist/desktop/RepoStudio 0.1.1.exe`

**Remaining follow-up**:
- FRG-1521 tracks removing fallback-only packaging behavior by resolving standalone asset bundling (`.desktop-build/next`) when symlink privileges are unavailable.

---

## Repo Studio release hard gate: strict standalone packaging vs local Windows symlink permissions (2026-02-27)

**Problems**:
1. After enforcing strict release packaging (`desktop:package:win` requires standalone), local packaging now exits hard-fail in non-privileged Windows shells.
2. Without an explicit gate, fallback manifest mode could still produce installers that lacked bundled runtime web assets.

**Causes**:
- Next standalone tracing still performs symlink creation under pnpm store paths.
- Local shell lacks symlink privilege (`EPERM`), so standalone output is incomplete.

**Fix**:
- Added strict mode to desktop build:
  - `node src/desktop/build.mjs --require-standalone` refuses fallback mode.
- Added standalone verifier:
  - `packages/repo-studio/src/desktop/verify-standalone.mjs`
  - enforces `.desktop-build/next/BUILD_ID`, `.desktop-build/next/static`, and standalone server candidate presence.
- Updated packaging command:
  - `desktop:package:win` now runs strict build + verifier before `electron-builder`.
- Added tag-driven release workflow (`.github/workflows/release-repo-studio-desktop.yml`) that runs verify gates and standalone assertions before artifact upload.

**Operational note**:
- Local strict packaging may still fail without symlink privilege (expected).
- Release packaging is delegated to CI Windows runners where symlink creation is expected to be available.

---

## ERR_PNPM_LOCKFILE_CONFIG_MISMATCH prevention (2026-02-28)

**Cause**:
- Overrides changed without lockfile refresh, or pnpm version mismatch between local and CI.
- v0.1.5 release failed because CI `pnpm install --frozen-lockfile` hit `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`.

**Fix**:
- Added `packageManager: "pnpm@9.15.5"` to root `package.json` so the repo declares exact pnpm version.
- Added `pnpm.overrides` (lucide-react, @types/react, @types/react-dom) at root when missing.
- CI workflows use pnpm/action-setup reading version from `packageManager`; local dev uses Corepack (`corepack enable`).

**Guardrail**:
- When changing `pnpm.overrides` or root `package.json` dependencies, run `pnpm install` and include `pnpm-lock.yaml` in the commit.
- On Windows, `pnpm install` can fail with `EPERM: operation not permitted, unlink` on `esbuild.exe` if another process (IDE, antivirus, build) has it locked. Close other apps and retry.

---

## Desktop release smoke: install location and timeouts (2026-02-28)

**Cause**:
- NSIS can install to registry-stored prior path; assuming `/D=` is respected causes false failures when install actually went elsewhere.
- Short health poll (2 min) fails on low-RAM or cold-start runners.

**Fix**:
- Added registry-based `InstallLocation` detection in `smoke-install.mjs` (Windows): query `HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*` for `DisplayName = "RepoStudio"`.
- CI uses Node `smoke-install.mjs` instead of duplicated PowerShell; `--health-timeout-ms 360000` (6 min) for low-RAM tolerance.
- Win-unpacked smoke extended to 72 × 5s with diagnostics on failure (AppData, port 3020, processes, desktop-startup.log).

**Guardrail**:
- Do not assume `/D=` is the actual install path; query uninstall registry on Windows.

---

## Desktop release reliability closeout: CI guard scope + runtime probe + downloadable diagnostics (2026-02-28)

**Cause**:
- CI/release jobs were still coupled to `guard:workspace-semantics`, which can fail independently of desktop installer/runtime health and block release confidence work.
- Post-install smoke relied on logs only, with no downloadable failure payload.
- Silent install smoke validated launch health, but did not explicitly assert runtime dependency contract + codex CLI readiness as a separate post-install probe.

**Fix**:
- Removed `guard:workspace-semantics` from CI/release workflow gates (script remains for manual/local checks).
- Added explicit package-manager verification in CI/release (`package.json packageManager` vs `pnpm --version`) before `pnpm install --frozen-lockfile`.
- Added installed-runtime probe script (`packages/repo-studio/src/desktop/smoke-runtime-readiness.mjs`) that launches installed RepoStudio and hard-fails on:
  - `/api/repo/health` not ready,
  - `/api/repo/runtime/deps` not reporting `desktopRuntimeReady=true`,
  - `/api/repo/codex/session/status` not reporting `codex.readiness.cli.installed=true`.
- Added failure artifact upload in release workflow for:
  - `repostudio-smoke-result*.json`,
  - `repostudio-runtime-probe*.json`,
  - desktop startup log copied from `%APPDATA%`.

**Guardrail**:
- Desktop release reliability checks should prioritize install/launch/runtime readiness; keep non-critical semantic policy checks out of release gating path.

---

## Desktop installer smoke false-stall on upgraded installs (2026-02-28)

**Cause**:
- Silent installer smoke idle detection watched only the requested `/D=` probe directory.
- On upgraded/bad-prior installs, NSIS may write to a previously registered install location, so smoke saw "no progress" and could kill a valid install attempt.

**Fix**:
- Added shared install-location resolution (`install-locations.mjs`) and switched smoke monitoring to known install candidates:
  - requested install dir,
  - registry `InstallLocation`,
  - legacy/default local-program paths.
- Changed stall policy to fail only on `stalled-before-progress`; once any install progress is observed, smoke relies on explicit timeout rather than idle-killing.
- Tightened registry fallback launch detection to require actual `RepoStudio.exe` existence before treating the path as install-success candidate.

**Guardrail**:
- Never treat requested `/D=` path as the only source of installer progress.
- For Windows installer smoke, monitor known install candidates and keep stall aborts constrained to "no progress ever observed."

---

## Desktop release telemetry retention + timeout bounding (2026-02-28)

**Cause**:
- Some long-running release steps had no explicit timeout bound, increasing hang risk.
- Smoke/probe JSON artifacts were mostly failure-only, making successful-run baselines hard to compare during later regressions.

**Fix**:
- Added explicit `timeout-minutes` at release job and critical-step level in `.github/workflows/release-repo-studio-desktop.yml`.
- Enabled CI smoke/probe artifact writing for both pass/fail paths with `REPOSTUDIO_WRITE_SMOKE_ARTIFACTS=1`.
- Added always-on artifact upload (`repostudio-desktop-smoke-reports`) for:
  - `repostudio-smoke-result*.json`
  - `repostudio-runtime-probe*.json`
  - `repostudio-upgrade-repair*.json`
  - `repostudio-repair*.json`
- Added post-smoke owned-process cleanup step (`desktop:cleanup:owned`) to reduce stale process interference between smoke phases.

**Guardrail**:
- Keep release workflow deadlines explicit and keep smoke/probe telemetry downloadable on successful runs, not only failures.

---

## Desktop smoke triage UX: summary-first + local diff tooling (2026-02-28)

**Cause**:
- Investigations still required downloading raw JSON artifacts before seeing basic smoke/probe status.
- There was no small built-in utility to compare two report files from different runs.

**Fix**:
- Added `packages/repo-studio/src/desktop/summarize-smoke-reports.mjs` and package script:
  - `pnpm --filter @forge/repo-studio run desktop:smoke:summary -- --dir <runnerTemp> --write-summary`
- Release workflow now runs `Publish Desktop Smoke Summary` (`if: always()`) to write a compact markdown summary to `GITHUB_STEP_SUMMARY`.
- Added `packages/repo-studio/src/desktop/diff-smoke-artifacts.mjs` and package script:
  - `pnpm --filter @forge/repo-studio run desktop:smoke:diff -- --a <reportA.json> --b <reportB.json>`
- Diff tool filters volatile fields by default (PID/timestamps/artifact paths) so comparisons focus on behavioral differences.

**Guardrail**:
- Keep in-run summaries concise and keep deep diagnostics in artifacts.
- Use `desktop:smoke:diff` before changing smoke logic when debugging regressions between two runs.

---

## Desktop release notes lacked runtime-health context (2026-02-28)

**Cause**:
- Even after smoke/report artifacts improved, release pages still showed changelog-only text unless someone manually inspected the Actions run.

**Fix**:
- `release-repo-studio-desktop.yml` now:
  - uploads `repostudio-smoke-summary.md` in `repostudio-desktop-smoke-reports`,
  - downloads smoke reports in the `release` job,
  - builds `release-body.md` with a `Desktop Release Status` section,
  - publishes release with `body_path: release-body.md` and generated notes enabled.

**Guardrail**:
- Keep release descriptions self-diagnosing: include a compact desktop health section plus generated release notes.

---

## Silent install smoke one-shot brittleness in CI (2026-02-28)

**Cause**:
- Release workflow treated silent install smoke as a single-attempt gate.
- Recoverable cases (bad prior install state) required manual rerun rather than automated repair flow.

**Fix**:
- Updated release workflow to run:
  - primary: `desktop:smoke:silent`
  - retry on failure: `desktop:smoke:repair`
  - enforcement step that passes if either path succeeds, fails only if both fail.

**Guardrail**:
- Keep runtime readiness probe strict, but allow install smoke one automatic repair retry before failing release packaging.

---

*(Add new entries when new errors are found and fixed.)*

<!-- forge-loop:generated:start -->
## Forge Loop Snapshot

# Errors and Attempts

No errors recorded yet.
<!-- forge-loop:generated:end -->
