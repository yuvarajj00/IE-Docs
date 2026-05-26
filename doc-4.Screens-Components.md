# Mapping Studio UI — Screen-by-Screen Component Map

> One narrative entry per route. For each screen you'll see exactly **which components it uses, why, and how they fit together** — including the dialogs that open from it, the services it calls, and the journey to/from other screens.

---

## How Every Screen is Wrapped

Before walking through individual screens, here's the global frame that wraps everything (`app.html`):

```
┌──────────────────────────────────────────────────────────┐
│  App  (root component)                                   │
│  ┌──────────┬─────────────────────────────────────────┐  │
│  │          │                                         │  │
│  │ Sidebar  │  <router-outlet>  ← the screen renders  │  │
│  │ (maybe)  │       here                              │  │
│  │          │                                         │  │
│  └──────────┴─────────────────────────────────────────┘  │
│  <router-outlet name="modal">  ← overlay screens render here
└──────────────────────────────────────────────────────────┘
```

The **Sidebar** is *conditional*. The `App` component checks the current URL and renders Sidebar only for the four "global layout" screens:

```ts
// app.ts
const GLOBAL_LAYOUT_ROUTES = ['/dashboard', '/system', '/crosswalk', '/template-builder'];
showSidebar = computed(() => GLOBAL_LAYOUT_ROUTES.some(r => this.currentUrl().startsWith(r)));
```

So when we say "the screen uses Sidebar," that simply means the URL starts with one of those four. The wizard and studio screens deliberately hide it to get full-width canvas room.

---

## Screen 1 — `/dashboard`

> The landing page. Lists templates, shows quick stats.

When you visit `/dashboard`:

- The screen **uses `Sidebar`** to display the left side-nav rail (Dashboard, System, Crosswalk, Templates links).
- The main area is filled by **`DashboardComponent`**, the screen's host.

`DashboardComponent` builds its UI from these pieces:

- A Material **`mat-toolbar`** at the top for the title, search box, "New Template" button, and "Import" button.
- A row of Material **`mat-card`** stat tiles (Templates / Systems / Crosswalks / Rule Sets) — each tile is clickable and routes to its corresponding screen.
- A Material **`mat-progress-bar`** that shows while the data loads.
- A table of templates, where each row is rendered inline (no separate card component for this — the dashboard does it in its own template).

When the user clicks the trash icon on a row, the dashboard **opens `DeleteTemplateDialogComponent`** (via `MatDialog`). That dialog returns `true` or `false`; if true, the dashboard calls `DashboardService.deleteProject(id)` and removes the row.

### What the screen calls

`DashboardComponent` injects **`DashboardService`** for everything network-related. On load, `DashboardService.getDashboardData()` runs three HTTP requests in parallel (`forkJoin`) — one to `/system`, one to `/crosswalk`, one to `/template` — and merges the results into one tidy object containing stats + the first 10 templates.

### What happens when you click around

| Click | What happens |
|---|---|
| **New Template** button | Navigates to `/templates/new/schema-setup` (Screen 7). |
| **Import** button | Opens a file picker (no separate component). |
| A **stat tile** (Systems / Crosswalks) | Navigates to that screen (`/system` or `/crosswalk`). |
| A **template row** | Navigates to `/templates/{id}/studio?edit=true` (Screen 8 in edit mode). |
| The **pencil** icon | Same as clicking the row — opens the studio in edit mode. |
| The **copy** icon | POSTs to `/template/{id}/duplicate`, prepends the new row to the list. |
| The **download** icon | Streams a zip from `/template/{id}/export` and triggers a browser download. |
| The **trash** icon | Opens `DeleteTemplateDialogComponent` → on confirm, deletes. |

### Component tree on this screen

```
App
├── Sidebar
└── <router-outlet>
    └── DashboardComponent
        ├── mat-toolbar (top bar)
        ├── mat-card × 4    (stat tiles)
        ├── mat-progress-bar (while loading)
        └── (a table of templates rendered inline)
            └── [on delete click] DeleteTemplateDialogComponent  (in MatDialog overlay)
```

---

## Screen 2 — `/system`

> Manage source/target systems (Salesforce, Legacy DB, FHIR R4, etc.).

When you visit `/system`:

- The screen **uses `Sidebar`** for left navigation.
- The main area is filled by the **`System`** component (yes, the class is just called `System` — note this collides with the `System` interface from `system.service.ts`; consumers alias it as `SystemModel`).

`System` builds its UI from:

- A top toolbar with the page title and a **"Register System"** button.
- A grid layout that renders **one `SystemCardComponent` per system** (6 cards per page).
- A manual pagination control (Prev / 1 / 2 / 3 / Next) at the bottom.

Each `SystemCardComponent` is a small presentational component. It accepts a `system` input and emits two events: `edit` and `delete`. The parent listens for these to decide what dialog to open.

When the user:

- Clicks **"Register System"** → `System` **opens `RegisterSystemDialogComponent`** in create mode.
- Clicks the **edit** icon on a card → `System` opens the same `RegisterSystemDialogComponent`, but in edit mode (passing the existing system data).
- Clicks the **delete** icon → `System` **opens `DeleteSystemDialogComponent`**.

`RegisterSystemDialogComponent` is one of the most polished components in the codebase. It uses Angular reactive forms with an **async duplicate-name validator** that debounces input and hits the backend to check if `(name, type)` is already taken.

### What the screen calls

The screen injects **`SystemService`**. On load, it calls `getAllSystems()`, filters out soft-deleted ones, sorts by `updatedAt` descending, and slices for the current page. Creating uses `createSystem()`; editing uses `updateSystem()`; deleting uses `softDeleteSystem()` (no hard deletes).

### Component tree on this screen

```
App
├── Sidebar
└── <router-outlet>
    └── System
        ├── (top toolbar — inline)
        ├── SystemCardComponent × 6        (the grid)
        ├── (pagination — inline)
        └── (when buttons click)
            ├── RegisterSystemDialogComponent    (MatDialog overlay — create or edit)
            └── DeleteSystemDialogComponent      (MatDialog overlay)
```

---

## Screen 3 — `/crosswalk`

> Manage code-mapping lookup tables (ICD-9 → ICD-10, M/F → male/female, etc.).

When you visit `/crosswalk`:

- The screen **uses `Sidebar`** for left navigation.
- The main area is filled by **`CrosswalkTablesComponent`**.

The structure mirrors the System screen exactly — that's deliberate, the two have a near-identical UX.

- Top toolbar with title and **"Create Crosswalk"** button.
- A grid of **`CrosswalkCardComponent`** instances — one per crosswalk.
- Each card shows the crosswalk name, a count of how many mappings it contains, and edit/delete buttons.

When the user:

- Clicks **"Create Crosswalk"** → opens **`CrosswalkCreateDialogComponent`** in create mode.
- Clicks edit on a card → same dialog in edit mode.
- Clicks delete on a card → opens **`DeleteCrosswalkDialogComponent`**.

### A subtle thing this screen has to handle

The .NET backend sometimes returns property names in PascalCase (`Mappings`, `Source`, `Target`) and sometimes in camelCase, depending on the serializer version. The component normalizes both:

```ts
mappings: (c.mappings || c.Mappings || []).map((m: any) => ({
  source:        m.source        ?? m.Source        ?? '',
  target:        m.target        ?? m.Target        ?? '',
  targetDisplay: m.targetDisplay ?? m.TargetDisplay ?? undefined,
  targetSystem:  m.targetSystem  ?? m.TargetSystem  ?? undefined,
}))
```

Without these fallbacks, the page would silently render empty cards on a backend version mismatch.

### What the screen calls

It uses two services in parallel (`forkJoin`):
- **`CrosswalkService`** for `getAllCrosswalks`, `createCrosswalk`, `updateCrosswalk`, `deleteCrosswalk`.
- **`SystemService`** for the source/target system labels shown on each card.

### Component tree on this screen

```
App
├── Sidebar
└── <router-outlet>
    └── CrosswalkTablesComponent
        ├── (top toolbar — inline)
        ├── CrosswalkCardComponent × N
        └── (when buttons click)
            ├── CrosswalkCreateDialogComponent    (MatDialog overlay)
            └── DeleteCrosswalkDialogComponent    (MatDialog overlay)
```

---

## Screen 4 — `/template-builder`

> A standalone Scriban editor — write templates by hand instead of using the visual studio.

When you visit `/template-builder`:

- The screen **uses `Sidebar`** for left navigation (in the sidebar this link is labelled **"Templates"**).
- The main area is filled by **`TemplateBuilderComponent`**.

The screen is split into two columns:

- **Left column** — a big `<textarea>` where you type Scriban template code. Below it, a "Field Code Generator" form that builds a single field's Scriban from a structured UI (source path, target path, transformation type, etc.).
- **Right column** — a list of **snippet groups** (Crosswalk, Variables, Loops, Conditionals). Each snippet, when clicked, gets inserted at the current caret position in the textarea.

### Why this is a bit ugly

The snippet insertion uses **direct DOM manipulation** of the textarea via `@ViewChild`:

```ts
@ViewChild('templateEditor') templateEditor!: ElementRef;
insertSnippet(code: string): void {
  const ta = this.templateEditor.nativeElement;
  const start = ta.selectionStart;
  ta.value = ta.value.substring(0, start) + code + ta.value.substring(ta.selectionEnd);
  // …
}
```

This is the worst single anti-pattern in the codebase. The Refactoring Playbook proposes replacing it with a `ScribanEditorDirective` or swapping in Monaco/CodeMirror.

### What the screen calls

- **`CrosswalkService`** to populate the crosswalk picker (so snippets reference real crosswalk names).
- **`RuleSetService`** — but this is a deprecated stub that always returns `[]`, so the ruleset panel is always empty.

### What "Save Template" does

⚠ Currently nothing useful. The save button logs to console and shows a "✅ Template saved" snackbar, but doesn't actually persist anywhere. This is an unfinished feature.

### Component tree on this screen

```
App
├── Sidebar
└── <router-outlet>
    └── TemplateBuilderComponent
        ├── <textarea> (raw Scriban editor)
        ├── (Field Code Generator form — inline)
        └── (Snippet panel — inline groups of buttons)
```

No child components. The whole screen is inside one big template.

---

## Screen 5 — `/templates/new/schema-setup`

> **Wizard step 1** — name the template, pick systems, upload source + target schemas.

When you visit `/templates/new/schema-setup`:

- The Sidebar is **hidden** — this is a wizard screen, full-width canvas.
- The main area is filled by **`NewTemplateComponent`**.

`NewTemplateComponent` is the wizard host. It uses three components to build the screen:

- **`StepperComponent`** at the top — the three-step breadcrumb showing `[● SCHEMA SETUP] [○ MAPPING STUDIO] [○ EXPORT BUNDLE]`. Only the active step is clickable; this prevents users jumping ahead.
- **`TemplateMetadataFormComponent`** — the main form body (title, description, source/target system dropdowns, two schema upload buttons).
- A footer with Cancel and Continue buttons (inline in the wizard template).

### How the metadata form works

`TemplateMetadataFormComponent` doesn't manage its own state — it pushes every change into **`NewTemplateStateService`**, a small signal-based store shared between the wizard and the studio.

When the user clicks **"Upload Source Schema"**, the form **opens `SchemaUploadDialogComponent`**. That dialog has two modes:

1. **File mode** — drag-and-drop or browse to a `.json` / `.xml` file (max 10 MB). It uses the `parseSchemaFile` utility to validate and return an `UploadedSchema` object.
2. **FHIR mode** — search through the bundled FHIR R4 resource library (loaded from `docs/fhir.schema.json` via `FhirSchemaLoaderService`) and pick a resource type like Patient, Observation, Encounter.

The dialog returns the chosen schema; the form stashes it in `NewTemplateStateService`.

### When you click Continue

`NewTemplateComponent` checks `NewTemplateStateService.isValid()`. If valid:

```ts
const templateId = this.currentTemplateId || crypto.randomUUID();
this.router.navigate(['templates', templateId, 'studio']);
```

It generates a fresh UUID (or reuses the existing ID in edit mode) and navigates to the studio. The next screen reads the wizard state from `NewTemplateStateService` and starts the mapping.

### Component tree on this screen

```
App
└── <router-outlet>                          (no Sidebar)
    └── NewTemplateComponent
        ├── StepperComponent                  (wizard breadcrumb)
        ├── TemplateMetadataFormComponent
        │   └── [on upload click] SchemaUploadDialogComponent  (MatDialog overlay)
        └── (Cancel + Continue buttons — inline)
```

---

## Screen 6 — `/templates/:id/studio`

> **Wizard step 2** — the visual mapping studio. The biggest, busiest screen in the app.

When you visit `/templates/{id}/studio`:

- The Sidebar is **hidden** (this screen needs every pixel).
- The main area is filled by **`MappingStudioComponent`**, which is the orchestrator.

`MappingStudioComponent` is doing a lot. Visually the screen breaks into 4 regions:

```
┌────────────────────────────────────────────────────────────────────┐
│ TOP: StepperComponent (breadcrumb) + title + back button           │
├──────────────┬──────────────────────────────┬──────────────────────┤
│              │                              │                      │
│ LEFT:        │ CENTER:                      │ RIGHT:               │
│ SchemaTree   │  Bridge canvas — a list of   │ ValidationPreview    │
│ (source)     │  BridgeCardComponent × N     │ Component (the       │
│              │                              │ generated Scriban    │
│              │  Drop sources here to        │ template + bridge    │
│ (drag fields │  create bridges, see live    │ counts + version     │
│ from here)   │  status per bridge           │ panel)               │
│              │                              │                      │
├──────────────┴──────────────────────────────┤                      │
│ Continue to Export button                   │                      │
└─────────────────────────────────────────────┴──────────────────────┘
                                          (+ optional render-preview modal layered over the whole thing)
```

So the components in play are:

- **`StepperComponent`** — same breadcrumb as the wizard, but with step 2 active.
- **`SchemaTreeComponent` × 2** — one for the source schema, one for the target. Same component, different `mode` input (`'source'` or `'target'`). Source nodes are draggable; target nodes accept drops.
- **`BridgeCardComponent` × N** — one per bridge the user has created. Renders source → target paths, the chosen transform, status pill, and per-bridge actions (edit, delete, toggle, export-bundle).
- **`BridgeEditorComponent`** — a simpler legacy inline editor. Still imported, conditionally rendered, but largely superseded by the dialog below. (Candidate for deletion — see Refactoring Playbook.)
- **`FieldConnectionDialogComponent`** — the big 1,864-line dialog that opens whenever the user creates or edits a bridge. Hosts the transformation pattern picker, operator catalog, value-map rules, crosswalk picker, etc.
- **`ValidationPreviewComponent`** — the right panel. Shows valid/warning/invalid counts, the generated Scriban template with syntax highlighting, edit mode, saved versions, and the "Viva" preview tab.
- **`ConfirmDialogComponent`** — opens for destructive actions: "Delete bridge?", "Reset all bridges?", "Leave the studio with unsaved changes?".

### What happens when the user creates a bridge

This is the classic interaction:

1. User drags `Patient.firstName` from the **source** `SchemaTreeComponent`.
2. They drop it onto `name.given` in the **target** `SchemaTreeComponent`.
3. The target tree emits `fieldDropped({ sourcePath, targetPath })`.
4. `MappingStudioComponent` catches the event and **opens `FieldConnectionDialogComponent`**, passing both paths in.
5. The user picks a transform (or just leaves it as "direct"), maybe selects a crosswalk, clicks Save.
6. The dialog emits `commit(...)`.
7. `MappingStudioComponent.onConnectionCommit` runs, calling `studio.addBridge(...)` and `studio.updateBridge(...)` on the **`StudioStateService`** signal store.
8. The `bridges` signal updates → a new `BridgeCardComponent` appears in the canvas → `ValidationPreviewComponent` recounts the totals.

### What "Preview" does (and how the modal route is involved)

When the user clicks the Preview button:

```ts
this.router.navigate([{ outlets: { modal: ['render-preview'] } }]);
```

This activates the named outlet (Screen 7 below). The URL becomes `/templates/abc/studio(modal:render-preview)`. The studio doesn't unmount — `FhirRenderPreviewComponent` just renders **on top** of it.

### What the screen calls

It injects **`StudioStateService`** — the central signal store for the entire studio session. It also calls **`CrosswalkService`** at load to populate the dropdown that `FieldConnectionDialogComponent` uses.

### The four init branches

When the studio mounts, it has to figure out what state to load. It checks four conditions in order:

| Condition | What it does |
|---|---|
| `?edit=true` is in the URL | Calls `studio.loadFromBackend(id)` — full reload from the backend |
| `NewTemplateStateService` has wizard data | Initializes from that in-memory state (the user just came from the wizard) |
| Path has an `:id` but neither of the above | Starts a fresh empty studio with that ID |
| No ID at all (shouldn't happen) | Redirects to `/templates/new/schema-setup` |

### Component tree on this screen

```
App
└── <router-outlet>                              (no Sidebar)
    └── MappingStudioComponent
        ├── StepperComponent                       (wizard step 2)
        ├── SchemaTreeComponent (source side)
        ├── SchemaTreeComponent (target side)
        ├── BridgeCardComponent × N                (one per bridge)
        ├── BridgeEditorComponent                  (legacy, rarely shown)
        ├── ValidationPreviewComponent             (right panel)
        └── (when actions trigger)
            ├── FieldConnectionDialogComponent     (MatDialog overlay — bridge config)
            └── ConfirmDialogComponent             (MatDialog overlay — destructive ops)

App                                                (parallel — when "Preview" clicked)
└── <router-outlet name="modal">
    └── FhirRenderPreviewComponent                 (overlays the whole studio)
```

---

## Screen 7 — `(modal:render-preview)` (overlay route)

> An overlay that floats above the Mapping Studio. Lets the user test their template by feeding it real source data and seeing the rendered FHIR output.

This isn't a standalone page — it's a **named-outlet route**. It only ever appears layered on top of Screen 6.

When you trigger it (from inside the studio), the URL changes to:

```
/templates/abc-123/studio(modal:render-preview)
```

The part in parentheses is the named-outlet segment. Both outlets stay active simultaneously — the studio keeps rendering in the primary outlet, and **`FhirRenderPreviewComponent`** renders in the `modal` outlet.

`FhirRenderPreviewComponent` builds its UI as a two-pane modal:

- **Left pane** — an editable `<textarea>` containing the source JSON. It's seeded automatically from `StudioStateService.sourceRawSchema()`, or built from sample values if no raw schema exists.
- **Right pane** — read-only output showing what the backend rendered.

### How the live update works

The component listens for changes to the left pane via an `RxJS Subject` with `debounceTime(800)` and `distinctUntilChanged`. Whenever the user pauses typing for 800ms, it automatically re-renders:

```ts
this._sourceChange$
  .pipe(debounceTime(800), distinctUntilChanged())
  .subscribe(() => this.render());
```

`render()` POSTs to `/template/render-by-id` with the current source JSON and updates the right pane with the response.

### How to close it

Either Esc or the close button calls:

```ts
this.router.navigate([{ outlets: { modal: null } }]);
```

That clears the named outlet. The URL reverts to `/templates/abc-123/studio`. The studio is exactly as the user left it.

### Component tree on this screen

```
App
├── Sidebar  (hidden — primary outlet still shows the studio)
├── <router-outlet>
│   └── MappingStudioComponent  (unchanged, still rendering)
└── <router-outlet name="modal">
    └── FhirRenderPreviewComponent       (the floating overlay)
        ├── Left pane: editable source JSON textarea
        └── Right pane: rendered FHIR output
```

No child components — the whole overlay is one tightly-scoped component.

---

## Screen 8 — `/export-bundle`

> **Wizard step 3** — final review, downloads the bundle.

When you visit `/export-bundle`:

- The Sidebar is **hidden** (wizard screen).
- The main area is filled by **`ExportBundleComponent`**.

This is the simplest of the three wizard screens. It uses:

- **`StepperComponent`** at the top, now showing step 3 as active.
- A summary block (inline) listing what's about to be bundled: source schema, target schema, bridge count, crosswalk count, "schemas available" status.
- A big **"Download Bundle"** button.

### What "Download Bundle" actually does

When the user clicks the button, `ExportBundleComponent` calls **`ExportBundleService.downloadBundle()`**. That service does a lot of work in parallel:

1. Resolves system IDs to display names via `SystemService.getAllSystems()`.
2. Fetches crosswalks referenced by `codeMap` bridges via `CrosswalkService.getAllCrosswalks()`.
3. Fetches the backend's persisted template via `StudioStateService` (which calls `/template/{id}`).
4. Builds 4 JSON files in memory: `source-schema.json`, `target-schema.json`, `crosswalk.json`, `template.json`.
5. Zips them using the `fflate` library.
6. Triggers a browser download (`URL.createObjectURL` + `<a>.click()`).
7. After successful download, calls `studio.markAsCompleted()` which PUTs the template status to `COMPLETED`.

Filename is `{title}_{version}.zip`, e.g. `Patient_to_FHIR_v3.zip`.

### Safety net

`ExportBundleComponent.ngOnInit` checks that there's actually an active studio session:

```ts
if (!this.studio.projectId()) {
  this.router.navigate(['/templates/new/schema-setup']);
}
```

If someone hits this URL directly without going through the studio, they get bounced back to the wizard start.

### Component tree on this screen

```
App
└── <router-outlet>                          (no Sidebar)
    └── ExportBundleComponent
        ├── StepperComponent                  (wizard step 3, active)
        ├── (summary cards — inline)
        └── (Download button — inline)
```

---

## The Redirect-Only Screens

There are a few URLs that don't render their own components — they just redirect.

### `/` → `/dashboard`

The root URL. Defined with `pathMatch: 'full'` so it doesn't shadow every other route. Used by:

- Direct bookmarks.
- The "Cancel" button on the wizard (`/templates/new/schema-setup` → router navigates to `/`).

### `/home` → `/dashboard`

A legacy alias from an earlier version of the app. Kept for backwards compatibility — old bookmarks and email links still work.

### `/projects/new/schema-setup` → `/templates/new/schema-setup`

Same story — the app was renamed from "projects" to "templates" at some point. Old links redirect.

### `/projects/:id/studio` → `/templates/:id/studio`

The `:id` parameter is automatically forwarded by Angular's router. So `/projects/abc/studio` becomes `/templates/abc/studio`. (⚠ query params like `?edit=true` are lost in the redirect.)

### `**` → `/dashboard`

The wildcard — any URL that doesn't match any of the above. Silently redirects to dashboard. (Could be replaced with a proper 404 page; see Refactoring Playbook.)

---

## Quick Reference — Which Screen Uses Which Components

| Screen | Sidebar? | Hosts | Embeds | Opens (dialogs) | Calls (services) |
|---|---|---|---|---|---|
| `/dashboard` | ✓ | `DashboardComponent` | mat-toolbar, mat-card, mat-progress-bar | `DeleteTemplateDialogComponent` | `DashboardService` |
| `/system` | ✓ | `System` | `SystemCardComponent` × 6 | `RegisterSystemDialogComponent`, `DeleteSystemDialogComponent` | `SystemService` |
| `/crosswalk` | ✓ | `CrosswalkTablesComponent` | `CrosswalkCardComponent` × N | `CrosswalkCreateDialogComponent`, `DeleteCrosswalkDialogComponent` | `CrosswalkService`, `SystemService` |
| `/template-builder` | ✓ | `TemplateBuilderComponent` | (none — all inline) | (none) | `CrosswalkService`, `RuleSetService` (stub) |
| `/templates/new/schema-setup` | ✗ | `NewTemplateComponent` | `StepperComponent`, `TemplateMetadataFormComponent` | `SchemaUploadDialogComponent` | `StudioStateService`, `NewTemplateStateService`, `SystemService`, `FhirSchemaLoaderService` |
| `/templates/:id/studio` | ✗ | `MappingStudioComponent` | `StepperComponent`, `SchemaTreeComponent` × 2, `BridgeCardComponent` × N, `BridgeEditorComponent`, `ValidationPreviewComponent` | `FieldConnectionDialogComponent`, `ConfirmDialogComponent` | `StudioStateService`, `NewTemplateStateService`, `CrosswalkService` |
| `(modal:render-preview)` | n/a | `FhirRenderPreviewComponent` | (none) | (none) | `StudioStateService` |
| `/export-bundle` | ✗ | `ExportBundleComponent` | `StepperComponent` | (none) | `StudioStateService`, `ExportBundleService`, `SystemService`, `CrosswalkService` |

---

## The Three Components That Show Up On Multiple Screens

A small group of components are reused across screens. Knowing where they appear helps when planning changes.

### `Sidebar`
Appears on: `/dashboard`, `/system`, `/crosswalk`, `/template-builder`.
Owned by `App`. Renders a static nav rail with 4 links and `routerLinkActive` for highlighting.

### `StepperComponent`
Appears on: `/templates/new/schema-setup`, `/templates/:id/studio`, `/export-bundle`.
The 3-step breadcrumb. Each screen passes a different `steps` array (with the current step marked `active`) and listens for the `stepClick` event.

### `ConfirmDialogComponent`
Opened by: anywhere that needs a generic Yes/No confirmation. Most heavily used by `MappingStudioComponent` for "Delete bridge?", "Reset all bridges?", "Leave with unsaved changes?".

---

*End of screen-by-screen guide.*