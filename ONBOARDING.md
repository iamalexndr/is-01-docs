# Excalidraw Developer Onboarding

## What is Excalidraw?

Excalidraw is an open-source virtual whiteboard for creating hand-drawn style diagrams, wireframes, and sketches. It's designed to be collaborative, end-to-end encrypted, and works offline as a PWA.

The project is both a **React component library** (`@excalidraw/excalidraw` on npm) and a **full web application** (excalidraw.com) — they live in the same monorepo.

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

**Rule of thumb:** If it's a general editor feature, work in `packages/`. If it's specific to excalidraw.com, work in `excalidraw-app/`.

**Package dependency order:**
```
@excalidraw/excalidraw
  ├── @excalidraw/common
  ├── @excalidraw/element
  │     ├── @excalidraw/math
  │     └── @excalidraw/common
  └── @excalidraw/utils
```

---

## Architecture

### State Management

State is managed with **Jotai** (atomic). There is no Redux or Context for editor state.

- `packages/excalidraw/editor-jotai.ts` — editor atoms (library-level)
- `excalidraw-app/app-jotai.ts` — app atoms (collab, theme, etc.)
- The central `AppState` type lives in `packages/excalidraw/types.ts` and holds all UI + canvas state (tool, zoom, pan, selections, drawing properties, etc.)

### Rendering

Two-layer canvas rendering in `packages/excalidraw/scene/`:

- **`interactiveScene.ts`** — live canvas with interactions (what the user sees while drawing)
- **`staticScene.ts`** — deterministic output used for export/snapshots
- **`staticSvgScene.ts`** — SVG export

RoughJS applies the hand-drawn aesthetic on top of the canvas API.

### Element System

Elements (rectangles, arrows, text, images, etc.) are **immutable objects** with version numbers. Mutations always produce new element instances — this is critical for collaborative conflict resolution and undo/redo.

Types are defined in `packages/element/types.ts`. Supported element types: `rectangle`, `diamond`, `ellipse`, `arrow`, `line`, `freedraw`, `text`, `image`, `frame`, `embeddable`.

### Actions System

Editor operations (copy, paste, align, delete, etc.) are declarative actions defined in `packages/excalidraw/actions/`. Each action file follows the pattern `action[FeatureName].ts(x)` and declares its keyboard shortcut, menu binding, and handler.

### Data Flow

```
User interaction
  → Action handler (packages/excalidraw/actions/)
    → Jotai atom update (AppState / elements)
      → React re-render
        → Canvas redraw (interactiveScene.ts)
```

Serialization (`data/encode.ts`) produces compressed JSON (`.excalidraw` files). Deserialization (`data/restore.ts`) handles schema migrations across versions.

---

## First Steps

### 1. Prerequisites

- Node.js >= 18.0.0
- Yarn (installed globally or via corepack)

### 2. Setup

```bash
git clone https://github.com/excalidraw/excalidraw.git
cd excalidraw
yarn install
```

### 3. Start the dev server

```bash
yarn start
# App runs at http://localhost:3000
```

### 4. Run tests

```bash
yarn test:typecheck    # TypeScript check
yarn test:update       # Run all tests (updates snapshots)
yarn fix               # Auto-fix formatting and lint
```

---

## Development Commands

```bash
yarn start              # Dev server (excalidraw-app)
yarn test:typecheck     # TypeScript compilation check
yarn test:update        # Run all tests with snapshot updates
yarn test:app           # Run Vitest in watch mode
yarn fix                # Auto-fix ESLint + Prettier
yarn build              # Production build (excalidraw-app)
yarn build:packages     # Build all library packages
```

**Before every commit:**
```bash
yarn test:typecheck && yarn test:update && yarn fix
```

---

## Key Files to Know

| File | Purpose |
|---|---|
| `packages/excalidraw/index.tsx` | Library entry point, `<Excalidraw>` component |
| `packages/excalidraw/types.ts` | Core types: `AppState`, `ExcalidrawProps` |
| `packages/excalidraw/components/App.tsx` | Main editor component |
| `packages/excalidraw/editor-jotai.ts` | Jotai store setup |
| `packages/excalidraw/scene/interactiveScene.ts` | Live canvas rendering |
| `packages/excalidraw/data/restore.ts` | Deserialize / migrate drawings |
| `packages/element/types.ts` | Element type definitions |
| `excalidraw-app/App.tsx` | Application wrapper |
| `excalidraw-app/collab/Collab.tsx` | Collaboration logic |

---

## Conventions

**Imports:**
- Use package aliases: `import { ... } from "@excalidraw/common"`
- `import type { ... }` for TypeScript-only imports (ESLint enforced)
- Import order: builtin → external → `@excalidraw/*` → parent → sibling

**Naming:**
- Components: `PascalCase.tsx`
- Utilities: `camelCase.ts`
- Tests: `FileName.test.ts` alongside source files

**Elements are immutable** — never mutate an element in place. Always create a new object with an incremented version.

**Do not import directly from `jotai`** — use `editor-jotai` or `app-jotai` wrappers.

---

## CI Checks (must pass on PRs)

| Check | Command |
|---|---|
| TypeScript | `yarn test:typecheck` |
| Lint | `yarn test:code` |
| Formatting | `yarn test:other` |
| Tests | `yarn test:app` |
| Bundle size | automated via size-limit action |
| PR title | must follow semantic commit format |

---

## Getting Help

- **Docs:** https://docs.excalidraw.com
- **Contributing guide:** https://docs.excalidraw.com/docs/introduction/contributing
- **Discord:** https://discord.gg/UexuTaE
- **Issues:** https://github.com/excalidraw/excalidraw/issues
