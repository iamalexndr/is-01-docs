# Excalidraw Developer Onboarding

## What is Excalidraw?

Excalidraw is an open-source virtual whiteboard for sketching diagrams, wireframes, and ideas in a hand-drawn aesthetic. It runs entirely in the browser, works offline as a PWA, and supports real-time collaboration with end-to-end encryption.

The project serves two purposes at once: it's a **publishable React component library** (`@excalidraw/excalidraw` on npm) that other products can embed, and it's the full **excalidraw.com web application** — both living in the same monorepo. This dual nature shapes almost every structural decision in the codebase.

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | React 19, TypeScript 5.9 (strict) |
| State | Jotai (atomic state) |
| Canvas | Canvas API + RoughJS (hand-drawn style) |
| Styling | SASS/SCSS |
| Build (app) | Vite 5 |
| Build (packages) | esbuild |
| Testing | Vitest + @testing-library/react |
| Linting | ESLint + Prettier |
| Collaboration | Socket.io + Firebase |
| Monorepo | Yarn Workspaces |

---

## Repository Structure

The repo is cleanly split between the library packages and the application:

```
excalidraw/
├── packages/
│   ├── excalidraw/        # @excalidraw/excalidraw — the npm library (main package)
│   ├── common/            # @excalidraw/common — shared constants, color utils
│   ├── element/           # @excalidraw/element — shape types, geometry, mutations
│   ├── math/              # @excalidraw/math — 2D math, vectors, transforms
│   └── utils/             # @excalidraw/utils — clipboard, file export/import
├── excalidraw-app/        # The excalidraw.com application
│   ├── App.tsx            # App-level wrapper
│   ├── collab/            # Real-time collaboration
│   └── data/              # Firebase, local storage, file management
├── examples/              # Integration examples (Next.js, plain HTML)
├── scripts/               # Build and release scripts
└── dev-docs/              # Developer documentation
```

The most important mental model: **`packages/` is the library, `excalidraw-app/` is the product**. Any feature that belongs in the editor itself — a new tool, a rendering fix, a new element type — lives under `packages/`. Anything that's specific to excalidraw.com — collaboration, Firebase, PWA registration, analytics — lives in `excalidraw-app/`. When in doubt, ask whether the change would make sense if someone embedded the library in their own app. If yes, it belongs in `packages/`.

The packages form a layered dependency chain. Lower-level packages know nothing about higher-level ones:

```
@excalidraw/excalidraw        ← main library, depends on everything below
  ├── @excalidraw/common      ← foundational constants and utilities
  ├── @excalidraw/element     ← shape types and geometry
  │     ├── @excalidraw/math  ← pure 2D math
  │     └── @excalidraw/common
  └── @excalidraw/utils       ← export/import, clipboard
```

---

## Architecture

### State Management

The editor's entire runtime state — the active tool, zoom level, pan offset, selected elements, current stroke color, whether the grid is on, and dozens of other properties — lives in a single `AppState` object defined in [packages/excalidraw/types.ts](packages/excalidraw/types.ts). This object is exposed as Jotai atoms, not passed through props or stored in a Redux slice.

The two relevant store files are:
- [packages/excalidraw/editor-jotai.ts](packages/excalidraw/editor-jotai.ts) — atoms for the library-level editor state
- [excalidraw-app/app-jotai.ts](excalidraw-app/app-jotai.ts) — atoms for app-specific state like collaboration and theme

Jotai was chosen over Redux because it allows fine-grained subscriptions — a component that only cares about the zoom level won't re-render when the stroke color changes.

### Rendering

The canvas is rendered in two separate layers, both in [packages/excalidraw/scene/](packages/excalidraw/scene/):

**[interactiveScene.ts](packages/excalidraw/scene/interactiveScene.ts)** handles the live canvas the user interacts with. It redraws on every state change and includes selection handles, hover states, and in-progress drawing feedback.

**[staticScene.ts](packages/excalidraw/scene/staticScene.ts)** produces a deterministic, interaction-free render of the elements. This is what gets used when exporting to PNG. **[staticSvgScene.ts](packages/excalidraw/scene/staticSvgScene.ts)** does the same for SVG exports.

RoughJS sits between the canvas API and the element geometry — it intercepts draw calls and adds the hand-drawn jitter. This means the aesthetic is applied uniformly across all shape types without each one needing to implement it.

### Element System

Every shape on the canvas is an *element* — a plain TypeScript object with a type, geometry, style properties, and a version number. Elements are **immutable**: you never mutate one in place. Any change produces a new object with an incremented version. This constraint is load-bearing — the version number is how the collaboration layer detects and resolves conflicts, and how the undo/redo history tracks changes.

Element types (`rectangle`, `diamond`, `ellipse`, `arrow`, `line`, `freedraw`, `text`, `image`, `frame`, `embeddable`) and their shared and specific properties are all defined in [packages/element/types.ts](packages/element/types.ts).

### Actions System

Rather than scattering editor logic across components, all discrete editor operations are defined as *actions* in [packages/excalidraw/actions/](packages/excalidraw/actions/). Each action file (e.g. `actionAlign.tsx`, `actionClipboard.tsx`) declares what it does, what keyboard shortcut triggers it, where it appears in menus, and the handler function that executes it. This makes it straightforward to add a new editor operation without touching component code — you define the action and register it.

### Data Flow

Understanding how a user interaction becomes a canvas update is useful early on:

```
User interaction (click, keypress, drag)
  → Action handler  (packages/excalidraw/actions/)
    → Jotai atom update  (AppState and/or elements array)
      → React re-render
        → Canvas redraw  (interactiveScene.ts)
```

Saving a drawing serializes the elements and AppState to compressed JSON via [data/encode.ts](packages/excalidraw/data/encode.ts), producing a `.excalidraw` file. Loading runs the reverse through [data/restore.ts](packages/excalidraw/data/restore.ts), which also handles schema migrations so old files continue to open correctly.

---

## First Steps

### 1. Prerequisites

You need **Node.js >= 18.0.0** and **Yarn**. If you don't have Yarn, the easiest way is via corepack which ships with Node:

```bash
corepack enable
```

Verify your setup:

```bash
node --version   # should be >= 18.0.0
yarn --version   # should print a version number
```

### 2. Clone and install

```bash
git clone https://github.com/excalidraw/excalidraw.git
cd excalidraw
yarn install
```

`yarn install` from the root installs dependencies for all workspaces at once. You don't need to `cd` into individual packages — Yarn Workspaces handles everything from the root. This step also sets up the symlinks between packages so that, for example, `excalidraw-app` can import `@excalidraw/excalidraw` directly from source without a build step.

### 3. Start the dev server

```bash
yarn start
# App runs at http://localhost:3000
```

This starts the `excalidraw-app` Vite dev server with hot module replacement. Because the packages are symlinked, any change you make in `packages/excalidraw/` or any other package is reflected in the browser immediately — no separate package build step needed during development.

### 4. Explore the codebase

A good starting path through the code:

1. [packages/excalidraw/index.tsx](packages/excalidraw/index.tsx) — see what the library exposes publicly
2. [packages/excalidraw/types.ts](packages/excalidraw/types.ts) — read `AppState` and `ExcalidrawProps` to understand what drives the editor
3. [packages/excalidraw/components/App.tsx](packages/excalidraw/components/App.tsx) — the main editor component where most interaction logic lives
4. [packages/excalidraw/actions/](packages/excalidraw/actions/) — pick any action file to see the pattern for adding editor operations

### 5. Make a change and verify it

Before touching anything real, do a quick smoke test to confirm your environment works end-to-end:

```bash
yarn test:typecheck    # TypeScript compiles cleanly
yarn test:update       # All tests pass (updates snapshots if needed)
yarn fix               # No formatting or lint issues
```

All three should exit cleanly. If they do, your setup is correct and you're ready to contribute. Run the same three commands before every commit — the CI enforces all of them on every PR.

---

## Development Commands

```bash
yarn start              # Dev server for excalidraw-app (localhost:3000)
yarn test:typecheck     # TypeScript compilation check
yarn test:update        # Run all tests, update snapshots
yarn test:app           # Run Vitest in watch mode
yarn fix                # Auto-fix ESLint + Prettier issues
yarn build              # Production build of excalidraw-app
yarn build:packages     # Compile all library packages with esbuild
```

**Before every commit:**
```bash
yarn test:typecheck && yarn test:update && yarn fix
```

---

## Key Files to Know

| File | What it does |
|---|---|
| [packages/excalidraw/index.tsx](packages/excalidraw/index.tsx) | Library entry point — exports the `<Excalidraw>` component |
| [packages/excalidraw/types.ts](packages/excalidraw/types.ts) | Defines `AppState`, `ExcalidrawProps`, and all core types |
| [packages/excalidraw/components/App.tsx](packages/excalidraw/components/App.tsx) | The main editor component — most interaction logic lives here |
| [packages/excalidraw/editor-jotai.ts](packages/excalidraw/editor-jotai.ts) | Jotai store setup for the library |
| [packages/excalidraw/scene/interactiveScene.ts](packages/excalidraw/scene/interactiveScene.ts) | Live canvas rendering |
| [packages/excalidraw/data/restore.ts](packages/excalidraw/data/restore.ts) | Deserializes and migrates saved drawings |
| [packages/element/types.ts](packages/element/types.ts) | All element type definitions |
| [excalidraw-app/App.tsx](excalidraw-app/App.tsx) | Application-level wrapper around the library |
| [excalidraw-app/collab/Collab.tsx](excalidraw-app/collab/Collab.tsx) | Real-time collaboration logic |

---

## Conventions

**Imports:** Always use package aliases rather than relative paths across package boundaries — `import { ... } from "@excalidraw/common"`, not `../../common/src/...`. Use `import type { ... }` for TypeScript-only imports (ESLint enforces this). Import order goes: builtins → external packages → `@excalidraw/*` packages → parent directories → siblings.

**Do not import directly from `jotai`** — always go through `editor-jotai` or `app-jotai`. These wrappers configure the correct store instance.

**Naming:** React components use `PascalCase.tsx`, utility modules use `camelCase.ts`, and test files sit alongside their source as `FileName.test.ts`.

**Elements are immutable.** Never mutate an element object in place. Always create a new object and increment its `version` field. Code that violates this will subtly break undo/redo and collaboration.

---

## CI Checks

All of these must pass before a PR can merge:

| Check | What it runs |
|---|---|
| TypeScript | `yarn test:typecheck` |
| Lint | `yarn test:code` (ESLint) |
| Formatting | `yarn test:other` (Prettier) |
| Tests | `yarn test:app` (Vitest) |
| Bundle size | size-limit action — flags regressions automatically |
| PR title | must follow the semantic commit format (e.g. `feat:`, `fix:`) |

---

## Getting Help

- **Docs:** https://docs.excalidraw.com
- **Contributing guide:** https://docs.excalidraw.com/docs/introduction/contributing
- **Discord:** https://discord.gg/UexuTaE
- **Issues:** https://github.com/excalidraw/excalidraw/issues
