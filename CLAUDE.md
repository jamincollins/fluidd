# Fluidd AI Development Guide

Fluidd is a Vue 2.7 + TypeScript web interface for Klipper 3D printers that communicates with Moonraker via WebSocket.

## Architecture Overview

- **Vue 2.7 + Vuetify 2**: UI framework with Material Design components
- **Vuex Store**: 30 namespaced modules mirroring Klipper/Moonraker domains (`printer/`, `files/`, `console/`, `macros/`, `webcams/`, `mmu/`, `spoolman/`, etc.)
- **WebSocket Communication**: Real-time JSON-RPC via custom `WebSocketClient` in `src/plugins/socketClient.ts`
- **Component Structure**: Class-style components with `vue-property-decorator`; mixins-based architecture with `StateMixin` providing common printer state access

## Key Patterns

### Component Architecture

All components use **class-style decorators** — no Options API or Composition API:

```typescript
// Standard component
@Component({ components: { /* ... */ } })
export default class MyComponent extends Vue {
  @Prop({ type: String, required: true })
  readonly label!: string

  @VModel({ type: Boolean })
  open?: boolean
}

// Component needing printer state — extend via Mixins()
@Component({ components: { /* ... */ } })
export default class PrinterWidget extends Mixins(StateMixin) {
  get klippyReady (): boolean {
    return this.$typedGetters['printer/getKlippyReady']
  }
}
```

**Available mixins** (`src/mixins/`): `StateMixin`, `FilesMixin`, `ServicesMixin`, `BrowserMixin`, `CameraMixin`, `ToolheadMixin`, `AfcMixin`, `MmuMixin`

### State Management

- Store modules in `src/store/` — each has `state.ts`, `getters.ts`, `mutations.ts`, `actions.ts`, `types.ts`
- Use `$typedState` and `$typedGetters` for type-safe store access (defined in `src/plugins/filters.ts`)
- Use `Vue.set()` for reactive dynamic state properties
- Module definition: `export const auth = { namespaced, state, getters, actions, mutations } satisfies Module<AuthState, RootState>`

### WebSocket Integration

- All printer communication through `SocketActions` in `src/api/socketActions.ts` (not direct HTTP)
- Pattern: `baseEmit<T>(method, { dispatch, wait, params })`
- Use `wait` parameter for UI loading states: `wait: Waits.onPrintStart`
- Wait constants defined in `src/globals.ts` (`Waits` object, 80+ operation types)
- Real-time updates handled via store mutations from socket events
- Auto-reconnect with configurable interval (`Globals.SOCKET_RETRY_DELAY`)

### Component Registration

- **Auto-imported** (no manual import needed): components in `src/components/common/`, `layout/`, `ui/` — via `unplugin-vue-components` with `VuetifyResolver`
- **Manual import** required: widget components, view components
- **Lazy-loaded**: `EChart` via `Vue.component('EChart', () => import('./vue-echarts-chunk'))`
- Generated types: `components.d.ts` at repo root

## Development Workflow

### Build Toolchain

- **Vite 8** — build tool and dev server
- **`@pedrolamas/plugin-vue2`** — Vue 2 SFC support for Vite
- **`unplugin-vue-components/rolldown`** — auto-imports components from `src/components/common|layout|ui`
- **`sass-embedded`** — SCSS preprocessor (variables auto-injected via `@/scss/variables`)
- **vitest v4** — unit test runner (jsdom environment)
- **`commit-and-tag-version`** — release versioning (`npm run release`)
- **ESLint flat config** (`eslint.config.mjs`) — enforced at dev time via `vite-plugin-checker` with `useFlatConfig: true`
- **`vite-plugin-checker`** — runs vue-tsc and ESLint during dev (disabled at build time)
- **`skott`** — circular dependency detection (`npm run circular-check`)

### Essential Commands

```bash
npm run bootstrap      # Install git hooks (after clone)
npm run dev            # Start development server (port 8080)
npm run build          # Production build
npm run type-check     # TypeScript validation (vue-tsc)
npm run lint           # ESLint with Vue/TS rules
npm run test           # Vitest unit tests
npm run circular-check # Check for circular dependencies
```

### File Organization

```
src/
├── api/                # HTTP (axios) and WebSocket (custom JSON-RPC) clients
├── components/
│   ├── common/         # Shared dialogs & status components (auto-imported)
│   ├── layout/         # App shell: AppBar, AppDrawer, etc. (auto-imported)
│   ├── settings/       # Settings page components
│   ├── ui/             # Reusable: AppBtn, AppDialog, AppChart (auto-imported)
│   └── widgets/        # Feature widgets: camera/, filesystem/, macros/, mmu/, etc.
├── directives/         # Custom Vue directives (v-safe-html for DOMPurify)
├── locales/            # i18n YAML files (24 languages)
├── mixins/             # Vue mixins (StateMixin, FilesMixin, etc.)
├── monaco/             # TextMate grammars and editor themes
├── plugins/            # Vue plugins (i18n, httpClient, socketClient, vuetify, filters)
├── router/             # Vue Router (hash mode) with auth guards
├── scss/               # Global styles and Vuetify variable overrides
├── store/              # 30 Vuex modules (printer, files, config, webcams, etc.)
├── types/              # UI-specific TypeScript types
├── typings/            # Global .d.ts declarations (Klipper, Moonraker namespaces)
├── util/               # Helper functions (30+)
├── views/              # Page components (Dashboard, Console, Jobs, etc.)
└── workers/            # Web Workers (parseGcode, mjpegStream, sandboxedEval)
```

### Router & Authentication

- Hash-based routing (`#/path`)
- JWT token auth with auto-refresh (axios interceptors)
- Key routes: `/`, `/console`, `/jobs`, `/tune`, `/diagnostics`, `/configure`, `/settings`

### Icons & Theming

- MDI icons via `@mdi/js` — mapped in `src/globals.ts` (`Icons` object, 150+ mappings)
- Usage: `<v-icon>{{ $globals.Icons.close }}</v-icon>`
- Vuetify theme with custom dark/light overrides in `src/scss/variables.scss`
- PWA support with service worker in `src/sw.ts` (Workbox, injectManifest strategy)

### Monaco Editor

- Setup in `src/components/widgets/filesystem/setupMonaco.ts`
- TextMate grammars (onigasm WASM) for `gcode`, `klipper-config`, `log` languages
- Custom CodeLens providers (links to Klipper docs from config sections)
- Document symbol + folding range providers for klipper-config and gcode
- Worker setup in `setupMonaco.features.ts` (editor, JSON, CSS workers)

## Integration Points

### Klipper/Moonraker Communication

- All printer commands via `SocketActions` methods
- Store updates from WebSocket events (not polling)
- File operations through Moonraker's file API (`src/store/files/`)
- HTTP client (`src/plugins/httpClient.ts`) for file uploads, auth tokens

### Component Communication

- Parent-child: Props down, events up
- Cross-component: Vuex store or `EventBus` (`src/eventBus.ts`)
- Flash messages: `EventBus.$emit(text, { timeout })` — displayed by `FlashMessage` component

### Dynamic Imports

- `import.meta.glob()` used in `src/dynamicImports.ts` for lazy-loading:
  - `I18nLocales` — locale YAML files
  - `MonacoLanguageImports` — TextMate grammars
  - `CameraComponents` — camera service Vue components

## Testing Conventions

- Unit tests in `src/util/__tests__/*.spec.ts` with Vitest + jsdom
- Global test functions (`describe`, `it`, `expect`) — `globals: true` in vitest config
- Setup file: `tests/unit/setup.ts`
- Time manipulation utility: `timeTravel(date, callback)` in `tests/unit/utils.ts`
- Parameterized tests: `it.each([...])` pattern
- Test store actions/mutations independently from UI

## Code Style

- Vue class-style components with `vue-property-decorator` (`@Component`, `@Prop`, `@VModel`, `Mixins()`)
- ESLint enforced: `neostandard` + `pluginVue.configs['flat/vue2-recommended']` + `pluginRegexp` + `@vue/eslint-config-typescript`
- `.editorconfig` rules: 2 spaces, LF line endings, UTF-8, trim trailing whitespace, max line 100 (code)
- camelCase for variables/methods, PascalCase for components
- Use `consola` for logging, not `console.log` (configured in `src/setupConsola.ts` — warn in prod, verbose in dev)
- Type imports: `import type { ... }` for types only (`verbatimModuleSyntax: true`)
- `satisfies` keyword for store module type checking
- Conventional commits required: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`, `types`, `i18n`

## Common Gotchas

- Vue 2.7 limitations: no Composition API in production builds
- WebSocket reconnection handled automatically by `socketClient.ts`
- File uploads use FormData with progress tracking in store
- Dynamic imports for code splitting (see `vue-echarts-chunk.ts`, `src/dynamicImports.ts`)
- SCSS deprecation warnings silenced: `import`, `global-builtin`, `slash-div`, `if-function`
- `@/scss/variables` auto-injected into all SCSS/Sass files via Vite config
- `path` aliased to `path-browserify` for browser compatibility
- Strict Vuex mode enabled only in dev (`strict: import.meta.env.DEV`)

## Communication Style

- Be extremely concise in responses
- Sacrifice grammar for brevity
- Focus on essential info only
