# Building OpenClaw from source

This guide explains what the OpenClaw package is and how to build it yourself.

## What this package is

**OpenClaw** is a monorepo that includes:

- **Core CLI / gateway** – Node.js (TypeScript): WhatsApp gateway CLI (Baileys web) with Pi RPC agent. Entry: `openclaw.mjs` → `dist/entry.js`.
- **Web UI** – Vite app in `ui/` (dashboard, chat, config).
- **Native apps** (optional): macOS (`apps/macos`), iOS (`apps/ios`), Android (`apps/android`), shared Swift code in `apps/shared/OpenClawKit`.
- **Extensions** – Channel adapters and plugins under `extensions/`.
- **Skills** – Bundled skills under `skills/`.

The **published npm package** is built from the root: it ships the compiled `dist/` tree, `openclaw.mjs`, extensions, docs, and related assets (see `package.json` → `files`).

---

## Prerequisites

- **Node.js** ≥ 22.12.0 (see `package.json` → `engines.node`).
- **pnpm** 10.23.0 (recommended; use Corepack: `corepack enable && corepack prepare pnpm@10.23.0 --activate`).
- **Git** – the repo may use submodules; CI runs `git submodule update --init --recursive` (see below).
- **Optional (for native apps):**
  - **macOS app**: Xcode (CI uses 26.1), XcodeGen, SwiftLint, SwiftFormat, Swift toolchain.
  - **iOS app**: Same as macOS + iOS Simulator.
  - **Android app**: Java 21, Android SDK, Gradle 8.11+.

---

## Steps to build the package yourself

### 1. Get the source

```bash
git clone <repo-url> openclaw
cd openclaw
```

If the project uses git submodules (check for a `.gitmodules` file or CI config), initialize them:

```bash
git submodule sync --recursive
git submodule update --init --recursive
```

### 2. Install dependencies

From the repo root:

```bash
pnpm install
```

This runs `scripts/postinstall.js` (git hooks, completion setup, and optionally patched deps for non-pnpm installs). Use frozen install in CI or for reproducible builds:

```bash
pnpm install --frozen-lockfile
```

### 3. Build the core package (TypeScript → dist)

This compiles `src/` to `dist/`, bundles A2UI assets, copies hook metadata, and writes build info:

```bash
pnpm build
```

What `pnpm build` does (see `package.json` → `scripts.build`):

1. **`pnpm canvas:a2ui:bundle`** – `scripts/bundle-a2ui.sh`: builds the Canvas A2UI bundle into `src/canvas-host/a2ui/` (skipped if A2UI sources are missing, e.g. in Docker).
2. **`tsc -p tsconfig.json --noEmit false`** – compiles TypeScript from `src/` to `dist/` (overrides `noEmit` in tsconfig).
3. **`node --import tsx scripts/canvas-a2ui-copy.ts`** – copies A2UI bundle assets from `src/canvas-host/a2ui/` to `dist/canvas-host/a2ui/`.
4. **`node --import tsx scripts/copy-hook-metadata.ts`** – copies `HOOK.md` files from `src/hooks/bundled` to `dist/hooks/bundled`.
5. **`node --import tsx scripts/write-build-info.ts`** – writes build metadata (version, git commit) under `dist/`.

After this, the CLI runs via:

- `node openclaw.mjs` (which loads `dist/entry.js`), or
- `pnpm openclaw` / `pnpm start` (which use `scripts/run-node.mjs` and can use `tsgo` for on-the-fly compile if available).

### 4. (Optional) Build the web UI

To build the Vite-based UI (e.g. for packaging or serving the dashboard):

```bash
pnpm ui:build
```

This runs `node scripts/ui.js build`, which executes the `build` script in `ui/` (e.g. `vite build`). For local development:

```bash
pnpm ui:dev
```

### 5. Full package build (what gets published)

Before packing/publishing, the project runs:

```bash
pnpm prepack
```

Which is (see `package.json` → `scripts.prepack`):

```bash
pnpm build && pnpm ui:build
```

So a full self-build matching “publish” is:

```bash
pnpm install
pnpm build
pnpm ui:build
```

Then the contents under `dist/`, `openclaw.mjs`, and everything listed in `package.json` → `files` form the built package.

### 6. (Optional) Build native apps

- **macOS**: From repo root, `pnpm mac:package` or `bash scripts/package-mac-app.sh`; or build the Swift package: `swift build --package-path apps/macos --configuration release`.
- **iOS**: `pnpm ios:gen` then open the Xcode project and build, or use `pnpm ios:build` / `pnpm ios:run` (see `package.json`).
- **Android**: `pnpm android:assemble` or `cd apps/android && ./gradlew :app:assembleDebug`.

---

## Quick reference

| Goal              | Command(s)                                      |
|-------------------|--------------------------------------------------|
| Install deps      | `pnpm install`                                  |
| Build core only   | `pnpm build`                                    |
| Build + UI        | `pnpm build && pnpm ui:build` or `pnpm prepack` |
| Run CLI           | `pnpm openclaw` or `pnpm start`                 |
| Run gateway dev   | `pnpm gateway:dev`                              |
| Lint              | `pnpm lint`                                     |
| Format check      | `pnpm format`                                   |
| Tests             | `pnpm test` (after `pnpm canvas:a2ui:bundle` if needed) |

---

## CI alignment

- **Install**: `pnpm install --frozen-lockfile` (with submodules updated as in `.github/workflows/ci.yml`).
- **Checks**: `pnpm build && pnpm lint`, `pnpm canvas:a2ui:bundle && pnpm test`, `pnpm format`, `pnpm protocol:check`.
- **macOS app**: Lint (SwiftLint/SwiftFormat), then `swift build --package-path apps/macos --configuration release` and `swift test`.
- **Android**: Java 21 + Android SDK; from `apps/android`: `./gradlew :app:assembleDebug` and `:app:testDebugUnitTest`.

Using the steps above you can replicate the core and UI build on your machine; add submodule init and the same frozen install if you want to match CI exactly.
