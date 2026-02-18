# Copilot Instructions

## Build, test, and lint commands

- Clone recursively: the repository includes Git submodules that host handler dependencies, so always run `git clone --recursive https://github.com/p2r3/convert`.
- Install Bun dependencies with `bun install`.
- Start the dev server with `bunx dev` (runs `vite`) or `bunx vite`.
- Build for production with `bunx build` (runs `tsc && vite build`) and preview the production bundle with `bunx preview`.
- Generate the supported-format cache after building by running `bun buildCache.js [output.json]` (the dist folder must exist). The script launches Puppeteer, waits for `printSupportedFormatCache()` to emit, and writes `cache.json` so `src/main.ts` can skip the slow format enumeration on startup.
- Docker compose runs:
  - `docker compose -f docker/docker-compose.yml up -d` (prebuilt image).
  - `docker compose -f docker/docker-compose.yml -f docker/docker-compose.override.yml up --build -d` (local build for development).
- There are no dedicated unit-test or linter scripts; `bunx build` provides TypeScript checking, and manual end-to-end validation happens through the browser UI.

## High-level architecture

- `src/main.ts` wires the single-page UI: it manages file selection, search filters, simple/advanced mode, and conversion button state, and it exposes globals (`window.supportedFormatCache`, `window.traversionGraph`, `window.printSupportedFormatCache`, `showPopup`, `hidePopup`) needed by other tooling (such as `buildCache.js`).
- Conversion capabilities are registered in `src/handlers/index.ts`, which imports every handler implementation (canvas, FFmpeg, ImageMagick, zip utilities, audio/midi, text serializers, etc.), wraps their instantiation in try/catch to degrade gracefully, and exports the resulting array consumed by the UI.
- Each handler implements `FormatHandler` from `src/FormatHandler.ts`, describing supported `FileFormat`s via `FormatDefinition`/`CommonFormats` builders, exposing `init()`/`doConvert()` and optionally `supportAnyInput`. The main thread calls every handler as needed: it initializes missing ones, caches their `supportedFormats`, and invokes `doConvert()` while traversing a conversion path.
- The conversion planner is `src/TraversionGraph.ts`: it builds a directed graph where nodes are MIME types and edges represent handler transitions. Cost heuristics (depth penalty, category-change costs, handler priority, lossy multipliers) feed a Dijkstra search (via `PriorityQueue`) to find reasonable multi-step routes between formats, and `main.ts` iterates over those paths until a handler chain succeeds.
- Format metadata is cached in `window.supportedFormatCache` and `cache.json`; `main.ts` first tries to `fetch("cache.json")`, falls back to running every handler if the cache is missing, then calls `window.traversionGraph.init()` to build the search graph. `buildCache.js` reproduces that flow headlessly so you can ship a precalculated `cache.json` (call `printSupportedFormatCache()` in the browser to inspect the structure).

## Key conventions

- Handler naming: the class is named `{tool}Handler`, the file is `{tool}.ts`, and the handler’s `supportedFormats` should declare formats through `FormatDefinition.builder()` so formats stay consistent with `CommonFormats`.
- When implementing `doConvert()`, never mutate the `FileData.bytes` buffer in place. Clone inputs via `new Uint8Array(bytes)` if you have to adjust them, and return newly constructed `FileData` objects so multiple handlers can reuse the same source safely.
- Keep MIME matching kosher: run MIME strings through `normalizeMimeType(src/normalizeMimeType.ts)` so handler registrations align with the `filterButtonList` lookups in `main.ts`.
- Register every new handler in `src/handlers/index.ts` (the list is iterated in priority order). Handlers may throw during import, so wrap constructions in `try/catch` the same way the index does; the UI ignores handlers that fail to load.
- The format explorer exposes two modes, simple and advanced, toggled via `ui.modeToggleButton`. Simple mode deduplicates display entries by MIME+format, while advanced mode keeps handler-specific names visible.
- When you add a dependency with a WebAssembly binary, mention it in `vite.config.js` to expose it under `/convert/wasm/` (linking directly to `node_modules` is disallowed).
- Use the `window.printSupportedFormatCache()` helper after the format list is built (see the console log `Built initial format list.`). Saving that JSON to `cache.json` and committing it reduces first-load delay, and `src/main.ts` will consume it automatically.
