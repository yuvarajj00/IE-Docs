# Mapping Studio UI — Component & Utility Deep Dive

> Companion document to `Angular_Project_Analysis.md`.
>
> Where the analysis answers **"what's in the codebase and is it healthy?"**, this document answers **"how does each piece actually work?"** — with code, scenarios, and concrete input/output examples.
>
> Every component and utility in `src/app/` is covered. For each one you get:
>
> - **Purpose** — one-line summary
> - **Inputs / Outputs / Key state** — the contract
> - **Logic** — how it does the work
> - **Code** — the critical method or two
> - **Scenario** — a realistic user story
> - **Example** — concrete data flowing through it

---

## Table of Contents

**Part 1 — Components**

1. [Root & Layout](#1-root--layout) — `App`, `Sidebar`, `Header`
2. [Shared Widgets](#2-shared-widgets) — `Stepper`, `StatusBadge`, `TagBadge`, `ProgressBar`, `ConfirmDialog`
3. [Dashboard](#3-dashboard) — `DashboardComponent`, `DeleteTemplateDialog`
4. [Template Wizard (Step 1)](#4-template-wizard-step-1) — `NewTemplate`, `TemplateMetadataForm`, `SchemaUploadDialog`
5. [Mapping Studio (Step 2)](#5-mapping-studio-step-2) — `MappingStudio`, `SchemaTree`, `BridgeCard`, `BridgeEditor`, `FieldConnectionDialog`, `ValidationPreview`, `FhirRenderPreview`
6. [Export Bundle (Step 3)](#6-export-bundle-step-3)
7. [System Management](#7-system-management) — `System`, `SystemCard`, `RegisterSystemDialog`, `DeleteSystemDialog`
8. [Crosswalk Management](#8-crosswalk-management) — `CrosswalkTables`, `CrosswalkCard`, `CrosswalkCreateDialog`, `DeleteCrosswalkDialog`
9. [Template Builder](#9-template-builder) — `TemplateBuilderComponent`

**Part 2 — Utilities**

10. [`schema-parse.util.ts`](#10-schema-parseutilts)
11. [`schema-to-nodes.util.ts`](#11-schema-to-nodesutilts)
12. [`path-context.util.ts`](#12-path-contextutilts)
13. [`validation.util.ts`](#13-validationutilts)
14. [`date-format.util.ts`](#14-date-formatutilts)
15. [`template-to-scriban.util.ts`](#15-template-to-scribanutilts)

---

# Part 1 — Components

---

## 1. Root & Layout

### 1.1 `App` (root component)

- **File:** `src/app/app.ts` / `app.html` / `app.css`
- **Selector:** `app-root`
- **Purpose:** The application shell. Hosts the primary `<router-outlet>` and a `<router-outlet name="modal">` for modal-style routes. Decides whether the sidebar is shown.

**State**

```ts
private currentUrl = signal('/');
showSidebar = computed(() =>
  GLOBAL_LAYOUT_ROUTES.some(r => this.currentUrl().startsWith(r))
);
```

`GLOBAL_LAYOUT_ROUTES = ['/dashboard', '/system', '/crosswalk', '/template-builder']`. The wizard, mapping studio and export-bundle screens have their own embedded toolbars, so they **suppress** the global sidebar.

**Logic**

```ts
constructor(private router: Router) {
  this.router.events
    .pipe(filter(e => e instanceof NavigationEnd))
    .subscribe((e) => {
      this.currentUrl.set((e as NavigationEnd).urlAfterRedirects);
    });
}
```

On every successful navigation, the `currentUrl` signal updates and the `showSidebar` `computed` is re-evaluated. Angular's change detection re-renders the shell automatically.

**Template (`app.html`)**

```html
<div class="app-container">
  @if (showSidebar()) { <app-sidebar /> }
  <main class="content"><router-outlet></router-outlet></main>
</div>
<router-outlet name="modal"></router-outlet>
```

> **Scenario:** A user clicks "New Template" on the dashboard. The router navigates to `/templates/new/schema-setup`. `NavigationEnd` fires → `currentUrl.set('/templates/new/schema-setup')` → `showSidebar` recomputes to `false` (no `GLOBAL_LAYOUT_ROUTES` prefix matches) → the sidebar disappears, giving the wizard a full-width canvas.

> **Leak note:** The `subscribe(...)` has no `takeUntilDestroyed`. `App` lives for the whole app lifetime so it's harmless in practice, but `takeUntilDestroyed(inject(DestroyRef))` is the modern idiom.

---

### 1.2 `Sidebar`

- **File:** `src/app/core/layout/sidebar/sidebar.ts`
- **Selector:** `app-sidebar`
- **Purpose:** Static left-rail navigation. Pure presentational component.

**Code**

```ts
@Component({
  selector: 'app-sidebar',
  imports: [RouterLink, RouterLinkActive],
  templateUrl: './sidebar.html',
  styleUrl: './sidebar.css',
})
export class Sidebar {}
```

The entire logic lives in the template — `[routerLink]` and `routerLinkActive="active"` directives wire up the menu items. Zero TypeScript.

> **Scenario:** The dashboard is open. The Sidebar renders a "Dashboard" item with `routerLinkActive` matching `/dashboard`, so it gets the `.active` class for visual emphasis.

---

### 1.3 `Header` — ⚠ Dead Code

- **File:** `src/app/core/layout/header/header.ts`
- **Selector:** `app-header`
- **Status:** **Never used.** `grep` for `<app-header` returns only the definition.

```ts
export class Header {
  constructor(private router: Router) {}
  goToNewTemplate(): void {
    this.router.navigate(['/templates/new/schema-setup']);
  }
}
```

The component compiles, the selector is declared, but no template references it. Safe to delete the entire `core/layout/header/` folder.

---

## 2. Shared Widgets

### 2.1 `StepperComponent`

- **File:** `src/app/shared/stepper/stepper.ts`
- **Selector:** `app-stepper`
- **Purpose:** A 3-step "Schema Setup → Mapping Studio → Export Bundle" breadcrumb.

**API**

```ts
export interface StepperStep {
  label: string;
  state: 'active' | 'completed' | 'inactive';
  route?: string;
}

@Input()  steps: StepperStep[] = [];
@Output() stepClick = new EventEmitter<{ step: StepperStep, index: number }>();
```

**Logic — only the ACTIVE step emits**

```ts
onStepClick(step: StepperStep, index: number): void {
  if (step.state === 'active') {
    this.stepClick.emit({ step, index });
  }
}
```

This is deliberately restrictive: clicking an "inactive" or "completed" step does nothing. The Mapping Studio uses this so a user can't accidentally jump *backwards* and lose unsaved work; `MappingStudioComponent.onStepClick` further blocks navigation to "SCHEMA SETUP" entirely (the only writable route is forward to `EXPORT BUNDLE`).

> **Scenario:** User is in Mapping Studio. The stepper shows `[✓ SCHEMA SETUP] [● MAPPING STUDIO] [○ EXPORT BUNDLE]`. They click "EXPORT BUNDLE" — but state is `inactive`, so nothing happens. Then they finish bridging, click the "Continue to Export" button which itself calls `router.navigate(['/export-bundle'])`.

> **Example input:**
> ```ts
> [
>   { label: 'SCHEMA SETUP', state: 'completed', route: 'templates/new/schema-setup' },
>   { label: 'MAPPING STUDIO', state: 'active' },
>   { label: 'EXPORT BUNDLE', state: 'inactive' },
> ]
> ```
> Output: only the middle step is clickable.

---

### 2.2 `StatusBadge` — ⚠ Dead Code

```ts
@Input() status: 'COMPLETED' | 'INPROGRESS' = 'INPROGRESS';
```

Renders a colored pill. **Not referenced in any template**, even though the Dashboard absolutely needs status pills — it uses Material `mat-chip` instead. Safe to delete.

---

### 2.3 `TagBadge` — ⚠ Dead Code

```ts
@Input() label: string = '';
```

Same story as `StatusBadge`. Unused.

---

### 2.4 `ProgressBar` — ⚠ Dead Code

```ts
@Input() value: number = 0;
```

Dashboard uses Material's `mat-progress-bar`. Unused.

---

### 2.5 `ConfirmDialogComponent`

- **File:** `src/app/shared/confirm-dialog/confirm-dialog.component.ts`
- **Purpose:** A generic Yes/No confirmation dialog opened via `MatDialog`.

**Data contract**

```ts
export interface ConfirmDialogData {
  title: string;
  message: string;
  confirmText?: string;    // default "Confirm"
  cancelText?: string;     // default "Cancel"
  icon?: string;           // Material icon name, default "warning"
  warn?: boolean;          // applies .warn red styling, default true
}
```

**Result**: a boolean — `true` on confirm, `false` on cancel/close.

**Code — the two action handlers**

```ts
cancel():  void { this.dialogRef.close(false); }
confirm(): void { this.dialogRef.close(true); }
```

> **Scenario — used by MappingStudio.goBack():** The user has 7 unsaved bridges. They click the back arrow. The studio opens this dialog:
> ```ts
> this.dialog.open(ConfirmDialogComponent, {
>   data: {
>     title: 'Leave Mapping Studio?',
>     message: 'You have unsaved bridge mappings…',
>     confirmText: 'Yes, Leave',
>     cancelText: 'Stay Here',
>     icon: 'warning',
>     warn: true
>   },
>   width: '460px'
> }).afterClosed().subscribe(confirmed => {
>   if (confirmed) this.router.navigate(['/templates/new/schema-setup']);
> });
> ```
> User clicks "Stay Here" → `confirmed = false` → no navigation.

---

## 3. Dashboard

### 3.1 `DashboardComponent`

- **File:** `src/app/features/dashboard/dashboard.component.ts`
- **Route:** `/dashboard` (also the `/` redirect target)
- **Purpose:** Landing page. Lists templates with search, shows 4 stat tiles, allows open/edit/duplicate/delete/export per template.

**Key signals**

```ts
search             = signal('');
isLoading          = signal(true);
showAllMappings    = signal(false);
private allProjects = signal<MappingProject[]>([]);
stats              = signal<DashboardStats>({ templates: 0, systems: 0, crosswalks: 0, ruleSets: 0 });

mappings = computed(() => {           // ← filtered + paged
  const q = this.search().trim().toLowerCase();
  let projects = this.allProjects();
  if (q) projects = projects.filter(m =>
    m.title.toLowerCase().includes(q) ||
    m.sourceSystem.toLowerCase().includes(q) ||
    m.targetSystem.toLowerCase().includes(q) ||
    m.tags.some(t => t.toLowerCase().includes(q)) ||
    m.status.toLowerCase().includes(q)
  );
  return this.showAllMappings() ? projects : projects.slice(0, 10);
});
```

The component-local signals **drive the template directly**. Type `"acme"` in the search box → `search.set('acme')` → `mappings` recomputes → `*ngFor` re-renders.

**Logic flow — `ngOnInit → loadDashboardData()`**

```ts
this.dashboardService.getDashboardData().subscribe({
  next: data => {
    this.stats.set(data.stats);
    this.allProjects.set(data.recentProjects);   // ← triggers `mappings`
    this.isLoading.set(false);
  },
  error: err => { console.error(err); this.isLoading.set(false); }
});
```

`getDashboardData` itself does `forkJoin({ systems, crosswalks, templates })` to derive stats from the actual arrays, then maps each template through `_createMappingProject` (with a coverage fallback computation).

**Per-row actions**

| Method | What it does |
|---|---|
| `openTemplate(t)` | `navigate(['/templates', t.id, 'studio'], { queryParams: { edit: 'true' } })` |
| `deleteTemplate(t, $event)` | Opens `DeleteTemplateDialog`, then `dashboardService.deleteProject(id)`; removes from `allProjects` signal and decrements `stats.templates` |
| `duplicateTemplate(t)` | POST `/template/{id}/duplicate`, prepend new row to list |
| `exportTemplate(t)` | GET `/template/{id}/export` as `Blob`, then `URL.createObjectURL` + `<a>.click()` to download |

> **Scenario — Search & open**
>
> 1. User opens `/`. Redirect → `/dashboard`. `isLoading=true` shows a `mat-progress-bar`.
> 2. `getDashboardData()` returns: `{ stats: {templates: 12, systems: 4, …}, recentProjects: [12 templates] }`.
> 3. UI shows two stat tiles ("Templates: 12", "Systems: 04") and the first 10 templates.
> 4. User types `"FHIR"` → `search.set('FHIR')` → `mappings()` filters projects to those whose title/tags/system match → table shows 3 results.
> 5. User clicks the first row → `openTemplate(t)` → URL becomes `/templates/abc123/studio?edit=true`.

> **Performance note:** `mappings` and `displayedMappingsCount` duplicate the same filter logic. Extract a `_filteredProjects = computed(...)` and base the other two on it.

### 3.2 `DeleteTemplateDialogComponent`

- **File:** `src/app/features/dashboard/delete-template-dialog/delete-template-dialog.component.ts`
- **Purpose:** Confirmation dialog before deleting a template (note: separate from the generic `ConfirmDialog` — this one has its own styling for the dashboard context).
- **Data contract:** `{ templateName: string }`
- **Result:** `boolean`

> **Example:** Dashboard calls it with `{ templateName: 'Patient → FHIR R4' }` → dialog text shows "Are you sure you want to delete 'Patient → FHIR R4'?" → user confirms → `dialogRef.close(true)`.

---

## 4. Template Wizard (Step 1)

### 4.1 `NewTemplateComponent`

- **File:** `src/app/features/template-wizard/new-template.ts`
- **Route:** `/templates/new/schema-setup`
- **Purpose:** Step 1 of the wizard. Wraps the stepper + `TemplateMetadataFormComponent`. Generates a UUID for new templates and navigates to the studio on Continue.

**Key methods**

```ts
ngOnInit(): void {
  this.route.queryParams.subscribe(params => {     // ⚠ no takeUntilDestroyed
    const templateId = params['templateId'];
    if (templateId) this.loadExistingTemplate(templateId);
  });
}

continueToMapping(): void {
  if (!this.isValid()) return;
  const templateId = this.currentTemplateId || crypto.randomUUID();
  this.router.navigate(['templates', templateId, 'studio']);
}
```

**Edit-mode hydration**

```ts
private loadExistingTemplate(templateId: string): void {
  this.stateService.setEditMode(true);
  this.studioService.loadFromBackend(templateId).subscribe({
    next: loaded => {
      if (loaded) {
        const title = this.studioService.title();
        const sourceSystem = this.studioService.sourceSystem();
        const sourceRawSchema = this.studioService.sourceRawSchema();
        // … push it all into NewTemplateStateService
        this.stateService.setTitle(title);
        this.stateService.setSourceSystem(sourceSystem);
        if (sourceRawSchema) this.stateService.setSchema({ type: 'source', ... });
      }
    }
  });
}
```

It **uses StudioStateService as a backend fetcher**, reads its signals back out, and pushes them into `NewTemplateStateService`. This is a small "model migration" between the two stores.

> **Scenario — New template flow**
>
> 1. User clicks "New Template" on dashboard.
> 2. Lands on `/templates/new/schema-setup`. `currentTemplateId = ''`, `isLoading = false`.
> 3. User types title "ABC → FHIR", picks Source/Target systems, uploads two schema files. `NewTemplateStateService.isValid()` becomes `true`.
> 4. "Continue" button enables. Click it → `crypto.randomUUID()` returns `'2f1c…'` → navigate to `/templates/2f1c…/studio`.

### 4.2 `TemplateMetadataFormComponent`

- **File:** `src/app/features/template-wizard/template-metadata-form/template-metadata-form.ts`
- **Purpose:** The form half of the wizard. Reactive form with title, description, source & target system dropdowns, and two "schema upload" buttons that each open `SchemaUploadDialogComponent`.

**Form**

```ts
form = this.fb.group({
  title: ['', [Validators.required, Validators.minLength(3)]],
  description: [''],
  sourceSystem: ['', Validators.required],
  targetSystem: ['', Validators.required],
});
```

It listens to `form.valueChanges` and pushes each change into `NewTemplateStateService.setTitle` / `setDescription` / `setSourceSystem` / `setTargetSystem`. Schema selections are also pushed via `setSchema()`.

> **Scenario — Upload source schema**
>
> User clicks "Upload Source Schema" → `MatDialog.open(SchemaUploadDialogComponent, { data: { type: 'source' } })` → dialog returns an `UploadedSchema` → component calls `stateService.setSchema(result)` → `NewTemplateStateService._hasValidJson()` sees both source & target are present → `isValid` becomes `true` → parent's Continue button enables.

### 4.3 `SchemaUploadDialogComponent`

- **File:** `template-metadata-form/schema-upload-dialog/schema-upload-dialog.ts`
- **Purpose:** Two upload modes: (1) browse/select a file (.json or .xml), or (2) pick a FHIR resource type from the bundled `fhir.schema.json` (3.3 MB).

**Modes:**

- **File mode**: drag-and-drop or browse → `parseSchemaFile(file, type)` (utility) → if `ok`, close dialog with the `UploadedSchema`.
- **FHIR mode**: search "Patient" → `fhirLoader.loadFhirResource('Patient')` → returns `SchemaNode[]` + raw FHIR snapshot → dialog closes with `{ type, fileName: 'Patient.json', rawText: <fhirJson>, summary: { detectedFormat: 'json', rootName: 'Patient' } }`.

> **Example:** Source mode, user uploads `customer.json` (8 KB). `parseSchemaFile` returns:
> ```ts
> {
>   ok: true,
>   schema: {
>     type: 'source',
>     fileName: 'customer.json',
>     fileSize: 8192,
>     mimeType: 'application/json',
>     rawText: '{"customer":{"id":1,...}}',
>     summary: { detectedFormat: 'json', topLevelCount: 1, isValid: true }
>   }
> }
> ```
> Dialog closes with this object. The metadata form then calls `setSchema(...)`.

---

## 5. Mapping Studio (Step 2)

This is the heart of the app — **6 components, ~3,300 LOC, 4 init branches, drag-drop, dialogs, named outlet, and live validation**.

### 5.1 `MappingStudioComponent`

- **File:** `src/app/features/mapping-studio/mapping-studio.ts`
- **Route:** `/templates/:id/studio` (with optional `?edit=true`)
- **Purpose:** Orchestrates the source tree, target tree, bridge list, field-connection dialog and validation preview.

**The four initialization branches**

```ts
ngOnInit(): void {
  this.loadCrosswalks();
  const projectId = this.route.snapshot.paramMap.get('id')
                 ?? this.route.snapshot.queryParamMap.get('id') ?? '';

  if (this._handleEditMode(projectId))               return;  // RULE 2: ?edit=true → load from backend
  const state = this.newProjectState.state();
  if (this._handleNewTemplateFromWizard(projectId, state)) return;  // RULE 1: wizard handed us data
  if (this._handleDirectInitialization(projectId))   return;  // RULE 3: bare ID, empty start
  this._redirectToSchemaSetup();                              // RULE 4: nothing → go back
}
```

Each branch is a separate private method, returning `true` if it handled the case. The order matters: **edit** is checked first because the URL might have `?edit=true&id=…` *plus* in-memory wizard state from a previous session — edit always wins.

**Bridge creation flow**

```ts
onBridgeCreated(event: { sourcePath: string; targetPath: string }): void {
  this.connectionSourcePath = event.sourcePath;
  this.connectionTargetPath = event.targetPath;
  this.connectionTransform = 'none';
  this.connectionDialogOpen = true;
}
```

When `SchemaTreeComponent` emits `fieldDropped`, this method captures the paths and **opens `FieldConnectionDialogComponent`** (it's rendered conditionally in the template via `*ngIf="connectionDialogOpen"`). The dialog later emits `commit` → `onConnectionCommit(event)` actually mutates the studio store.

**The `onConnectionCommit` algorithm** is the most subtle:

```ts
onConnectionCommit(event: {...}): void {
  // Preserve existing params, overlay only fields the dialog emitted.
  // Prevents wiping params like recordType, sourceArrayPath, collectionTypeField
  // that are derived elsewhere and not managed by the dialog.
  const existingParams = this.editingBridgeId
    ? { ...(this.studio.bridges().find(b => b.id === this.editingBridgeId)?.transform?.params ?? {}) }
    : {};
  const newParams: Record<string, string> = { ...existingParams };
  if (event.ruleSetId !== undefined)     newParams['ruleSetId'] = event.ruleSetId || '';
  if (event.crosswalkName !== undefined) newParams['crosswalkName'] = event.crosswalkName || '';
  // … 4 more
  const transformConfig = event.transform !== 'none'
    ? { kind: event.transform, ruleSetIds: …, params: newParams }
    : { kind: 'none' };

  if (this.editingBridgeId) this.studio.updateBridge(this.editingBridgeId, {…});
  else                       this.studio.addBridge(event.sourcePath, event.targetPath);
}
```

The "overlay" pattern (only set keys the dialog explicitly emitted) is important because *other parts of the system* (the bridge normalization service) derive params like `recordType`, `sourceArrayPath`, `collectionTypeField`. If the dialog blindly replaced `params`, those would be lost on every edit.

**Canvas drop zone**

```ts
onCanvasDragOver(event: DragEvent) { event.preventDefault(); this.canvasDragOver = true; }
onCanvasDrop(event: DragEvent) {
  event.preventDefault();
  this.canvasDragOver = false;
  const sourcePath = event.dataTransfer?.getData('text/plain');
  if (!sourcePath) return;
  this.connectionSourcePath = sourcePath;
  this.connectionTargetPath = '';
  this.connectionTransform = 'none';
  this.connectionDialogOpen = true;     // open dialog with target unset
}
```

So there are **two ways to create a bridge**:
1. Drop a source field directly **onto a target field** in the target tree → `SchemaTree` emits both paths → dialog opens with both set.
2. Drop a source field onto the empty **canvas** → dialog opens with only source set; user picks target inside the dialog.

> **Scenario — Edit existing bridge**
>
> 1. User on `/templates/abc/studio` sees 12 bridges in the list.
> 2. Clicks the pencil icon on the "PatientName.Family → name.family" bridge.
> 3. `BridgeCard` emits `edit('bridge_5')` → `onEditBridge('bridge_5')` fires.
> 4. Method finds the bridge in `studio.bridges()`, sets `editingBridgeId = 'bridge_5'`, hydrates `connectionSourcePath`, `connectionTargetPath`, `connectionTransform`, and opens the dialog.
> 5. User changes transform from "none" → "dateFormat" with param `"yyyy-MM-dd"`. Hits Save.
> 6. Dialog emits `commit({ sourcePath, targetPath, transform: 'dateFormat', params: {...} })`.
> 7. `onConnectionCommit` overlays `params['ruleSetId']`, `params['crosswalkName']`, etc. — but **preserves** `params['recordType'] = 'PATIENT'` that the normalization service set earlier.
> 8. `studio.updateBridge('bridge_5', { ... })` runs. The `bridges` signal updates. The `BridgeCard` for bridge_5 re-renders.

### 5.2 `SchemaTreeComponent`

- **File:** `mapping-studio/schema-tree/schema-tree.ts`
- **Purpose:** A custom, virtualization-free tree component with HTML5 drag-drop, expand/collapse, search filtering, type-color icons and "mapped" highlighting.

**Inputs / Outputs**

```ts
@Input()  nodes: SchemaNode[] = [];
@Input()  mode: 'source' | 'target' = 'source';
@Input()  searchQuery = '';
@Input()  mappedPaths: string[] = [];
@Output() fieldDropped       = new EventEmitter<{sourcePath: string; targetPath: string}>();
@Output() configureConnection = new EventEmitter<FlatTreeNode>();
```

The same component renders both sides — the `mode` input governs drag *direction*:
- `source` mode: draggable, NOT droppable.
- `target` mode: droppable, NOT draggable.

**Flattening algorithm — the core**

```ts
private _flatten(nodes: SchemaNode[], depth: number): FlatTreeNode[] {
  const q = this.searchQuery.trim().toLowerCase();
  const result: FlatTreeNode[] = [];
  for (const node of nodes) {
    const selfMatch  = !q || node.label.toLowerCase().includes(q);
    const childMatch = !q || this._anyDescendantMatches(node, q);
    if (!selfMatch && !childMatch) continue;     // skip whole subtree
    const hasChildren = !!(node.children?.length);
    const shouldExpand = hasChildren
      ? (this._expandedIds.has(node.id) || (!!q && childMatch))
      : false;
    result.push({ ...node, depth, isExpanded: shouldExpand });
    if (hasChildren && shouldExpand) {
      result.push(...this._flatten(node.children!, depth + 1));
    }
  }
  return result;
}
```

Key insight: **a search query forces nodes to auto-expand** if any descendant matches (so the matching child becomes visible). Without a query, only manually-expanded nodes show their children.

**Drag-drop**

```ts
onDragStart(event: DragEvent, node: FlatTreeNode): void {
  if (!this.isDraggable(node)) return;
  event.dataTransfer?.setData('text/plain', node.path);
  if (event.dataTransfer) event.dataTransfer.effectAllowed = 'link';
  this.draggingPath = node.path;
}

onDrop(event: DragEvent, node: FlatTreeNode): void {
  event.preventDefault();
  this.dragOverPath = null;
  if (!this.isDraggable(node)) return;     // target side filter
  const sourcePath = event.dataTransfer?.getData('text/plain');
  if (sourcePath) this.fieldDropped.emit({ sourcePath, targetPath: node.path });
}
```

It uses **native HTML5 drag-drop** (no Angular CDK). Source path is stuffed into `dataTransfer.text/plain`, and the drop handler pulls it back out. This sidesteps the CDK's parent-child constraints (the source tree and target tree are independent components) but loses the CDK's preview/placeholder polish.

**Mapped highlighting**

```ts
isMapped(path: string): boolean {
  if (this.mappedPaths.indexOf(path) !== -1) return true;
  // Bridges store generic [] notation (e.g. "Records[].PatientId")
  // but source schema nodes may retain real indices (e.g. "Records[0].PatientId").
  const normalizedPath = path.replace(/\[\d+\]/g, '[]');
  return normalizedPath !== path && this.mappedPaths.indexOf(normalizedPath) !== -1;
}
```

Bridges always normalize to `[]`, but tree nodes can carry concrete indices. This double-check covers both.

> **Scenario — Search & drag**
>
> 1. Source tree shows `Patient → identifier → Records[0..N]`.
> 2. User types `"birth"` in source search. `_flatten` finds `Patient.birthDate` deep in the tree and auto-expands the path.
> 3. User drags `Patient.birthDate` over the target tree's `Patient.birthDate` node.
> 4. `event.dataTransfer.setData('text/plain', 'Patient.birthDate')` runs on the source side.
> 5. Target side's `onDragOver` calls `preventDefault()` and sets `dragOverPath = 'Patient.birthDate'` → CSS highlights the row.
> 6. User releases → `onDrop` reads `'Patient.birthDate'` back from `dataTransfer` and emits `fieldDropped({ sourcePath: 'Patient.birthDate', targetPath: 'Patient.birthDate' })`.
> 7. Parent `MappingStudio` opens the `FieldConnectionDialog`.

### 5.3 `BridgeCardComponent`

- **File:** `mapping-studio/bridge-card/bridge-card.ts`
- **Purpose:** A rich card per bridge: shows source path with type icons, target path with FHIR type info, logic selector (none / transform / crosswalk), operator summary, condition groups, status pill.

**The rehydration problem**

A bridge stored on the backend has only a string blob in `transform.params['valueMapRules']`. To render the card *exactly* as the user left it last session, the card must **parse that blob back into UI state**. Two legacy formats coexist:

1. **New format** (post-refactor) — a single `OperatorSummary` object: `{ operator, params, scribanExpr, preProcessSteps, activePattern }`.
2. **Legacy format** — an array of `{operator: 'equals'|'notEquals', sourceValue, destValue}` rules.

```ts
private _rehydrateTransformRules(): void {
  this.operatorSummary = null;
  if (this.selectedLogic !== 'transform') return;
  const raw = this.bridge?.transform?.params?.['valueMapRules'];
  if (!raw) { this._ensureDefaultTransformState(); return; }
  try {
    const parsed = JSON.parse(raw);
    if (this._tryLoadOperatorSummary(parsed)) return;   // new format
    if (this._tryLoadLegacyRules(parsed)) return;       // legacy format
  } catch { /* ignore */ }
  this._ensureDefaultTransformState();
}
```

The card **gracefully degrades** through three layers (new → legacy → default empty group).

**Logic switch**

```ts
private _determineSelectedLogic(kind: string, crosswalkName: string): void {
  if (kind === 'codeMap' || crosswalkName) {
    this.selectedLogic = 'crosswalk';
    this.selectedCrosswalkName = crosswalkName || null;
    this.conditionGroups = [];
  } else if (kind === 'customScript') {
    this.selectedLogic = 'transform';
  } else {
    this.selectedLogic = 'none';
  }
}
```

> **Scenario — Status display**
>
> Bridge has `status: 'warning', message: 'Type mismatch: string → date. A transform is recommended.'`. The card renders a yellow ⚠ chip with the message in a tooltip. The user clicks the pencil → `edit.emit(bridge.id)` → parent opens `FieldConnectionDialog` and lets them pick the `dateFormat` transform.

### 5.4 `BridgeEditorComponent` — ⚠ Partially Superseded

- **File:** `mapping-studio/bridge-editor/bridge-editor.ts`
- **Purpose:** A *simpler* inline editor with 5 hard-coded transform options. Largely replaced by `FieldConnectionDialog`.

**Code (74 LOC total)**

```ts
@Input()  bridge!: Bridge;
@Output() save  = new EventEmitter<Partial<Bridge>>();
@Output() close = new EventEmitter<void>();

readonly transforms = [
  { value: 'none',         label: 'None' },
  { value: 'concat',       label: 'Concat (append value)' },
  { value: 'static',       label: 'Static Value' },
  { value: 'dateFormat',   label: 'Date Format (ISO → YYYY-MM-DD)' },
  { value: 'codeMap',      label: 'Code Map' },
  { value: 'customScript', label: 'Custom Script' },
];

runTest(): void {
  switch (this.transformKind) {
    case 'static':     this.testOutput = this.paramValue; break;
    case 'concat':     this.testOutput = this.testInput + ` ${this.paramValue}`; break;
    case 'dateFormat': this.testOutput = new Date(this.testInput).toISOString().split('T')[0]; break;
    default:           this.testOutput = this.testInput;
  }
}
```

It also has a "Test" button that runs a synthetic transformation locally so the user can preview output. The big `FieldConnectionDialog` doesn't have this test-in-place mode — so this component is **partially valuable**, even if 90% of users never reach it.

> **Recommendation:** Either merge `runTest()` into the dialog or delete this component entirely.

### 5.5 `FieldConnectionDialogComponent`

- **File:** `mapping-studio/field-connection-dialog/field-connection-dialog.ts`
- **Size:** **1,864 LOC** — the largest single source file in the project.
- **Purpose:** The full bridge configuration surface. Supports:
  - **4 transformation patterns**: one-to-one, many-to-one, one-to-many, many-to-many.
  - **3 logic modes**: direct, transform (with operators), conditional.
  - **15 value-map operators**: equals, notEquals, isNull, isNotNull, contains, notContains, startsWith, endsWith, greaterThan, lessThan, greaterOrEqual, lessOrEqual, in, notIn, matches.
  - **RuleSet selection** (legacy concept, still wired through deprecated `RuleSetService`).
  - **Crosswalk selection**.
  - **Pre-process pipeline** of operator steps (e.g. `string.trim | string.downcase`).
  - **Scriban operator catalog** — categories and ~80 operators imported from `scriban-operators.constants.ts`.

**The data model**

```ts
export interface ValueMapRule {
  id: number;
  conditions: ValueMapCondition[];   // multiple conditions per rule
  conditionLogic: 'and' | 'or';
  targetValue: string;
  targetValueType: 'literal' | 'field';
  targetDescription?: string;
  targetSystem?: string;
}
export interface ValueMapCondition {
  sourceField: string;
  operator: ValueMapOperator;
  compareValue: string;
  compareValueType: 'literal' | 'field';
}
```

A *bridge* can therefore carry a complex decision tree: "IF `record.gender == 'M' AND record.age > 18` THEN output `'adult-male'`; ELSE IF …".

**Hydration**

```ts
private _hydrateFromBridge(bridge: Bridge): void {
  // Source paths — split comma-joined multi-source back into array
  const sources = bridge.sourcePath.split(',').map(s => s.trim()).filter(Boolean);
  if (sources.length > 0) {
    this.selectedSourcePaths = sources;
    this.selectedSourcePath  = sources[0];
  }
  // … similar for targets
  // … then parse valueMapRules JSON, transform kind, ruleSetIds, crosswalkName, etc.
}
```

The dialog is the **inverse function** of `MappingStudio.onConnectionCommit` — it takes a stored `Bridge`, reconstructs every UI control's state, then emits the modified bridge on Save.

> **Scenario — Many-to-one with conditions**
>
> 1. Bridge target: `Patient.gender` (FHIR `code`).
> 2. Source: drag in `Records[].gender` AND `Records[].sex`.
> 3. Dialog detects 2 source paths → pattern = `'many-to-one'`.
> 4. User picks **transform** logic, operator `equals`.
> 5. Adds 3 rules:
>    - `gender == 'M' → 'male'`
>    - `gender == 'F' → 'female'`
>    - `sex == 'X' → 'other'`
> 6. Clicks Save. Dialog emits:
>    ```ts
>    commit({
>      sourcePath: 'Records[].gender,Records[].sex',
>      targetPath: 'Patient.gender',
>      transform: 'customScript',
>      valueMapRules: JSON.stringify({
>        operator: 'equals',
>        activePattern: 'many-to-one',
>        params: {...},
>        scribanExpr: '{{ ... if … else … end }}',
>      })
>    })
>    ```
> 7. `MappingStudio.onConnectionCommit` stores it. `BridgeCard` rehydrates the operator summary on next render.

### 5.6 `ValidationPreviewComponent`

- **File:** `mapping-studio/validation-preview/validation-preview.ts` (819 LOC)
- **Purpose:** Right-hand panel of the studio. Shows:
  - Bridge counts (valid / warning / invalid).
  - The auto-generated **Scriban template** with syntax highlighting and edit mode.
  - The **Template JSON** (the canonical representation that goes to the backend).
  - A **"Viva" view** — runs Scriban locally against user-pasted JSON.
  - A **versions panel** to load any previously-saved Scriban revision.

**Template JSON builder — the canonical export shape**

```ts
get templateJson(): object {
  const bridges = this.activeBridges.map(b => this._templateJson_buildBridgeEntry(b));
  const normalizeNode = (n: any) => this._templateJson_normalizeNode(n);
  return {
    title: this.title || 'Mapping Template',
    ...(this.templateId ? { templateId: this.templateId } : {}),
    sourceSystem: this.sourceFormat || 'Source',
    targetSystem: this.targetFormat || 'Target',
    tags: this._deriveTags(),
    coverage: this.validPercent,
    status: this._projectStatus(),
    bridges,
    sourceSchema: { nodes: this.sourceNodes.map(normalizeNode) },
    targetSchema: { nodes: this.targetNodes.map(normalizeNode) }
  };
}
```

`_templateJson_normalizePath` strips all `[N]` markers so the persisted shape uses generic `[]` notation. This is the **canonicalized** form everything downstream expects.

**Coverage & status**

```ts
get validPercent(): number {
  const total = this.activeBridges.length;
  return total === 0 ? 0 : Math.round((this.validCount / total) * 100);
}

private _projectStatus(): string {
  if (this.activeBridges.length === 0) return 'PENDING';
  if (this.validPercent === 100) return 'COMPLETED';
  return 'INPROGRESS';
}
```

Note: this is *workspace-side* coverage (% valid bridges). The dashboard service has its own *target-field* coverage (% target fields mapped). They mean different things.

**Syntax highlighter**

```ts
get templateJsonHighlighted(): SafeHtml {
  return this.sanitizer.bypassSecurityTrustHtml(this._syntaxHighlight(this.templateJsonString));
}
```

The comment in code is reassuring but **rule of thumb**: every `bypassSecurityTrustHtml` call should be audited. Here it's safe because `templateJsonString` comes from `JSON.stringify` of an internal object, and `_syntaxHighlight` escapes `&`, `<`, `>` before wrapping spans.

> **Scenario — Live editing the template**
>
> 1. Studio has 5 bridges. Preview auto-generates a Scriban template like:
>    ```scriban
>    {
>      "resourceType": "Patient",
>      "id": "{{ source.id }}",
>      "name": [{ "family": "{{ source.lastName }}", "given": ["{{ source.firstName }}"] }]
>    }
>    ```
> 2. User clicks "Edit Template". `isEditMode = true`, `originalTemplate` snapshot saved.
> 3. User adds a custom `"telecom"` array. Clicks "Save Template".
> 4. Component calls `studio.saveScribanTemplate(customTemplate)` → POST `/template/{id}/scriban`.
> 5. The versions panel refreshes — a new entry "v3 by Claude (just now)" appears.

### 5.7 `FhirRenderPreviewComponent`

- **File:** `mapping-studio/fhir-render-preview/fhir-render-preview.ts`
- **Route:** `(modal:render-preview)` — opened via named outlet.
- **Purpose:** A modal that POSTs source JSON to `/template/render-by-id` and shows the backend's rendered output. Useful for "does my template actually produce valid FHIR?" testing.

**Debounced live re-render**

```ts
ngOnInit(): void {
  if (!this.sourceJson.trim()) {
    const raw = this.studio.sourceRawSchema();
    if (raw) {
      try { this.sourceJson = JSON.stringify(JSON.parse(raw), null, 2); }
      catch { this.sourceJson = raw; }
    } else {
      const nodes = this.studio.sourceNodes();
      if (nodes.length) this.sourceJson = JSON.stringify(this._buildSampleValues(nodes), null, 2);
    }
  }
  Promise.resolve().then(() => this.render());           // initial render, deferred to avoid ExpressionChanged…
  this._sourceChange$.pipe(debounceTime(800), distinctUntilChanged()).subscribe(() => this.render());
}
```

**The `Promise.resolve().then(...)` trick** dodges `ExpressionChangedAfterItHasBeenCheckedError`: the constructor sets `isLoading=false`, but `render()` immediately sets it to `true`. Doing it in a microtask defers the second update to the next CD cycle.

> **⚠ Leak:** That second `subscribe()` has no `takeUntilDestroyed` or `ngOnDestroy`. Because this component lives in the `modal` outlet and can be opened/closed repeatedly, every close-reopen cycle leaks a subscription. Fix:
> ```ts
> this._sourceChange$
>   .pipe(debounceTime(800), distinctUntilChanged(), takeUntilDestroyed())
>   .subscribe(() => this.render());
> ```

> **Scenario — Test render**
>
> 1. User in Mapping Studio clicks "Preview". `router.navigate([{outlets:{modal:['render-preview']}}])` → URL becomes `/templates/abc/studio(modal:render-preview)`.
> 2. Component initializes, populates `sourceJson` from `studio.sourceRawSchema()`.
> 3. First render fires after the microtask. Backend returns `{ rendered: '{"resourceType":"Patient","id":"123",…}', success: true, …}`.
> 4. User edits the source JSON. 800 ms later, re-render fires automatically.
> 5. User closes → URL reverts to `/templates/abc/studio`.

---

## 6. Export Bundle (Step 3)

### 6.1 `ExportBundleComponent`

- **File:** `src/app/features/export-bundle/export-bundle.component.ts`
- **Route:** `/export-bundle`
- **Purpose:** Final wizard step. Shows summary (bridge count, crosswalk count, etc.) and downloads a `.zip` bundle.

**Computed properties via signals**

```ts
get crosswalkCount(): number {
  return this.studio.bridges().filter(b => b.enabled && b.transform?.kind === 'codeMap').length;
}

get schemasAvailable(): boolean {
  return this.studio.sourceNodes().length > 0 && this.studio.targetNodes().length > 0;
}
```

**Download flow**

```ts
async downloadBundle(): Promise<void> {
  if (this.isDownloading()) return;
  this.isDownloading.set(true);
  try {
    await this.exportService.downloadBundle();
    this.studio.markAsCompleted();                  // ← marks status: COMPLETED on backend
    this.snack.open('Bundle downloaded — template marked as completed ✓', 'Close', { duration: 4000 });
  } catch (err: any) {
    // … verbose error message extraction
    this.snack.open(`Export failed: ${msg}`, 'Close', { duration: 6000 });
  } finally {
    this.isDownloading.set(false);
  }
}
```

The `ExportBundleService.downloadBundle()` itself builds a `.zip` with **4 files**:

1. `source-schema.json` — raw source schema or parsed nodes.
2. `target-schema.json` — same for target.
3. `crosswalk.json` — fetches all crosswalks referenced by `codeMap` bridges.
4. `template.json` — the canonical template metadata + bridges (or the backend's stored one).

Filename: `{title}_{version}.zip`, e.g. `Patient_to_FHIR_v3.zip`.

> **Scenario:** User finishes mapping, clicks "Download Bundle". 4 HTTP calls fire in parallel (systems, crosswalks, template, scriban). `fflate.zip` builds the archive in memory. `URL.createObjectURL(blob)` → `<a>.click()` → file lands in Downloads folder.

---

## 7. System Management

### 7.1 `System` component

- **File:** `src/app/features/system/system.ts`
- **Route:** `/system`
- **Purpose:** Lists registered source/target systems (e.g. "Salesforce", "FHIR R4 Patient"). Paginated, sorted, soft-delete.
- **⚠ Naming clash:** Class `System` collides with the `System` interface in `system.service.ts` — every consumer aliases the interface as `SystemModel`.

**Pagination via cached page numbers**

```ts
private _totalPages = 0;
private _pageNumbers: number[] = [];

private updatePaginationCache(): void {
  this._totalPages = Math.ceil(this.totalCount / this.pageSize);
  this._pageNumbers = [];
  for (let i = 1; i <= this._totalPages; i++) this._pageNumbers.push(i);
}
```

The pre-computed array avoids running `Array.from({length:N})` on every CD cycle (which would also fail `ExpressionChanged…` if computed in a getter from a template). It's a workaround for the lack of `OnPush` and `trackBy` — both would solve the same problem more cleanly.

**Sort + paginate**

```ts
loadSystems(): void {
  this.loading = true; this.cdr.detectChanges();
  this.systemService.getAllSystems().subscribe({
    next: (systems) => {
      const activeSystems = systems.filter(s => !s.isDeleted);
      activeSystems.sort((a, b) => {
        const aDate = new Date(a.updatedAt ?? a.createdAt ?? 0).getTime();
        const bDate = new Date(b.updatedAt ?? b.createdAt ?? 0).getTime();
        if (bDate !== aDate) return bDate - aDate;
        // tiebreak by createdAt
        return new Date(b.createdAt ?? 0).getTime() - new Date(a.createdAt ?? 0).getTime();
      });
      this.totalCount = activeSystems.length;
      this.updatePaginationCache();
      const start = this.currentPage * this.pageSize;
      this.systems = activeSystems.slice(start, start + this.pageSize);
      this.loading = false;
      this.cdr.detectChanges();
    }
  });
}
```

> **Scenario — Soft delete**
>
> 1. User clicks delete on system "Old Salesforce v1".
> 2. `DeleteSystemDialogComponent` opens with `{ systemName: 'Old Salesforce v1', systemType: 'source' }`.
> 3. On confirm, `systemService.softDeleteSystem(name)` → backend sets `isDeleted: true`.
> 4. Component removes the card locally, decrements `totalCount`, recomputes pagination.
> 5. If current page is now empty and `currentPage > 0`, jumps back one page.

### 7.2 `SystemCardComponent`

Pure presentational:

```ts
@Input({ required: true }) system!: SystemModel;
@Output() edit   = new EventEmitter<SystemModel>();
@Output() delete = new EventEmitter<SystemModel>();
```

Note `@Input({ required: true })` — Angular 16+ syntax that makes the input *mandatory*; missing it causes a compile error.

### 7.3 `RegisterSystemDialogComponent`

- **Purpose:** Create or edit a system. Reactive form with **async duplicate-name validator**.

```ts
this.form = this.fb.group({
  name: ['', [Validators.required, Validators.minLength(2), Validators.maxLength(100)]],
  type: ['', Validators.required],
  description: ['', [Validators.maxLength(500)]]
}, { asyncValidators: [this.duplicateSystemFormValidator.bind(this)] });
```

The async validator hits the API to ensure no duplicate `name + type` combination exists (excluding the current ID in edit mode). Uses `debounceTime` so each keystroke doesn't fire a request.

**Properly tears down its subscriptions:**

```ts
private destroy$ = new Subject<void>();
// … this.form.statusChanges.pipe(takeUntil(this.destroy$)).subscribe(...);
ngOnDestroy(): void { this.destroy$.next(); this.destroy$.complete(); }
```

This component is a good template for how the rest of the codebase **should** be doing subscription cleanup.

### 7.4 `DeleteSystemDialogComponent`

Simple `MAT_DIALOG_DATA: { systemName, systemType }` → boolean confirmation, same pattern as `DeleteTemplateDialogComponent`.

---

## 8. Crosswalk Management

### 8.1 `CrosswalkTablesComponent`

- **File:** `src/app/features/crosswalk/crosswalk-tables.component.ts`
- **Route:** `/crosswalk`
- **Purpose:** Lists crosswalks (vocabulary lookup tables). Each crosswalk maps source codes → target codes (e.g. ICD-9 → ICD-10, gender M/F → male/female).

**Backend response normalization**

```ts
forkJoin({
  crosswalks: this.crosswalkService.getAllCrosswalks(),
  systems:    this.systemService.getAllSystems()
}).subscribe({
  next: (result) => {
    this.crosswalks = (result.crosswalks || []).map((c: any) => ({
      ...c,
      // Handle both camelCase (JS) and PascalCase (C# default)
      defaultSystemUrl: c.defaultSystemUrl ?? c.DefaultSystemUrl ?? undefined,
      mappings: (c.mappings || c.Mappings || []).map((m: any) => ({
        source:        m.source        ?? m.Source        ?? '',
        target:        m.target        ?? m.Target        ?? '',
        targetDisplay: m.targetDisplay ?? m.TargetDisplay ?? undefined,
        targetSystem:  m.targetSystem  ?? m.TargetSystem  ?? undefined,
      })),
    }));
    // … sort by updatedAt desc, fallback to ObjectId timestamp
  }
});
```

This is the most explicit example of the **C# / TS naming-convention mismatch** the codebase has to live with. (`JsonNamingPolicy.CamelCase` in .NET would fix it backend-side; until then, every list endpoint needs this dance.)

> **Scenario:** User clicks `+ New Crosswalk`. `openCrosswalkCreateDialog` opens. They name it "Gender", set source system "Legacy", target system "FHIR", add 3 mappings `M→male`, `F→female`, `U→unknown`. Save. Crosswalk appears in the list, sorted to the top by `updatedAt`.

### 8.2 `CrosswalkCardComponent`

Same shape as `SystemCardComponent`:

```ts
@Input({ required: true }) crosswalk!: Crosswalk;
@Input() count: number = 0;
@Output() edit   = new EventEmitter<Crosswalk>();
@Output() delete = new EventEmitter<Crosswalk>();
@Output() view   = new EventEmitter<Crosswalk>();
```

> **Note:** The `view` output is declared but `onCardClick` is empty. Either wire it up or remove it.

### 8.3 `CrosswalkCreateDialogComponent`

Larger reactive-form dialog. Lets you:
- Name the crosswalk.
- Pick source & target systems.
- Add/remove/edit individual mappings.
- Validate against duplicates via `CrosswalkService.checkMappingConflicts(...)`.

### 8.4 `DeleteCrosswalkDialogComponent`

Same pattern as the other delete dialogs — `{ crosswalkName }` → boolean.

---

## 9. Template Builder

### 9.1 `TemplateBuilderComponent`

- **File:** `src/app/features/template-builder/template-builder.component.ts`
- **Route:** `/template-builder`
- **Purpose:** A **second**, parallel template editor (alongside the Mapping Studio). Targets users who want to write raw Scriban directly with snippet helpers, rather than using the drag-drop workflow.

**Snippet catalog (hardcoded)**

```ts
snippets = [
  { category: 'Crosswalk', icon: 'compare_arrows', items: [
    { label: 'Lookup Value', code: "{{ crosswalk 'MappingName' source.fieldName 'default' }}" },
    { label: 'With Filter',  code: "{{ crosswalk 'MappingName' source.fieldName 'default' | string.downcase }}" },
    …
  ] },
  { category: 'Variables', icon: 'rule', items: [
    { label: 'Assign Variable', code: '{{~ var result = source.fieldName ~}}' },
    …
  ] },
  // Loops, Conditionals
];
```

**Direct DOM textarea manipulation — the worst pattern in the codebase**

```ts
@ViewChild('templateEditor') templateEditor!: ElementRef;

insertSnippet(code: string): void {
  const textarea = this.templateEditor?.nativeElement;
  if (!textarea) return;
  const start = textarea.selectionStart;
  const end   = textarea.selectionEnd;
  const text  = textarea.value;
  const newText = text.substring(0, start) + code + text.substring(end);
  this.templateForm.patchValue({ templateContent: newText });
  setTimeout(() => {
    textarea.focus();
    textarea.setSelectionRange(start + code.length, start + code.length);
  }, 0);
}
```

Direct manipulation of `.nativeElement` is **discouraged** in Angular — it breaks SSR, breaks `OnPush` predictability, and bypasses `Renderer2`. For a snippet inserter, the right fix is either (a) the `MatFormField` + custom directive that updates `selectionStart` through the `FormControl`, or (b) replace `<textarea>` with **CodeMirror** or **Monaco** which has a proper API.

**Field code generator** (a different concern — generates a Scriban snippet from a structured form)

```ts
generateFieldCode(): void {
  const v = this.fieldConfigForm.value;
  if (!v.sourceField || !v.targetField) { this.generatedCode = ''; return; }
  switch (v.transformationType) {
    case 'direct':
      this.generatedCode = `"${v.targetField}": "{{ ${v.sourceField} }}"`;
      break;
    case 'crosswalk':
      if (v.crosswalkName) {
        this.generatedCode = `"${v.targetField}": "{{ crosswalk '${v.crosswalkName}' ${v.sourceField} '${v.defaultValue || 'null'}' }}"`;
      } break;
    case 'ruleset':
      this.generatedCode = `{{~ var ${v.targetField} = evaluate_rule '${v.ruleSetId}' source ~}}{{ ${v.targetField} }}`;
      break;
    case 'custom':
      this.generatedCode = `"${v.targetField}": {{ ${v.customExpression} }}`;
      break;
  }
}
```

> **Scenario:** Power user opens `/template-builder` instead of the studio. They write a custom Scriban template by hand. Click "Crosswalk → Lookup Value" → `insertSnippet("{{ crosswalk '…' source.fieldName 'default' }}")` is inserted at cursor position. Click "Save Template" → currently just logs to console (not wired to backend yet) and shows a snackbar.

> **Note:** `saveTemplate()` ends with `console.log('Save template:', template);` and a snackbar — it doesn't call the backend. This is unfinished functionality.

---

# Part 2 — Utilities

---

## 10. `schema-parse.util.ts`

- **File:** `src/app/utils/schema-parse.util.ts`
- **Exports:** `MAX_FILE_SIZE`, `ALLOWED_EXTENSIONS`, `ALLOWED_MIME`, `parseSchemaFile(file, type)`, `parseSchemaText(text, type)`
- **Purpose:** Parses uploaded files (or pasted text) into the `UploadedSchema` shape consumed by the wizard.

**Algorithm — `parseSchemaFile`**

```ts
1. Check size (max 10 MB).
2. Check extension (.json or .xml only).
3. Read file as text via FileReader.
4. JSON path: JSON.parse() → if it works, count top-level keys, return UploadedSchema.
   XML path: new DOMParser().parseFromString(text, 'application/xml').
             If `<parsererror>` element exists, return error.
             Otherwise count root.children, capture root.tagName.
5. Resolve with { ok: true, schema: UploadedSchema }.
```

**Code (the JSON branch)**

```ts
if (ext === '.json') {
  try {
    const parsed = JSON.parse(rawText);
    const topLevelCount = typeof parsed === 'object' && parsed !== null
      ? Object.keys(parsed).length : 0;
    resolve({
      ok: true,
      schema: {
        type, fileName: file.name, fileSize: file.size,
        mimeType: file.type || 'application/json',
        rawText,
        summary: { detectedFormat: 'json', topLevelCount, isValid: true },
      },
    });
  } catch {
    resolve({ ok: false, error: 'Invalid JSON: file could not be parsed.' });
  }
}
```

> **Scenario:** User uploads `patient.xml` (2 KB). The XML branch fires. `DOMParser` parses successfully — no `<parsererror>` element. `root.tagName = 'Patient'`, `root.children.length = 4`. Returns:
> ```ts
> { type: 'source', fileName: 'patient.xml', fileSize: 2048,
>   mimeType: 'text/xml', rawText: '<Patient>…</Patient>',
>   summary: { detectedFormat: 'xml', rootName: 'Patient', topLevelCount: 4, isValid: true } }
> ```

> **Example error:** User uploads `notes.txt`. Step 2 fails → returns `{ ok: false, error: 'Unsupported file type. Only .json and .xml are allowed.' }`.

---

## 11. `schema-to-nodes.util.ts`

- **File:** `src/app/utils/schema-to-nodes.util.ts` (843 LOC)
- **Main export:** `schemaToNodes(schema: UploadedSchema): SchemaNode[]`
- **Purpose:** Converts an `UploadedSchema` into a tree of `SchemaNode` for the `SchemaTreeComponent`.

**Three branches**

```ts
export function schemaToNodes(schema: UploadedSchema): SchemaNode[] {
  resetSchemaNodeIdCounter();
  if ((schema as any).nodes) return (schema as any).nodes;   // FHIR loader pre-built nodes
  if (schema.summary?.detectedFormat === 'json') {
    const parsed = JSON.parse(schema.rawText);
    if (isFhirStructureDefinition(parsed)) {
      return fhirStructureDefinitionToNodes(parsed);          // FHIR JSON schema
    }
    return jsonToNodes(parsed, '');                           // Plain JSON sample
  }
  const doc = new DOMParser().parseFromString(schema.rawText, 'application/xml');
  return xmlToNodes(doc.documentElement, '');                 // XML
}
```

### 11.1 The plain-JSON path — `jsonToNodes`

```ts
function jsonToNodes(obj: unknown, parentPath: string): SchemaNode[] {
  if (typeof obj !== 'object' || obj === null || Array.isArray(obj)) return [];
  return Object.entries(obj).map(([key, val]) => {
    const path = parentPath ? `${parentPath}.${key}` : key;
    const type = inferType(val);
    if (type === 'object') return { id: path, label: key, path, type, isGroup: true, children: jsonToNodes(val, path) };
    if (type === 'array') {
      const arr = val as unknown[];
      const children = arr.map((item, index) => {
        const itemPath = `${path}[${index}]`;
        if (typeof item === 'object' && item !== null) {
          const preview = getObjectPreview(item) || 'item';
          return { id: itemPath, label: preview, path: itemPath, type: 'object', isGroup: true, children: jsonToNodes(item, itemPath) };
        }
        return { id: itemPath, label: formatPrimitiveValue(item), path: itemPath, type: inferType(item), isGroup: false };
      });
      return { id: path, label: `${key} [${arr.length}]`, path, type, isGroup: true, children };
    }
    return { id: path, label: key, path, type, isGroup: false };
  });
}
```

**Type inference**

```ts
function inferType(val: unknown): FieldType {
  if (val === null || val === undefined) return 'unknown';
  if (typeof val === 'boolean') return 'boolean';
  if (typeof val === 'number')  return 'number';
  if (Array.isArray(val))       return 'array';
  if (typeof val === 'object')  return 'object';
  if (typeof val === 'string')  return /^\d{4}-\d{2}-\d{2}/.test(val) ? 'date' : 'string';
  return 'unknown';
}
```

So a string starting with `2024-01-15` is auto-typed as `date`. Smart heuristic — but be aware it'll trip on values like `"2024-01-15 is when…"`.

> **Example input:**
> ```json
> {
>   "patientId": 123,
>   "name": { "first": "Jane", "last": "Doe" },
>   "visits": [
>     { "date": "2024-01-15", "doctor": "Dr. Smith" },
>     { "date": "2024-03-22", "doctor": "Dr. Jones" }
>   ]
> }
> ```
>
> **Output:**
> ```ts
> [
>   { id: 'patientId', label: 'patientId', path: 'patientId', type: 'number', isGroup: false },
>   { id: 'name', label: 'name', path: 'name', type: 'object', isGroup: true, children: [
>       { id: 'name.first', label: 'first', path: 'name.first', type: 'string' },
>       { id: 'name.last',  label: 'last',  path: 'name.last',  type: 'string' },
>   ]},
>   { id: 'visits', label: 'visits [2]', path: 'visits', type: 'array', isGroup: true, children: [
>     { id: 'visits[0]', label: 'date: "2024-01-15"', path: 'visits[0]', type: 'object', isGroup: true, children: [
>         { id: 'visits[0].date',   label: 'date',   path: 'visits[0].date',   type: 'date'   },
>         { id: 'visits[0].doctor', label: 'doctor', path: 'visits[0].doctor', type: 'string' },
>     ]},
>     { id: 'visits[1]', label: 'date: "2024-03-22"', path: 'visits[1]', type: 'object', isGroup: true, children: [...] },
>   ]}
> ]
> ```

Note how array-item labels use the **first string property** of the object (`date: "2024-01-15"`) as a preview — this is what makes the tree feel readable. Implemented by `getObjectPreview`:

```ts
function getObjectPreview(obj: unknown): string {
  const entries = Object.entries(obj as Record<string, unknown>);
  // Prefer the first string-valued property — type discriminators (e.g. RecordType)
  // are almost always strings.
  const stringEntry = entries.find(([, v]) => typeof v === 'string');
  const [key, value] = stringEntry ?? entries[0];
  return `${key}: ${formatPrimitiveValue(value)}`;
}
```

### 11.2 The FHIR `StructureDefinition` path

`fhirStructureDefinitionToNodes` is far more involved — it walks the FHIR `snapshot.element` array, handling:

- Array detection (`max === '*'` or numeric `> 1`).
- Forbidden elements (`max === '0'`).
- Internal `id` fields at depth > 2 (skip).
- Complex-type expansion (e.g. expanding `HumanName` into its 7 child fields) using the `FHIR_DATA_TYPES` lookup at the top of the file.
- Choice types (`birth[x]` → multiple `birthDate`, `birthDateTime`, etc.).

This is what makes the studio's tree show FHIR resources with their full inner structure rather than just one-level depth.

### 11.3 The XML path — `xmlToNodes`

```ts
function xmlToNodes(el: Element, parentPath: string): SchemaNode[] {
  return Array.from(el.children).map(child => {
    const path = parentPath ? `${parentPath}.${child.tagName}` : child.tagName;
    const hasChildren = child.children.length > 0;
    return {
      id: path, label: child.tagName, path,
      type: (hasChildren ? 'object' : 'string') as FieldType,
      isGroup: hasChildren,
      ...(hasChildren ? { children: xmlToNodes(child, path) } : {}),
    };
  });
}
```

Simpler than JSON: every XML element is either an `object` (has child elements) or a `string` (leaf). Attributes are *not* exposed as separate nodes — a limitation worth noting.

---

## 12. `path-context.util.ts`

- **File:** `src/app/utils/path-context.util.ts` (383 LOC)
- **Purpose:** Resolves a path string ("Patient.name[0].family") into a rich context object showing ancestors, types, FHIR-awareness, and array containment. This is the brain behind the FHIR-aware validation in the studio.

### 12.1 `PathContext` shape

```ts
export interface PathContext {
  segments: PathSegmentInfo[];        // every ancestor + the field itself
  fieldName: string;                  // leaf name
  fieldType: FieldType;               // string / number / array / …
  isInsideArray: boolean;
  arrayAncestors: string[];           // ['Records', 'Identifiers']
}
```

### 12.2 `resolvePathContext` — the workhorse

```ts
export function resolvePathContext(path: string, nodes: SchemaNode[]): PathContext {
  const ancestors: PathSegmentInfo[] = [];
  const node = _findWithAncestors(path, nodes, ancestors);
  const fieldName = node ? _extractName(node) : _nameFromPath(path);
  const fieldType: FieldType = node?.type ?? 'unknown';
  const segments = [...ancestors, { name: [fieldName], type: fieldType }];
  const arrayAncestors = ancestors.filter(s => s.type === 'array')
                                   .map(s => s.name.join('.'));
  return { segments, fieldName, fieldType, isInsideArray: arrayAncestors.length > 0, arrayAncestors };
}
```

It walks the tree depth-first; every time it descends into a child it **pushes an ancestor** (and pops back on the way up). Array items decorate the parent array's `arrayItemLabel` rather than pushing their own ancestor — that's important because indices like `[0]` aren't logical ancestors.

### 12.3 String-only fallback

```ts
export function resolvePathContextFromString(path: string): PathContext {
  const parts = path.split('.');
  const segments: PathSegmentInfo[] = [];
  const arrayAncestors: string[] = [];
  for (let i = 0; i < parts.length; i++) {
    const clean = parts[i].replace(/\[\d+\]/g, '');
    const isArray = /\[\d+\]/.test(parts[i]);
    const type: FieldType = isArray ? 'array' : (i < parts.length - 1 ? 'object' : 'string');
    segments.push({ name: [clean], type });
    if (isArray) arrayAncestors.push(clean);
  }
  // …
}
```

Used when schema nodes are unavailable — heuristically infers types from the path structure alone.

### 12.4 FHIR type compatibility

```ts
const _FHIR_TYPE_COMPAT: Record<string, (fhir: string) => boolean> = {
  string:  (f) => _FHIR_PRIMITIVE_SET.has(f) || _FHIR_COMPLEX_SET.has(f),
  boolean: (f) => f === 'boolean' || f === 'string',
  number:  (f) => f === 'integer' || f === 'decimal' || f === 'unsignedInt' || f === 'positiveInt' || f === 'string',
  date:    (f) => f === 'date' || f === 'dateTime' || f === 'instant' || f === 'string',
  array:   (f) => f === 'unknown',
  object:  (f) => f === 'BackboneElement' || f === 'Element' || f === 'object',
  unknown: (_) => true,
};

export function isPathTypeCompatible(fieldType: FieldType, fhirType: string): boolean {
  const checker = _FHIR_TYPE_COMPAT[fieldType];
  return checker ? checker(fhirType) : false;
}
```

Each source `FieldType` knows which FHIR types it can safely map to. `validation.util.ts` uses this to decide whether a bridge needs a transform.

> **Scenario — resolve a deep array path**
>
> Input: `path = "Patient.identifier[0].value"`, with schema where `identifier` is an array of objects.
>
> Trace:
> 1. Find `Patient` → push ancestor `{ name: ['Patient'], type: 'object' }`.
> 2. Recurse into `Patient.children`. Find `identifier` (type `array`) → push ancestor `{ name: ['identifier'], type: 'array' }`.
> 3. Find `identifier[0]` (array item) → decorate the previous array ancestor's `arrayItemLabel`. Do NOT push new ancestor.
> 4. Recurse into `identifier[0].children`. Find `value`.
>
> Output:
> ```ts
> {
>   segments: [
>     { name: ['Patient'], type: 'object' },
>     { name: ['identifier'], type: 'array', arrayItemLabel: { 'use': 'use: "official"' } },
>     { name: ['value'], type: 'string' }
>   ],
>   fieldName: 'value',
>   fieldType: 'string',
>   isInsideArray: true,
>   arrayAncestors: ['identifier']
> }
> ```

### 12.5 Other utilities exported

| Function | Use |
|---|---|
| `normalizePathIndices(path)` | Strip `[N]` → returns `"identifier.value"` |
| `inferFhirResourceType(path)` | First PascalCase segment, e.g. `"Patient"` |
| `isFhirResourcePath(path)` | True if path looks FHIR-shaped |
| `getCommonAncestorPath(p1, p2)` | Longest shared prefix |
| `findNodeAtPath(path, nodes)` | DFS for a node |
| `flattenAllNodes(nodes)` | All nodes as flat array |
| `flattenLeafPaths(nodes, maxDepth?)` | Leaves only |
| `buildNodeIndex(nodes)` | `Map<path, SchemaNode>` for O(1) lookups |
| `findNodesByType(type, nodes)` | All nodes of a given `FieldType` |
| `getPathSuggestions(query, nodes)` | Ranked search (score 100/80/60/40) |
| `resolveMultiPaths(paths, nodes)` | Batch resolve |
| `resolvePathContextEnriched(path, nodes)` | Adds `fhirResourceType`, `normalizedPath`, `fhirLeafType`, etc. |

---

## 13. `validation.util.ts`

- **File:** `src/app/utils/validation.util.ts` (205 LOC)
- **Purpose:** Bridge-level validation logic. Determines if a single bridge or a whole set is valid.

### 13.1 `validateBridge` — per-bridge check

```ts
const COMPATIBLE: Partial<Record<FieldType, FieldType[]>> = {
  string:  ['string', 'number', 'boolean', 'date', 'unknown'],
  number:  ['number', 'string', 'unknown'],
  boolean: ['boolean', 'string', 'unknown'],
  date:    ['date', 'string', 'unknown'],
  array:   ['array', 'unknown'],
  object:  ['object', 'unknown'],
  unknown: ['string', 'number', 'boolean', 'date', 'array', 'object', 'unknown'],
};

export function validateBridge(bridge: Bridge, srcNodes: SchemaNode[], tgtNodes: SchemaNode[]): Bridge {
  const src = findNode(bridge.sourcePath, srcNodes);
  const tgt = findNode(bridge.targetPath, tgtNodes);
  if (!src || !tgt) return { ...bridge, status: 'invalid', message: 'Path not found in schema.' };
  if (tgt.type === 'array' && src.type !== 'array') {
    return { ...bridge, status: 'warning', message: `Target is array — consider wrapInArray transform.` };
  }
  if (!(COMPATIBLE[src.type] ?? []).includes(tgt.type)) {
    return { ...bridge, status: 'warning', message: `Type mismatch: ${src.type} → ${tgt.type}. A transform is recommended.` };
  }
  return { ...bridge, status: 'valid', message: undefined };
}
```

> **Examples:**
>
> | Source type | Target type | Result | Message |
> |---|---|---|---|
> | `string` | `string` | `valid` | — |
> | `string` | `date` | `valid` (string→date is allowed) | — |
> | `boolean` | `date` | `warning` | "Type mismatch: boolean → date." |
> | `string` | `array` | `warning` | "Target is array — consider wrapInArray transform." |
> | `string` | (path not found) | `invalid` | "Path not found in schema." |

### 13.2 `validateBridgeSet` — set-level checks

Detects three problems the per-bridge check can't:

```ts
export function validateBridgeSet(bridges, srcNodes, tgtNodes): BridgeSetReport {
  const enabled = bridges.filter(b => b.enabled !== false);
  const validated = enabled.map(b => validateBridge(b, srcNodes, tgtNodes));

  // Duplicates
  const duplicateSourcePaths = [...countMap(enabled.map(b => b.sourcePath))]
                                .filter(([, c]) => c > 1).map(([p]) => p);
  const duplicateTargetPaths = [...countMap(enabled.map(b => b.targetPath))]
                                .filter(([, c]) => c > 1).map(([p]) => p);

  // Required target fields with no mapping
  const mappedTargets = new Set(enabled.map(b => b.targetPath));
  const unmappedRequiredTargets = findRequiredUnmapped(tgtNodes, mappedTargets);

  // …
}
```

`duplicateSourcePaths` = same field mapped twice (potential data duplication).
`duplicateTargetPaths` = two sources writing the same target (last-write-wins collision).
`unmappedRequiredTargets` = schema fields marked `required: true` with no active bridge.

### 13.3 `suggestTransformKind` — heuristic

```ts
export function suggestTransformKind(sourceType: FieldType, targetFhirType: string): string {
  if (sourceType === 'string' && ['date','dateTime','instant'].includes(targetFhirType)) return 'dateFormat';
  if (sourceType === 'date'   && targetFhirType === 'string') return 'dateFormat';
  if (['CodeableConcept', 'Coding'].includes(targetFhirType)) return 'codeMap';
  if (targetFhirType === 'code' && sourceType !== 'string') return 'codeMap';
  if (sourceType === 'number' && targetFhirType === 'string') return 'customScript';
  if (isPathTypeCompatible(sourceType, targetFhirType)) return 'none';
  return 'customScript';
}
```

When the user creates a bridge whose types don't match, the dialog can pre-select a sensible transform.

### 13.4 `findConflictingBridges` — FHIR-resource-aware

```ts
export function findConflictingBridges(bridges: Bridge[]): Array<[Bridge, Bridge]> {
  const enabled = bridges.filter(b => b.enabled !== false);
  const conflicts: Array<[Bridge, Bridge]> = [];
  for (let i = 0; i < enabled.length; i++)
    for (let j = i + 1; j < enabled.length; j++) {
      const sameResource = inferFhirResourceType(enabled[i].targetPath)
                        === inferFhirResourceType(enabled[j].targetPath);
      if (sameResource && enabled[i].targetPath === enabled[j].targetPath)
        conflicts.push([enabled[i], enabled[j]]);
    }
  return conflicts;
}
```

O(n²) but `n` is the bridge count (typically < 100). Fine.

---

## 14. `date-format.util.ts`

- **File:** `src/app/utils/date-format.util.ts` (47 LOC)
- **Export:** `dotnetToStrftime(fmt: string): string`
- **Purpose:** Convert a .NET-style format like `"yyyy-MM-dd HH:mm"` to a strftime/Scriban-compatible `"%Y-%m-%d %H:%M"`.

### 14.1 The whole function (it's small enough)

```ts
export function dotnetToStrftime(fmt: string): string {
  if (!fmt || fmt.includes('%')) return fmt; // already strftime
  let result = fmt;

  // Year
  result = result.replace(/yyyy/g, '%Y');
  result = result.replace(/yy/g,   '%y');

  // Month (uppercase M) — must come BEFORE minutes (lowercase mm)
  result = result.replace(/MM/g, '%m');
  result = result.replace(/M/g,  '%m');

  // Day — replace dd first, then guard against re-matching 'd' inside '%d'
  result = result.replace(/dd/g,      '%d');
  result = result.replace(/(?<!%)d/g, '%d');

  // Hour 24h
  result = result.replace(/HH/g,      '%H');
  result = result.replace(/(?<!%)H/g, '%H');

  // Hour 12h
  result = result.replace(/hh/g,      '%I');
  result = result.replace(/(?<!%)h/g, '%I');

  // Minutes (lowercase mm — must come after MM is consumed)
  result = result.replace(/mm/g, '%M');

  // Seconds
  result = result.replace(/ss/g, '%S');

  // AM/PM
  result = result.replace(/tt/g, '%p');

  return result;
}
```

### 14.2 The order trap

The function has a **carefully chosen replacement order** because:

1. **`MM` vs `mm`**: in .NET, `MM` = Month, `mm` = minutes. If you ran `mm → %M` first, `MM` would already be `MM` and the next pass `MM → %m` would also match the leftover `M`s. Doing `MM → %m` first leaves nothing for the minute pass to confuse.
2. **`d` inside `%d`**: after `dd → %d`, the single-character pass `d → %d` would re-match the `d` *inside* `%d`. The lookbehind `(?<!%)d` skips it.

### 14.3 Examples

| Input | Output | Notes |
|---|---|---|
| `"yyyy-MM-dd"` | `"%Y-%m-%d"` | Standard ISO date |
| `"yyyy-MM-dd HH:mm:ss"` | `"%Y-%m-%d %H:%M:%S"` | DateTime |
| `"dd/MM/yyyy"` | `"%d/%m/%Y"` | European date |
| `"M/d/yy"` | `"%m/%d/%y"` | US short |
| `"hh:mm tt"` | `"%I:%M %p"` | 12h with AM/PM |
| `"%Y-%m-%d"` | `"%Y-%m-%d"` | Already strftime; no change |
| `""` | `""` | Empty stays empty |

> **Scenario:** Bridge has `transform.kind = 'dateFormat'`, `params['format'] = 'yyyy-MM-dd'`. The Scriban emitter calls `dotnetToStrftime` to convert to `'%Y-%m-%d'`, then writes `{{ source.birthDate | date.to_string '%Y-%m-%d' }}`.

---

## 15. `template-to-scriban.util.ts`

- **File:** `src/app/utils/template-to-scriban.util.ts` (1,270 LOC)
- **Main export:** `convertTemplateJsonToScriban(input: TemplateJsonInput): string`
- **Purpose:** Convert the canonical template JSON shape (from `ValidationPreview.templateJson`) into an executable Scriban template string.

> This is the biggest utility in the codebase. The walkthrough here is high-level — the algorithm is complex enough that a full line-by-line would be its own document.

### 15.1 Input shape

```ts
export interface TemplateJsonInput {
  title?: string;
  templateId?: string;
  sourceSystem?: string;
  targetSystem?: string;
  tags?: string[];
  coverage?: number;
  status?: string;
  bridges: TemplateBridge[];           // ← the meat
  sourceSchema?: unknown;
  targetSchema?: unknown;
  rootAccessor?: string;               // default 'input'
}

export interface TemplateBridge {
  sourcePath: string;
  targetPath: string;
  transform: { kind: string; params?: Record<string, string> };
  sourceContext?: { … };
  targetContext?: { … };
}
```

### 15.2 The top-level branch

```ts
export function convertTemplateJsonToScriban(input: TemplateJsonInput): string {
  if (!input.bridges?.length) return '{}';
  const rootAccessor = input.rootAccessor ?? 'input';

  // 1. Collect variable assignments (rule sets + crosswalk pre-loads)
  const { block: variableBlock, ids: evaluatedIds } = _collectVariableAssignments(bridges, rootAccessor);

  // 2. Detect EntityRecordCollection pattern
  const collectionGroups = _detectCollectionGroups(bridges, rootAccessor);
  if (collectionGroups.length > 0) {
    return _buildCollectionScriban(collectionGroups, input, variableBlock, evaluatedIds);
  }

  // 3. Default path
  // … build a JSON resource object, populated by per-bridge logic
}
```

There are **two main code paths**:

- **Collection pattern**: when bridges share an outer collection + records-array + record-type pattern (e.g. `EntityRecordCollections[].Records[].Field` with `params.recordType` filtering). Emits nested `{{~ for _coll in input.EntityRecordCollections ~}} {{~ if _coll.RecordType == 'X' ~}} {{~ for _rec in _coll.Records ~}}` loops.
- **Default path**: flat field-by-field generation.

### 15.3 Collection pattern in action

> **Example input:**
> ```ts
> bridges: [
>   { sourcePath: 'EntityRecordCollection.Records.MEME_CK',
>     targetPath: 'Patient.id',
>     transform: { kind: 'none', params: { recordType: 'MEME' } } },
>   { sourcePath: 'EntityRecordCollection.Records.MEME_SFX',
>     targetPath: 'Patient.gender',
>     transform: { kind: 'codeMap', params: { recordType: 'MEME', crosswalkName: 'Gender' } } },
> ]
> ```
>
> **Output Scriban:**
> ```scriban
> {{~ for _coll in input.EntityRecordCollection ~}}
>   {{~ if _coll.RecordType == "MEME" ~}}
>     {{~ for _rec in _coll.Records ~}}
> {
>   "resourceType": "Patient",
>   "id": "{{ _rec.MEME_CK }}",
>   "gender": "{{ crosswalk 'Gender' _rec.MEME_SFX 'unknown' }}"
> }
>     {{~ end ~}}
>   {{~ end ~}}
> {{~ end ~}}
> ```

### 15.4 Default path — flat resource

> **Example input:**
> ```ts
> bridges: [
>   { sourcePath: 'patient.id', targetPath: 'Patient.id', transform: { kind: 'none' } },
>   { sourcePath: 'patient.first', targetPath: 'Patient.name.given', transform: { kind: 'none' } },
>   { sourcePath: 'patient.last', targetPath: 'Patient.name.family', transform: { kind: 'none' } },
> ]
> ```
>
> **Output Scriban:**
> ```scriban
> {
>   "resourceType": "Patient",
>   "id": "{{ input.patient.id }}",
>   "name": {
>     "given": "{{ input.patient.first }}",
>     "family": "{{ input.patient.last }}"
>   }
> }
> ```

### 15.5 Variable block — for rule-sets

If any bridge references a `ruleSetId`, the generator emits a top-of-file variable block:

```scriban
{{~ var gender_lookup = evaluate_rule 'GENDER_RULES' input ~}}
{{~ var name_format  = evaluate_rule 'NAME_RULES'   input ~}}
{
  "gender": "{{ gender_lookup }}",
  "name":   "{{ name_format }}",
  ...
}
```

This is why `_collectVariableAssignments(bridges, rootAccessor)` runs *before* the rest — both branches need access to which IDs have already been evaluated.

### 15.6 Transform-kind dispatch

Each bridge's value expression is dispatched on `transform.kind`:

| Kind | Example output |
|---|---|
| `none` | `"{{ input.field }}"` |
| `static` | `"FixedString"` |
| `concat` | `"{{ input.first + ' ' + input.last }}"` |
| `dateFormat` | `"{{ input.dob \| date.to_string '%Y-%m-%d' }}"` (uses `dotnetToStrftime`) |
| `codeMap` | `"{{ crosswalk 'GenderMap' input.sex 'unknown' }}"` |
| `customScript` | passes through `params['expression']` or `params['valueMapRules']` Scriban |

### 15.7 Scenario — end-to-end

> 1. User finishes mapping in the studio. `ValidationPreview.templateJson` builds the canonical JSON.
> 2. User clicks "View Scriban Template". The component calls `convertTemplateJsonToScriban(templateJson)`.
> 3. The util detects a `recordType` param → collection pattern.
> 4. Emits the nested for/if/for Scriban structure.
> 5. Result is shown in the preview pane with `_syntaxHighlight()`.
> 6. User clicks "Save Template" → POST `/template/{id}/scriban` with the same string.
> 7. Later, the backend renders by running this template against incoming source data.

---

## Appendix — Files by Size

For quick orientation:

| File | LOC | Concentration of logic |
|---|---:|---|
| `field-connection-dialog.ts` | 1,864 | 4 patterns × 15 operators × 3 logic modes |
| `template-to-scriban.util.ts` | 1,270 | Scriban codegen, collection pattern detection |
| `studio-state.service.ts` | 1,123 | Bridge store + HTTP + coverage |
| `mapping-studio.ts` | 431 | 4-branch init, drag-drop, dialog orchestration |
| `validation-preview.ts` | 819 | Template generation, highlighting, version mgmt |
| `schema-to-nodes.util.ts` | 843 | JSON / XML / FHIR-StructureDefinition parsing |
| `scriban-template-emitter.service.ts` | 829 | Bridge → Scriban string emission |
| `dashboard.component.ts` | 287 | Stats + project list + per-row actions |
| `path-context.util.ts` | 383 | Tree walking + FHIR type inference |
| `system.ts` | 209 | Paginated CRUD list |
| `validation.util.ts` | 205 | Per-bridge + set-level validation |
| `bridge-card.ts` | 347 | Rehydration of stored params |
| `fhir-render-preview.ts` | 335 | Modal backend render |
| `schema-tree.ts` | 152 | Flat tree + drag-drop + search |
| `dashboard.service.ts` | 308 | Stats aggregation + coverage fallback |
| `stepper.ts` | 27 | Active-step-only emit |
| `date-format.util.ts` | 47 | .NET → strftime |

---

*End of deep dive.*