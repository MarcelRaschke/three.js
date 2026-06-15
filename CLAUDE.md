# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development server (port 8080, hot reload via passthrough build files)
npm start

# Build production bundles (Rollup → build/)
npm run build

# Run all checks (lint + unit tests + addon tests)
npm test

# Lint core source only (src/)
npm run lint

# Auto-fix lint issues across all areas (src, examples/jsm, editor, test, utils)
npm run lint-fix

# Unit tests (headless Puppeteer / QUnit)
npm run test-unit
npm run test-unit-addons

# Unit tests with browser window (no auto-reload on change; refresh manually)
npm run test-unit-headful

# E2E tests (~30 min, requires Chromium; CI uses xvfb-run -a npm run test-e2e)
npm run test-e2e
```

**Running a single unit test:** Start the dev server (`npm start`), then open `http://localhost:8080/test/unit/UnitTests.html` in a browser. Tests are QUnit-based and run in the browser. You can filter by module/test name using QUnit's built-in UI.

**Do not commit files in `build/`** — these are generated artifacts.

## Architecture

### Entry Points

The library has multiple entry points depending on renderer:

| File | Purpose |
|---|---|
| `src/Three.js` | Main export (Three.Core.js + WebGLRenderer + ShaderLib/UniformsLib/PMREMGenerator) |
| `src/Three.Core.js` | All non-renderer classes (~165 exports) |
| `src/Three.WebGPU.js` | WebGPU renderer entry |
| `src/Three.TSL.js` | Texture Shading Language (node-based shader authoring) |

### Core Class Hierarchy

- **`EventDispatcher`** — base mixin for all event-emitting objects
- **`Object3D`** (`src/core/Object3D.js`, ~40KB) — base for every scene node; owns the transform hierarchy, `add`/`remove`, matrix updates, and events (`added`, `removed`, `childadded`)
- **`BufferGeometry`** (`src/core/BufferGeometry.js`, ~34KB) — stores vertex/index data as typed array buffers via `BufferAttribute`
- **`Material`** subclasses in `src/materials/` — each wraps a shader; node-based variants live in `src/materials/nodes/`
- **`Camera`** → `PerspectiveCamera` / `OrthographicCamera` — extend `Object3D`; produce view/projection matrices
- **`WebGLRenderer`** (`src/renderers/WebGLRenderer.js`, ~108KB) — delegates to 18 sub-modules in `src/renderers/webgl/` (Attributes, Background, Binding, Capabilities, Extensions, Geometries, Info, Lights, Materials, Morphtargets, Objects, Output, Programs, Properties, RenderLists, RenderStates, State, Textures, Uniforms)

### Shader System

- `src/renderers/shaders/ShaderLib/` — per-material GLSL shaders (standard, phong, depth, etc.)
- `src/renderers/shaders/ShaderChunk/` — reusable GLSL snippets included by `#include <chunkName>`
- `src/renderers/shaders/UniformsLib.js` — shared uniform blocks (lights, fog, etc.)
- Custom shaders: use `ShaderMaterial` (includes chunks) or `RawShaderMaterial` (no injection)

### Node / TSL System

`src/nodes/` (19 subdirectories) implements the Texture Shading Language — a tree of node objects that compile to GLSL or WGSL. Used by WebGPU renderer and node-based materials in `src/materials/nodes/`. Key subdirs: `core/`, `lighting/`, `math/`, `geometry/`, `gpgpu/`, `procedural/`, `materialx/`.

### Addons (`examples/jsm/`)

36 directories of optional modules (loaders, controls, postprocessing, geometries, etc.) that are not part of the core bundle. These are published via the `three/addons/` package export and linted separately (`npm run lint-addons`).

### Build System

`npm run dev` writes passthrough re-export files to `build/` pointing at `src/` — enabling instant hot reload without a full Rollup compile. `npm run build` runs Rollup (config: `utils/build/rollup.config.js`) with a custom `glsl()` plugin (bundles `*.glsl.js` files inline) and Terser for `.min.js` variants.

## Code Conventions

- **Indentation:** tabs (enforced by `.editorconfig` and ESLint)
- **Quotes:** single quotes only
- **`const` preferred** over `let`/`var` wherever possible
- **JSDoc required** on public APIs — types must be specified for parameters and return values
- **No unused variables** (caught errors are the only exception)
- **ECMAScript 2022**, module mode (`type: "module"` in `package.json`)
- GLSL shader files use the `.glsl.js` extension (exported as template literals)

## Testing Conventions

Unit tests mirror the `src/` structure under `test/unit/src/` and use **QUnit** (`QUnit.module()`, `QUnit.test()`). The HTML harnesses (`test/unit/UnitTests.html`, `test/unit/UnitTestsAddons.html`) load tests via Puppeteer for CI or directly in a browser for development.

E2E tests in `test/e2e/` render actual examples with Puppeteer+Chromium and compare screenshots. CI parallelizes them across 4 workers; locally they require ~200MB Chromium download.
