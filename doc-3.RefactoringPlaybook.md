# Mapping Studio UI — Refactoring Playbook

>
> For each oversized file you get:
>
> 1. **Why it's a problem** (size, mixed concerns, change-frequency hotspots)
> 2. **The natural seams** (what cohesive groups already exist in the file)
> 3. **The proposed split** (new file/folder layout, with names)
> 4. **Before/after code** (real snippets, not pseudocode)
> 5. **Migration plan** (the safe order of moves)
> 6. **Tests & risks** (what to watch for)

---

## Files Ranked by Refactor Priority

| Rank | File | LOC | Refactor priority | Effort |
|---|---|---:|---|---|
| 1 | `field-connection-dialog.ts` | **1,864** | 🔴 Critical | High (2-3 weeks) |
| 2 | `template-to-scriban.util.ts` | **1,270** | 🔴 Critical | Medium |
| 3 | `studio-state.service.ts` | **1,123** | 🔴 Critical | Medium |
| 4 | `schema-to-nodes.util.ts` | **843** | 🟡 High | Medium |
| 5 | `scriban-template-emitter.service.ts` | **829** | 🟡 High | Medium |
| 6 | `validation-preview.ts` | **819** | 🟡 High | Medium |
| 7 | `mapping-studio.ts` | 432 | 🟢 Medium | Low |
| 8 | `dashboard.component.ts` | 287 | 🟢 Low (good already) | Low |
| 9 | `template-builder.component.ts` | 274 | 🟡 Medium | Low |

The rest of the codebase is below 300 LOC per file and doesn't need structural splits — it just needs the cleanup described in the analysis doc.

---

## Table of Contents

1. [Refactor 1 — `field-connection-dialog.ts` (1,864 LOC)](#refactor-1--field-connection-dialogts)
2. [Refactor 2 — `template-to-scriban.util.ts` (1,270 LOC)](#refactor-2--template-to-scribanutilts)
3. [Refactor 3 — `studio-state.service.ts` (1,123 LOC)](#refactor-3--studio-stateserviceservicets)
4. [Refactor 4 — `schema-to-nodes.util.ts` (843 LOC)](#refactor-4--schema-to-nodesutilts)
5. [Refactor 5 — `scriban-template-emitter.service.ts` (829 LOC)](#refactor-5--scriban-template-emitterservicets)
6. [Refactor 6 — `validation-preview.ts` (819 LOC)](#refactor-6--validation-previewts)
7. [Refactor 7 — `mapping-studio.ts` (432 LOC)](#refactor-7--mapping-studiots)
8. [Refactor 8 — `template-builder.component.ts` (274 LOC)](#refactor-8--template-buildercomponentts)
9. [Cross-Cutting Enhancements](#cross-cutting-enhancements)
10. [Recommended Folder Layout (Target State)](#recommended-folder-layout-target-state)

---

## Refactor 1 — `field-connection-dialog.ts`

### 1.1 Why it must be split

This is **the worst code-smell in the project**: a single class with:

- **130+ methods** across the class body.
- **40+ instance fields** (UI state, search queries, dropdowns, dates, operators, pre-process pipeline, rules, conditions, source/target paths…).
- **6 distinct responsibilities** mashed together (see below).
- An `onCommit()` method that's **94 lines long** and builds a 25-key emit payload from 4 disjoint pieces of state.

Every change becomes risky because **anything in the file can affect anything else**. The spec file is also enormous (~270 KB), proportional to the surface area.

### 1.2 The 6 responsibilities already present in the file

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. PATH SELECTION                                                   │
│    selectedSourcePath[s], selectedTargetPath[s], drag-drop, search   │
│    activePattern (one-to-one / many-to-one / one-to-many / m-to-m)  │
├─────────────────────────────────────────────────────────────────────┤
│ 2. CROSSWALK SELECTION                                              │
│    availableCrosswalks, selectedCrosswalkName, loadCrosswalks()     │
├─────────────────────────────────────────────────────────────────────┤
│ 3. RULESET SELECTION  (deprecated, but still wired)                 │
│    availableRuleSets, selectedRuleSetIds, transformRuleSets         │
├─────────────────────────────────────────────────────────────────────┤
│ 4. SCRIBAN OPERATORS                                                │
│    selectedOperatorValue, operatorParams, activeOpCategories         │
│    pre-process pipeline, source expression overrides                │
├─────────────────────────────────────────────────────────────────────┤
│ 5. DATE CARD UI                                                     │
│    DATE_CARDS, calcAgeRefType, formatDateOutputFormat, dateDiffUnit │
├─────────────────────────────────────────────────────────────────────┤
│ 6. CONDITIONAL VALUE-MAP RULES                                      │
│    valueMapRules[], conditions[], operators (equals/in/matches/…)   │
└─────────────────────────────────────────────────────────────────────┘
```

Each box is a **cohesive sub-feature** that today sees no separation.

### 1.3 Proposed split — child components + state services

```
mapping-studio/field-connection-dialog/
├── field-connection-dialog.ts          (was 1864 → ~250 LOC, orchestration only)
├── field-connection-dialog.html        (split into per-tab partial templates)
├── field-connection-dialog.css
├── field-connection-dialog.spec.ts
│
├── state/
│   ├── connection-form.store.ts        ← NEW: signal-based "view model"
│   ├── transformation-pattern.ts       ← TransformationPattern, LogicType (move from main file)
│   └── value-map-rule.ts               ← ValueMapRule, ValueMapCondition (move from main file)
│
├── tabs/
│   ├── source-tab/
│   │   ├── source-tab.component.ts     ← Path 1: source selection
│   │   ├── source-tab.component.html
│   │   └── source-tab.component.css
│   ├── crosswalk-tab/
│   │   └── crosswalk-tab.component.ts  ← Path 2: crosswalk select
│   ├── transform-tab/
│   │   ├── transform-tab.component.ts  ← Path 4: operator picker + pre-process
│   │   ├── operator-picker.component.ts
│   │   ├── pre-process-pipeline.component.ts
│   │   └── date-card/                  ← Path 5: date cards
│   │       ├── date-card.component.ts
│   │       └── date-card.types.ts
│   ├── value-map-rules/
│   │   ├── value-map-rules.component.ts ← Path 6: conditional rules
│   │   ├── rule-row.component.ts
│   │   └── condition-row.component.ts
│   └── target-tab/
│       └── target-tab.component.ts     ← Path 1 (target side)
│
└── scriban-operators.constants.ts      ← (already separate, leave it)
```

### 1.4 The key idea: a `ConnectionFormStore`

Today every piece of state is a plain field on the component. The smart move is to extract a **signal-based view-model store** that all child tabs read from. Each tab becomes a thin presenter.

#### Before (excerpt from current file)

```ts
// field-connection-dialog.ts (1864 LOC, abridged)
export class FieldConnectionDialogComponent implements OnInit, OnChanges, OnDestroy {
  @Input() sourcePath = '';
  @Input() targetPath = '';
  @Input() sourceNodes: SchemaNode[] = [];
  // … 6 more inputs

  selectedSourcePath  = '';
  selectedSourcePaths: string[] = [];
  selectedTargetPath  = '';
  selectedTargetPaths: string[] = [];

  // 1. Path selection
  activePattern: TransformationPattern = 'one-to-one';
  showSourceBrowser = false; showTargetBrowser = false;
  sourceSearchQuery = ''; targetSearchQuery = '';

  // 2. Crosswalk
  availableCrosswalks: Crosswalk[] = [];
  selectedCrosswalkName: string | null = null;

  // 3. RuleSet
  selectedRuleSetIds: string[] = [];
  rulesetDropdownOpen = false;
  availableRuleSets: any[] = [];
  collectionRuleSets: any[] = [];
  transformRuleSets: any[] = [];

  // 4. Operators
  selectedOperatorValue = 'direct';
  operatorParams: Record<string, string> = {};
  activeOpCategories: string[] = ['date'];
  operatorPickerOpen = false;
  operatorPickerQuery = '';
  sourceExpressions: Record<string, string> = {};

  // 5. Date cards
  selectedDateCard: string | null = null;
  calcAgeRefType: 'today' | 'specific' = 'today';
  calcAgeSpecificDate = '';
  formatDateOutputFormat = '';
  dateDiffBaseType: 'today' | 'specific' = 'today';
  dateDiffSpecificDate = '';
  dateDiffUnit = 'days';

  // 6. Conditional rules
  valueMapRules: ValueMapRule[] = [];
  private _ruleIdCounter = 0;

  // …130 methods follow, freely manipulating any of these fields…
  
  onCommit(): void {
    // 94 lines of nested conditionals reading from ALL six groups
  }
}
```

#### After: extract `ConnectionFormStore` (signal-based view model)

```ts
// state/connection-form.store.ts (~200 LOC, single responsibility)
import { Injectable, computed, signal } from '@angular/core';

@Injectable()  // NOT providedIn:'root' — scoped to the dialog
export class ConnectionFormStore {
  // ── 1. Path selection ─────────────────────────────────────────────
  readonly activePattern   = signal<TransformationPattern>('one-to-one');
  readonly sourcePaths     = signal<string[]>([]);
  readonly targetPaths     = signal<string[]>([]);
  readonly sourceSearch    = signal('');
  readonly targetSearch    = signal('');

  // ── 2. Crosswalk ───────────────────────────────────────────────────
  readonly crosswalkName   = signal<string | null>(null);

  // ── 3. RuleSet (deprecated) ────────────────────────────────────────
  readonly ruleSetIds      = signal<string[]>([]);

  // ── 4. Operators ───────────────────────────────────────────────────
  readonly operatorValue   = signal<string>('direct');
  readonly operatorParams  = signal<Record<string, string>>({});
  readonly preProcessSteps = signal<PreProcessStep[]>([]);
  readonly sourceOverrides = signal<Record<string, string>>({});

  // ── 5. Date cards ──────────────────────────────────────────────────
  readonly dateCard        = signal<DateCardState>({ kind: null });

  // ── 6. Conditional rules ───────────────────────────────────────────
  readonly valueMapRules   = signal<ValueMapRule[]>([]);

  // Derived
  readonly transformationType = computed<'none' | 'transform' | 'crosswalk'>(() => {
    if (this.crosswalkName())                return 'crosswalk';
    if (this.operatorValue() !== 'direct')   return 'transform';
    if (this.valueMapRules().length > 0)     return 'transform';
    return 'none';
  });
  
  readonly canCommit = computed(() => 
    this.sourcePaths().length > 0 && this.targetPaths().length > 0
  );

  // Mutations (one per logical action)
  setPattern(p: TransformationPattern): void  { this.activePattern.set(p); }
  selectSource(path: string): void            { this.sourcePaths.update(arr => [...arr, path]); }
  removeSource(i: number): void               { this.sourcePaths.update(arr => arr.filter((_, idx) => idx !== i)); }
  // … per-field setters

  /** Build emit payload — the only place onCommit serialization lives now. */
  toCommitPayload(): CommitEvent { /* moved from the 94-line onCommit() */ }
  
  /** Inverse — hydrate from an existing Bridge. */
  hydrateFromBridge(b: Bridge): void { /* moved from _hydrateFromBridge */ }
}
```

```ts
// field-connection-dialog.ts (NEW — ~250 LOC, orchestration only)
import { Component, Inject, OnInit } from '@angular/core';
import { ConnectionFormStore } from './state/connection-form.store';

@Component({
  selector: 'app-field-connection-dialog',
  standalone: true,
  providers: [ConnectionFormStore],          // ← scoped to this dialog only
  imports: [
    CommonModule,
    SourceTabComponent,                       // ← split children
    TargetTabComponent,
    CrosswalkTabComponent,
    TransformTabComponent,
  ],
  templateUrl: './field-connection-dialog.html',
})
export class FieldConnectionDialogComponent implements OnInit {
  @Input() sourcePath = '';
  @Input() targetPath = '';
  @Input() sourceNodes: SchemaNode[] = [];
  @Input() targetNodes: SchemaNode[] = [];
  @Input() existingBridge: Bridge | null = null;
  
  @Output() commit = new EventEmitter<CommitEvent>();
  @Output() close  = new EventEmitter<void>();

  activeTab: 'fields' | 'logic' = 'fields';

  constructor(public store: ConnectionFormStore) {}

  ngOnInit(): void {
    if (this.existingBridge) {
      this.store.hydrateFromBridge(this.existingBridge);
    } else {
      if (this.sourcePath) this.store.selectSource(this.sourcePath);
      if (this.targetPath) this.store.selectSource(this.targetPath);
    }
  }

  onCommit(): void {
    if (!this.store.canCommit()) return;
    this.commit.emit(this.store.toCommitPayload());
  }
}
```

```html
<!-- field-connection-dialog.html (NEW — declarative composition) -->
<div class="dialog">
  <header>
    <button (click)="activeTab = 'fields'" [class.active]="activeTab === 'fields'">Fields</button>
    <button (click)="activeTab = 'logic'"  [class.active]="activeTab === 'logic'">Logic</button>
  </header>

  @if (activeTab === 'fields') {
    <app-source-tab [nodes]="sourceNodes" />
    <app-target-tab [nodes]="targetNodes" />
  } @else {
    <app-crosswalk-tab />
    <app-transform-tab />
  }

  <footer>
    <button (click)="close.emit()">Cancel</button>
    <button [disabled]="!store.canCommit()" (click)="onCommit()">Save</button>
  </footer>
</div>
```

```ts
// tabs/source-tab/source-tab.component.ts (~120 LOC)
@Component({
  selector: 'app-source-tab',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './source-tab.component.html',
})
export class SourceTabComponent {
  @Input() nodes: SchemaNode[] = [];

  // ← the store is INJECTED FROM THE PARENT because @Injectable() not 'root'
  // so this child sees the SAME store instance as the dialog
  constructor(public store: ConnectionFormStore) {}

  onSelectNode(node: SchemaNode): void {
    this.store.selectSource(node.path);
  }

  isSelected(path: string): boolean {
    return this.store.sourcePaths().includes(path);
  }
}
```

### 1.5 Migration plan (safe order)

1. **Step 1 — extract types** (10 min, no behavior change)
   - Move `TransformationPattern`, `LogicType`, `ValueMapCondition`, `ValueMapRule`, `PreProcessStep` into `state/connection-form.types.ts`.
   - Import them back where used.

2. **Step 2 — create the empty store** (1 hr)
   - Create `ConnectionFormStore` with all signals defined, but `hydrateFromBridge` / `toCommitPayload` empty.
   - Provide it on the dialog component. Inject into the existing monolith.
   - Replace 5-10 fields per commit with `store.x()` reads and `store.setX()` writes. Verify the dialog still works after each pass.

3. **Step 3 — move `_hydrateFromBridge` and `onCommit` into the store** (half-day)
   - These two are the gnarliest because they touch every group. Moving them last means the store has settled.

4. **Step 4 — extract child components, one tab at a time** (1-2 days each)
   - Start with `SourceTabComponent` because it's the most isolated.
   - Then `CrosswalkTabComponent` (small).
   - Then `TransformTabComponent` (has the operator picker + pre-process).
   - Then `ValueMapRulesComponent` (has its own row-level state).
   - Then `DateCardComponent` (nested inside Transform).

5. **Step 5 — final cleanup** (half-day)
   - Delete the deprecated `RuleSetService` injection.
   - Delete `availableRuleSets`, `collectionRuleSets`, `transformRuleSets` arrays.
   - Audit the spec file — split it to mirror the new components.

### 1.6 Risks

| Risk | Mitigation |
|---|---|
| Breaking the rehydration of saved bridges | Keep the existing `_hydrateFromBridge` test cases; they exercise the round-trip. |
| Template-driven form values lost on tab switch | Store is the source of truth; child components don't hold local state. |
| Two-way binding patterns scattered through template | Convert `[(ngModel)]` to `[ngModel]` + `(ngModelChange)="store.setX($event)"`. |

### 1.7 Result

| Metric | Before | After (target) |
|---|---|---|
| Files | 1 | 12 |
| Largest file | 1,864 LOC | ~300 LOC (the store) |
| Average file | 1,864 LOC | ~150 LOC |
| Spec file size | ~270 KB | 6 specs × ~30 KB |
| Diff radius for a "new operator" change | All 1,864 lines | ~80 lines in `operator-picker` |

---

## Refactor 2 — `template-to-scriban.util.ts`

### 2.1 Why it must be split

1,270 LOC of pure functions, **38 module-level functions**, two completely separate code paths (collection vs default), plus orthogonal helpers (path cleaning, JSON formatting, expression building, FHIR pipe processing). A change to "how dateFormat is rendered" requires reading 1,270 lines to find the right spot.

### 2.2 The natural groups

Looking at the function list:

```
ENTRY              convertTemplateJsonToScriban
                   _populateDefaultResource
                   _bridgeToScribanValue
                   _bridgeToScribanValueWithRecord

PATH HELPERS       _cleanSeg
                   _getArrayGroupKey
                   _setNestedValue
                   _applyBridgeToArrayItem
                   _setArrayGroup

LABEL HELPERS      _extractLabelValue
                   _extractPipeContent

COLLECTION PATH    _resolveRecordTypeFromBridge
                   _computeSourceFieldOffset
                   _detectCollectionGroups
                   _emitSingleSiblingLines
                   _emitMultiSiblingLines
                   _emitCrossCollectionLines
                   _buildCollectionScriban
                   _collVarName, _recVarName
                   _buildResourceForGroup
                   _buildPrimaryGroupResourceObj
                   _emitNestedGroupBlock
                   _buildNestedCollectionScriban

TRANSFORM KINDS    _resolveCrosswalkLookup
                   _resolveDateFormatKind
                   _resolveCodeMapKind
                   _resolveCustomScriptKind

VMR / OVERRIDES    _buildVmrPathResolver
                   _resolveSourceExpressionArray
                   _processFhirPreProcessSteps
                   _resolveFhirTemplateVmr
                   _resolveNonFhirScribanExpr

VARIABLES          _collectIdsFromBridge
                   _collectVariableAssignments

NULL SAFETY        _wrapNullSafe

OUTPUT FORMATTING  _formatScriban
                   _formatScribanArray
                   _formatScribanObject
```

That's **7 cohesive groups**.

### 2.3 Proposed split

```
utils/scriban/
├── index.ts                        ← Re-exports convertTemplateJsonToScriban
├── scriban.types.ts                ← TemplateJsonInput, TemplateBridge, CollectionGroup
├── convert-template.ts             ← main entry (~80 LOC)
│
├── path-helpers.ts                 ← _cleanSeg, _getArrayGroupKey, _setNestedValue (~120 LOC)
├── label-helpers.ts                ← _extractLabelValue, _extractPipeContent (~60 LOC)
│
├── collection-pattern/
│   ├── detect-collection.ts        ← _detectCollectionGroups (~80 LOC)
│   ├── build-collection.ts         ← _buildCollectionScriban (~150 LOC)
│   ├── build-nested.ts             ← nested-collection branch (~120 LOC)
│   └── emit-siblings.ts            ← _emitSingleSibling, _emitMultiSibling, _emitCrossCollection (~150 LOC)
│
├── transforms/
│   ├── crosswalk-kind.ts           ← _resolveCrosswalkLookup, _resolveCodeMapKind (~80 LOC)
│   ├── date-format-kind.ts         ← _resolveDateFormatKind (~40 LOC)
│   ├── custom-script-kind.ts       ← _resolveCustomScriptKind, VMR resolvers (~200 LOC)
│   └── null-safe.ts                ← _wrapNullSafe (~20 LOC)
│
├── variables.ts                    ← _collectVariableAssignments + helpers (~80 LOC)
└── formatting.ts                   ← _formatScriban + array/object (~60 LOC)
```

### 2.4 Before/after on a representative call site

#### Before

```ts
// validation-preview.ts uses ONE import that hides the entire 1270-line module
import { convertTemplateJsonToScriban, TemplateJsonInput } from '../../../utils/template-to-scriban.util';
```

#### After — the public surface stays identical

```ts
// validation-preview.ts — unchanged import, same module specifier via index.ts
import { convertTemplateJsonToScriban, TemplateJsonInput } from '../../../utils/scriban';

// utils/scriban/index.ts
export { convertTemplateJsonToScriban } from './convert-template';
export type { TemplateJsonInput, TemplateBridge } from './scriban.types';
```

Consumers don't change. Internal organization becomes navigable.

#### After — the entry function

```ts
// utils/scriban/convert-template.ts (~80 LOC)
import { TemplateJsonInput } from './scriban.types';
import { collectVariableAssignments } from './variables';
import { detectCollectionGroups } from './collection-pattern/detect-collection';
import { buildCollectionScriban } from './collection-pattern/build-collection';
import { populateDefaultResource } from './default-path';
import { formatScriban } from './formatting';

export function convertTemplateJsonToScriban(input: TemplateJsonInput): string {
  if (!input.bridges?.length) return '{}';

  const rootAccessor = input.rootAccessor ?? 'input';
  const { block: variableBlock, ids: evaluatedIds } =
    collectVariableAssignments(input.bridges, rootAccessor);

  // Collection-pattern branch
  const collectionGroups = detectCollectionGroups(input.bridges, rootAccessor);
  if (collectionGroups.length > 0) {
    return buildCollectionScriban(collectionGroups, input, variableBlock, evaluatedIds);
  }

  // Default flat-resource branch
  const resource = populateDefaultResource(input, rootAccessor, evaluatedIds);
  return (variableBlock ? variableBlock + '\n' : '') + formatScriban(resource, 0);
}
```

The entry point becomes 20 lines. The conditional logic is self-documenting.

### 2.5 Bonus — pure functions become trivially unit-testable

Today, the spec file for this util sits alongside others in `services/spec/`. After splitting, each tiny module gets its own spec:

```
utils/scriban/
├── path-helpers.spec.ts          ← test _cleanSeg, _getArrayGroupKey in isolation
├── collection-pattern/
│   ├── detect-collection.spec.ts
│   └── build-collection.spec.ts  ← integration test for the whole branch
├── transforms/
│   └── date-format-kind.spec.ts
└── …
```

A typo in `_cleanSeg` no longer requires re-running all 60 cases for `convertTemplateJsonToScriban`.

---

## Refactor 3 — `studio-state.service.ts`

### 3.1 Why it must be split

1,123 LOC. **Three concerns mashed together**:

1. **State** — signals for bridges, schemas, formats, IDs.
2. **HTTP / Persistence** — `loadFromBackend`, `saveDraftToBackend`, `saveSchemas`, `renderById`, `saveScribanTemplate`, `fetchScriban*`.
3. **Domain logic** — `calculateCoverage`, `calculateCoverageDetails`, `canonicalize`, `bridgesEqual`, `incrementVersion`, `_revalidateAll`.

Today one service handles all three; the file's table of contents (from grep) shows the mixing:

```
init, addBridge, updateBridge, deleteBridge, toggleBridge      ← state
loadFromBackend, _loadSchemasWithFallback, _persistToBackend   ← persistence  
calculateCoverage, calculateCoverageDetails, canonicalize       ← domain
saveScribanTemplate, fetchScribanTemplate, fetchScribanVersions ← scriban-persistence
```

### 3.2 Proposed split — facade pattern (mirror the existing `ScribanTemplateService` pattern)

```
services/studio/
├── studio-state.service.ts         ← FACADE: keeps the same public API (~150 LOC)
│
├── studio.store.ts                 ← pure signal store (~250 LOC)
│   • signals: _projectId, _title, _bridges, _sourceNodes, _targetNodes, _version, etc.
│   • mutations: init, addBridge, updateBridge, deleteBridge, toggleBridge
│   • computed: bridgeCount, editingBridge, hasUnsavedChanges
│
├── studio-template.api.ts          ← Template HTTP (~250 LOC)
│   • loadFromBackend, saveDraftToBackend, _persistToBackend
│   • _mapBackendBridgeToBridge, _loadSchemasWithFallback, validateTemplateExists
│   • renderById, saveSchemas
│
├── studio-scriban.api.ts           ← Scriban HTTP (~150 LOC)
│   • saveScribanTemplate, fetchScribanTemplate
│   • fetchScribanVersions, fetchchScribanByVersion (typo!)
│
├── studio-persistence.service.ts   ← Save orchestration (~150 LOC)
│   • _generateTemplateContent, _buildSchemaObjects, _resolveBridgesToPersist
│   • _determineSaveStatus, _buildSavePayload
│
└── studio-domain.service.ts        ← Pure functions (~120 LOC)
    • calculateCoverage, calculateCoverageDetails, countLeafNodes
    • canonicalize, bridgesEqual, incrementVersion, versionNumber
```

### 3.3 The Facade preserves the public API

This is **the key** to a safe migration. Every consumer of `StudioStateService` keeps working without changes.

#### Before

```ts
// services/studio-state.service.ts (1123 LOC)
@Injectable({ providedIn: 'root' })
export class StudioStateService {
  private http = inject(HttpClient);
  private scribanService = inject(ScribanTemplateService);
  private bridgeNormalization = inject(BridgeNormalizationService);
  private readonly apiUrl = environment.apiUrl;

  private _projectId = signal('');
  private _bridges = signal<Bridge[]>([]);
  // … 25 more signals

  readonly projectId = this._projectId.asReadonly();
  readonly bridges = this._bridges.asReadonly();
  // … 25 more readonly exports

  init(projectId: string, title: string, …): void { /* mutate signals */ }
  addBridge(src: string, tgt: string): void { /* mutate signal */ }
  loadFromBackend(id: string): Observable<boolean> { /* HTTP + signal mutation */ }
  saveDraft(): StudioState { /* build payload, fire HTTP, return state */ }
  // … 50 more methods
}
```

#### After — facade

```ts
// services/studio/studio-state.service.ts (~150 LOC)
@Injectable({ providedIn: 'root' })
export class StudioStateService {
  constructor(
    private store: StudioStore,
    private templateApi: StudioTemplateApi,
    private scribanApi: StudioScribanApi,
    private persistence: StudioPersistenceService,
    private domain: StudioDomainService,
  ) {}

  // ── Re-export signals (zero overhead) ──────────────────────────────
  readonly projectId    = this.store.projectId;
  readonly title        = this.store.title;
  readonly bridges      = this.store.bridges;
  readonly sourceNodes  = this.store.sourceNodes;
  // …

  // ── Re-export computed ─────────────────────────────────────────────
  readonly bridgeCount      = this.store.bridgeCount;
  readonly editingBridge    = this.store.editingBridge;
  readonly hasUnsavedChanges = this.store.hasUnsavedChanges;

  readonly saveResult$ = new Subject<{ success: boolean; message: string }>();

  // ── State mutations (delegate) ─────────────────────────────────────
  init(...args: Parameters<StudioStore['init']>): void { this.store.init(...args); }
  addBridge(src: string, tgt: string): void              { this.store.addBridge(src, tgt); }
  updateBridge(id: string, c: Partial<Bridge>): void     { this.store.updateBridge(id, c); }
  deleteBridge(id: string): void                          { this.store.deleteBridge(id); }
  toggleBridge(id: string): void                          { this.store.toggleBridge(id); }
  resetAllBridges(): void                                 { this.store.resetAllBridges(); }

  // ── HTTP — delegate to API services ────────────────────────────────
  loadFromBackend(id: string): Observable<boolean> {
    return this.templateApi.load(id).pipe(
      tap(loaded => loaded && this.store.hydrate(/* … */))
    );
  }

  saveDraft(): StudioState {
    const state = this.store.snapshot();
    this.persistence.save(state)
      .subscribe(r => this.saveResult$.next(r));
    return state;
  }

  renderById(id: string, data: any, v?: string) {
    return this.templateApi.renderById(id, data, v);
  }

  saveScribanTemplate(c: string, by?: string) {
    return this.scribanApi.save(this.store.projectId(), c, this.store.version(), by);
  }

  // ── Convenience getters ────────────────────────────────────────────
  get isPersistedToBackend(): boolean { return this.store.isPersistedToBackend; }
}
```

```ts
// services/studio/studio.store.ts (~250 LOC) — PURE STATE
@Injectable({ providedIn: 'root' })
export class StudioStore {
  private _projectId = signal('');
  private _bridges = signal<Bridge[]>([]);
  // … all signals

  readonly projectId = this._projectId.asReadonly();
  readonly bridges = this._bridges.asReadonly();
  // … all readonlys

  init(projectId: string, title: string, src: SchemaNode[], tgt: SchemaNode[], …): void {
    this._projectId.set(projectId);
    this._title.set(title);
    this._sourceNodes.set(src);
    this._targetNodes.set(tgt);
    this._bridges.set([]);
  }

  addBridge(sourcePath: string, targetPath: string): void {
    const bridge: Bridge = { id: `bridge_${Date.now()}`, sourcePath, targetPath, enabled: true };
    this._bridges.update(arr => [...arr, bridge]);
  }
  // … other pure mutations, NO HTTP

  /** Snapshot the entire state — used by the persistence service. */
  snapshot(): StudioState {
    return {
      projectId: this._projectId(),
      title: this._title(),
      bridges: this._bridges(),
      // …
    };
  }

  /** Hydrate from backend response — used by the template API. */
  hydrate(state: Partial<StudioState>): void { /* update many signals at once */ }
}
```

```ts
// services/studio/studio-template.api.ts (~250 LOC) — PURE HTTP
@Injectable({ providedIn: 'root' })
export class StudioTemplateApi {
  private http = inject(HttpClient);
  private readonly apiUrl = environment.apiUrl;

  load(projectId: string): Observable<BackendTemplateResponse | null> {
    return this.http.get<BackendTemplateResponse>(`${this.apiUrl}/template/${projectId}`).pipe(
      catchError(() => of(null))
    );
  }

  save(payload: Record<string, any>): Observable<any> {
    return this.http.put(`${this.apiUrl}/template/${payload['id']}`, payload);
  }

  renderById(id: string, data: Record<string, any>, version?: string): Observable<RenderResponse> {
    return this.http.post<RenderResponse>(`${this.apiUrl}/template/render-by-id`,
      { templateId: id, version, values: data });
  }

  loadSchemas(id: string, version: string): Observable<TemplateSchemasResponse | null> {
    return this.http.get<TemplateSchemasResponse>(`${this.apiUrl}/template/${id}/schemas`).pipe(
      catchError(() => of(null))
    );
  }

  validateExists(id: string): Observable<boolean> {
    return this.http.head(`${this.apiUrl}/template/${id}`).pipe(
      map(() => true),
      catchError(() => of(false))
    );
  }
}
```

### 3.4 Migration plan (safe, incremental)

1. **Step 1 — extract `StudioDomainService`** (1 hr, lowest risk)
   - Move pure functions (no signal access): `canonicalize`, `bridgesEqual`, `incrementVersion`, `versionNumber`, `countLeafNodes`, `calculateCoverage`, `calculateCoverageDetails`.
   - All callers stay inside `StudioStateService`; just inject the new service.
   - No public-API change.

2. **Step 2 — extract `StudioScribanApi`** (2 hrs)
   - Move `saveScribanTemplate`, `fetchScribanTemplate`, `fetchScribanVersions`, `fetchchScribanByVersion`.
   - These are clean HTTP methods with their own DTOs; trivial move.

3. **Step 3 — extract `StudioTemplateApi`** (half-day)
   - Move all `/api/template/*` HTTP calls.
   - Keep `_mapBackendBridgeToBridge` etc. **as private** to this service.

4. **Step 4 — extract `StudioPersistenceService`** (half-day)
   - Move `_generateTemplateContent`, `_buildSchemaObjects`, `_resolveBridgesToPersist`, `_determineSaveStatus`, `_buildSavePayload`, `_persistToBackend`, `saveDraftToBackend`.
   - These coordinate the store + the two API services.

5. **Step 5 — extract `StudioStore`** (half-day)
   - Move all signals and pure mutations.
   - Keep `StudioStateService` as the facade re-exporting everything.
   - **This is the largest single move** but all the dependencies are in place by now.

6. **Step 6 — audit existing specs**
   - Spec file is huge; split it to mirror the new services.

### 3.5 Result

| Metric | Before | After |
|---|---|---|
| Largest file | 1,123 LOC | ~250 LOC |
| Public API of `StudioStateService` | 50 methods | 50 methods (unchanged) |
| Files | 1 | 6 |
| Consumers needing changes | — | **0** (facade preserves API) |

---

## Refactor 4 — `schema-to-nodes.util.ts`

### 4.1 Why it must be split

843 LOC because **three completely separate parsers live in one file**:

- **JSON sample parsing** (`jsonToNodes`, `inferType`, `getObjectPreview`, `formatPrimitiveValue`)
- **XML parsing** (`xmlToNodes`)
- **FHIR StructureDefinition parsing** (the dominant ~500 LOC: `isFhirStructureDefinition`, `fhirStructureDefinitionToNodes`, `expandComplexTypes`, `ensureParentExists`, `resolveTypeCode`, etc., plus the huge `FHIR_DATA_TYPES` lookup)

Adding a new format (e.g. AVRO, Protobuf) means adding more cases to the dispatcher and another ~200 LOC of parsing right next to the FHIR mess.

### 4.2 Proposed split

```
utils/schema-parsers/
├── index.ts                     ← export schemaToNodes
├── schema-to-nodes.ts           ← dispatcher only (~40 LOC)
├── id-generator.ts              ← nextSchemaNodeId, resetSchemaNodeIdCounter (~20 LOC)
│
├── json/
│   ├── json-to-nodes.ts         ← jsonToNodes (~80 LOC)
│   ├── infer-type.ts            ← inferType (~15 LOC)
│   └── preview.ts               ← getObjectPreview, formatPrimitiveValue (~40 LOC)
│
├── xml/
│   └── xml-to-nodes.ts          ← xmlToNodes (~25 LOC)
│
└── fhir/
    ├── fhir-structure-def.ts    ← is*StructureDefinition, fhirStructureDefinitionToNodes (~150 LOC)
    ├── fhir-types.ts            ← FHIR_DATA_TYPES constant (~250 LOC)
    ├── expand-complex.ts        ← expandComplexTypes (~120 LOC)
    ├── type-resolution.ts       ← resolveTypeCode, isPrimitiveType, isKnownComplexType, toFieldType (~50 LOC)
    └── tree-helpers.ts          ← ensureParentExists, cleanEmptyChildren (~80 LOC)
```

### 4.3 The dispatcher becomes a 20-line file

#### Before — buried in 843 LOC

```ts
export function schemaToNodes(schema: UploadedSchema): SchemaNode[] {
  resetSchemaNodeIdCounter();
  try {
    if ((schema as any).nodes) return (schema as any).nodes;
    if (schema.summary?.detectedFormat === 'json') {
      const parsed = JSON.parse(schema.rawText);
      if (isFhirStructureDefinition(parsed)) {
        return fhirStructureDefinitionToNodes(parsed);
      }
      return jsonToNodes(parsed, '');
    }
    const doc = new DOMParser().parseFromString(schema.rawText, 'application/xml');
    return xmlToNodes(doc.documentElement, '');
  } catch (e) {
    console.error('schemaToNodes error:', e);
    return [];
  }
}
```

#### After — self-contained dispatcher

```ts
// utils/schema-parsers/schema-to-nodes.ts
import { UploadedSchema } from '../../models/template.models';
import { SchemaNode } from '../../models/studio.models';
import { resetSchemaNodeIdCounter } from './id-generator';
import { jsonToNodes } from './json/json-to-nodes';
import { xmlToNodes } from './xml/xml-to-nodes';
import { isFhirStructureDefinition, fhirStructureDefinitionToNodes } from './fhir/fhir-structure-def';

export function schemaToNodes(schema: UploadedSchema): SchemaNode[] {
  resetSchemaNodeIdCounter();
  if ((schema as any).nodes) return (schema as any).nodes;

  try {
    if (schema.summary?.detectedFormat === 'json') {
      const parsed = JSON.parse(schema.rawText);
      return isFhirStructureDefinition(parsed)
        ? fhirStructureDefinitionToNodes(parsed)
        : jsonToNodes(parsed, '');
    }
    const doc = new DOMParser().parseFromString(schema.rawText, 'application/xml');
    return xmlToNodes(doc.documentElement, '');
  } catch (e) {
    console.error('schemaToNodes error:', e);
    return [];
  }
}
```

### 4.4 Bonus — `FHIR_DATA_TYPES` becomes its own file

The current file has a **250-line `const FHIR_DATA_TYPES`** at the top defining `Meta`, `Identifier`, `HumanName`, `Address`, `ContactPoint`, `CodeableConcept`, `Coding`, `Reference`, `Quantity`, `Period`, `Range`, `Ratio`, `Timing`, `Annotation`, `Attachment`, `Extension`, `Signature`, `Money`, `Narrative`, `BackboneElement`, `Element`, `Dosage`. Each definition is itself ~10 lines. Extracting this to `fhir/fhir-types.ts` is the single biggest readability win.

```ts
// utils/schema-parsers/fhir/fhir-types.ts (data only, no logic)
export interface FhirTypeField {
  name: string;
  type: string;
  isArray?: boolean;
}

export const FHIR_DATA_TYPES: Record<string, FhirTypeField[]> = {
  Meta: [
    { name: 'versionId', type: 'id' },
    { name: 'lastUpdated', type: 'instant' },
    { name: 'source', type: 'uri' },
    { name: 'profile', type: 'canonical', isArray: true },
    { name: 'security', type: 'Coding', isArray: true },
    { name: 'tag', type: 'Coding', isArray: true }
  ],
  Identifier: [ /* … */ ],
  HumanName:  [ /* … */ ],
  // …
};
```

---

## Refactor 5 — `scriban-template-emitter.service.ts`

### 5.1 Why split

829 LOC service responsible for **the entire emission stage** of the Scriban pipeline. It already collaborates with three other services (`ScribanPathParser`, `ScribanExpressionBuilder`, `ScribanBridgeEnrichment`), so it's already partially decomposed — but the orchestration is itself massive.

The service handles:
- **Single-collection** emission (single record-type filter)
- **Multi-collection** emission (multiple record types, sibling output)
- **Multi-record-type** emission (cross-collection scenarios)
- **Target-tree building** (intermediate `TargetTreeNode` structure)
- **Output serialization** (`TargetTreeNode` → Scriban string)

### 5.2 Proposed split

```
services/scriban/emitter/
├── scriban-template-emitter.service.ts  ← Facade entry (~120 LOC)
├── target-tree.ts                       ← TargetTreeNode types + builder (~150 LOC)
│
├── strategies/
│   ├── emission-strategy.ts             ← interface EmissionStrategy
│   ├── single-collection.strategy.ts    ← (~200 LOC)
│   ├── multi-collection.strategy.ts     ← (~200 LOC)
│   └── multi-record-type.strategy.ts    ← (~200 LOC)
│
└── serialize-tree.ts                    ← Target tree → string (~120 LOC)
```

### 5.3 The Strategy pattern in practice

#### Before — internal if/else

```ts
// scriban-template-emitter.service.ts (abridged)
generateScribanTemplate(bridges, targetFormat, sourceNodes?, targetNodes?, templateId?): string {
  // 50 lines of analysis
  if (this._hasMultipleRecordTypes(bridges)) {
    return this._emitMultiRecordType(bridges, …);   // 200 lines below
  }
  if (this._hasMultipleCollections(bridges)) {
    return this._emitMultiCollection(bridges, …);    // 200 lines below
  }
  return this._emitSingleCollection(bridges, …);      // 200 lines below
}
```

#### After — strategy selection

```ts
// services/scriban/emitter/scriban-template-emitter.service.ts (~120 LOC)
@Injectable({ providedIn: 'root' })
export class ScribanTemplateEmitterService {
  constructor(
    private singleCollection: SingleCollectionStrategy,
    private multiCollection:  MultiCollectionStrategy,
    private multiRecordType:  MultiRecordTypeStrategy,
  ) {}

  generateScribanTemplate(
    bridges: Bridge[],
    targetFormat: string,
    sourceNodes?: SchemaNode[],
    targetNodes?: SchemaNode[],
    templateId?: string,
  ): string {
    const strategy = this._pickStrategy(bridges);
    return strategy.emit({ bridges, targetFormat, sourceNodes, targetNodes, templateId });
  }

  private _pickStrategy(bridges: Bridge[]): EmissionStrategy {
    if (this._hasMultipleRecordTypes(bridges)) return this.multiRecordType;
    if (this._hasMultipleCollections(bridges)) return this.multiCollection;
    return this.singleCollection;
  }

  private _hasMultipleRecordTypes(bridges: Bridge[]): boolean { /* … */ }
  private _hasMultipleCollections(bridges: Bridge[]): boolean { /* … */ }
}
```

```ts
// services/scriban/emitter/strategies/emission-strategy.ts
export interface EmissionContext {
  bridges: Bridge[];
  targetFormat: string;
  sourceNodes?: SchemaNode[];
  targetNodes?: SchemaNode[];
  templateId?: string;
}

export interface EmissionStrategy {
  emit(ctx: EmissionContext): string;
}
```

```ts
// services/scriban/emitter/strategies/single-collection.strategy.ts
@Injectable({ providedIn: 'root' })
export class SingleCollectionStrategy implements EmissionStrategy {
  constructor(
    private pathParser: ScribanPathParserService,
    private exprBuilder: ScribanExpressionBuilderService,
    private enrichment: ScribanBridgeEnrichmentService,
    private targetTree: TargetTreeBuilder,
    private serializer: ScribanTreeSerializer,
  ) {}

  emit(ctx: EmissionContext): string {
    const enriched = this.enrichment.enrich(ctx.bridges);
    const tree     = this.targetTree.build(enriched);
    return this.serializer.serialize(tree);
  }
}
```

### 5.4 The Benefit

Adding a fourth strategy (e.g. "flat-array emission") becomes:

1. Create `strategies/flat-array.strategy.ts` (~150 LOC).
2. Register it in the constructor of `ScribanTemplateEmitterService`.
3. Add a branch to `_pickStrategy`.

Nothing else changes. Today, this addition would require modifying the 829-line file.

---

## Refactor 6 — `validation-preview.ts`

### 6.1 Why split

819 LOC component doing **four loosely-related jobs**:

1. **Template JSON building** — `templateJson` getter + `_templateJson_*` helpers (~100 LOC)
2. **Scriban template generation** — wraps the util, plus edit mode (~100 LOC)
3. **Custom JSON stringify** (with Scriban expression preservation) — `_customJsonStringify`, `_stringifyArray`, `_stringifyEntryArray`, `_stringifyObject`, `_buildObjectString` (~200 LOC)
4. **Syntax highlighting** — `_syntaxHighlight`, `_highlightValue`, `_highlightScribanExpr`, `_esc` (~80 LOC)
5. **Saved versions** — `loadSavedVersions`, `loadVersion`, `clearLoadedVersion`, `formatVersionDate` (~100 LOC)
6. **Viva mode** — `switchViewMode`, `convertVivaInput`, `vivaHighlightedOutput`, `copyVivaOutput` (~60 LOC)

### 6.2 Proposed split — extract presentation helpers as services + small subcomponents

```
mapping-studio/validation-preview/
├── validation-preview.ts                  ← UI orchestration (~200 LOC)
├── validation-preview.html
├── validation-preview.css
│
├── template-json-builder.service.ts       ← buildTemplateJson (~120 LOC)
├── scriban-json-stringifier.service.ts    ← _customJsonStringify (~200 LOC)
├── scriban-syntax-highlighter.service.ts  ← _syntaxHighlight, _highlightValue (~100 LOC)
│
└── parts/
    ├── version-panel.component.ts         ← list/load/clear saved Scriban versions (~120 LOC)
    └── viva-panel.component.ts            ← Viva mode (~80 LOC)
```

### 6.3 Before — everything inside the component

```ts
export class ValidationPreviewComponent {
  // ── Template JSON ─────────────────────────────────────
  private _templateJson_normalizePath(path: string): string { /* … */ }
  private _templateJson_normalizeNode(node: any): any        { /* … */ }
  private _templateJson_buildCleanParams(params: ...): ...    { /* … */ }
  private _templateJson_buildBridgeEntry(bridge: Bridge): ... { /* … */ }
  get templateJson(): object { /* … */ }
  
  // ── JSON stringify with Scriban preservation ──────────
  private _customJsonStringify(value: any, indent: number): string { /* … */ }
  private _stringifyPrimitive(value: any): string | undefined       { /* … */ }
  private _stringifyArray(value: any[], indent: number, ...): ...    { /* … */ }
  // 4 more helpers

  // ── Highlighting ──────────────────────────────────────
  private _syntaxHighlight(json: string): string         { /* … */ }
  private _highlightValue(raw: string): string           { /* … */ }
  private _highlightScribanExpr(expr: string): string    { /* … */ }
  private _esc(s: string): string                         { /* … */ }

  // ── Saved versions ────────────────────────────────────
  savedVersions: ScribanVersionSummary[] = [];
  selectedVersion: string | null = null;
  loadSavedVersions(): void  { /* HTTP + state */ }
  loadVersion(v: string): void { /* HTTP + state */ }
  // …

  // ── Viva mode ─────────────────────────────────────────
  vivaJsonInput = '';
  vivaOutput = '';
  switchViewMode(mode: 'scriban' | 'viva'): void { /* … */ }
  convertVivaInput(): void                        { /* … */ }
}
```

### 6.4 After — focused services and child components

```ts
// template-json-builder.service.ts (~120 LOC) — pure service, no UI
@Injectable({ providedIn: 'root' })
export class TemplateJsonBuilderService {
  build(input: BuildInput): object {
    const bridges = input.bridges
      .filter(b => b.enabled)
      .map(b => this._buildBridgeEntry(b));
    return {
      title: input.title || 'Mapping Template',
      ...(input.templateId ? { templateId: input.templateId } : {}),
      sourceSystem: input.sourceFormat || 'Source',
      targetSystem: input.targetFormat || 'Target',
      tags: this._deriveTags(input),
      coverage: this._validPercent(bridges),
      status: this._projectStatus(bridges),
      bridges,
      sourceSchema: { nodes: input.sourceNodes.map(n => this._normalizeNode(n)) },
      targetSchema: { nodes: input.targetNodes.map(n => this._normalizeNode(n)) }
    };
  }

  private _buildBridgeEntry(bridge: Bridge): Record<string, unknown> { /* … */ }
  private _normalizeNode(node: any): any { /* … */ }
  private _validPercent(bridges: Bridge[]): number { /* … */ }
  private _projectStatus(bridges: Bridge[]): string { /* … */ }
}
```

```ts
// scriban-syntax-highlighter.service.ts (~100 LOC) — pure service
@Injectable({ providedIn: 'root' })
export class ScribanSyntaxHighlighterService {
  highlight(json: string): string {
    return this._lines(json).map(line => this._highlightLine(line)).join('\n');
  }
  private _highlightLine(line: string): string { /* … */ }
  private _highlightValue(raw: string): string { /* … */ }
  private _highlightScribanExpr(expr: string): string { /* … */ }
  private _esc(s: string): string { /* … */ }
}
```

```ts
// parts/version-panel.component.ts (~120 LOC)
@Component({
  selector: 'app-version-panel',
  standalone: true,
  templateUrl: './version-panel.component.html',
})
export class VersionPanelComponent implements OnInit {
  @Input({ required: true }) templateId!: string;
  @Output() versionLoaded = new EventEmitter<ScribanTemplateDocument>();

  versions: ScribanVersionSummary[] = [];
  selectedVersion: string | null = null;
  isLoadingVersions = false;
  versionError = '';

  constructor(private scribanService: ScribanTemplateService) {}

  ngOnInit(): void {
    this.loadVersions();
  }

  loadVersions(): void { /* … */ }
  selectVersion(v: string): void { /* … */ }
  clearSelection(): void { /* … */ }
  formatDate(d: string): string { /* … */ }
}
```

```ts
// validation-preview.ts (NEW — ~200 LOC, UI orchestration)
@Component({
  selector: 'app-validation-preview',
  standalone: true,
  imports: [CommonModule, FormsModule, VersionPanelComponent, VivaPanelComponent],
  templateUrl: './validation-preview.html',
})
export class ValidationPreviewComponent implements OnChanges, OnDestroy {
  @Input({ required: true }) bridges: Bridge[] = [];
  @Input() sourceFormat = '';
  @Input() targetFormat = '';
  @Input() title = '';
  @Input() templateId = '';
  @Input() sourceNodes: SchemaNode[] = [];
  @Input() targetNodes: SchemaNode[] = [];
  @Output() close = new EventEmitter<void>();

  constructor(
    private builder: TemplateJsonBuilderService,
    private stringifier: ScribanJsonStringifierService,
    private highlighter: ScribanSyntaxHighlighterService,
    private sanitizer: DomSanitizer,
  ) {}

  get templateJson() {
    return this.builder.build({
      bridges: this.bridges, title: this.title, templateId: this.templateId,
      sourceFormat: this.sourceFormat, targetFormat: this.targetFormat,
      sourceNodes: this.sourceNodes, targetNodes: this.targetNodes,
    });
  }

  get templateJsonString(): string {
    return JSON.stringify(this.templateJson, null, 2);
  }

  get templateJsonHighlighted(): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(
      this.highlighter.highlight(this.templateJsonString)
    );
  }
  
  onVersionLoaded(doc: ScribanTemplateDocument): void {
    this.loadedSavedTemplate = doc.scribanContent;
  }
}
```

```html
<!-- validation-preview.html (now declarative) -->
<div class="preview">
  <header>
    <button (click)="viewMode = 'scriban'">Scriban</button>
    <button (click)="viewMode = 'viva'">Viva</button>
  </header>

  @if (viewMode === 'scriban') {
    <pre [innerHTML]="templateJsonHighlighted"></pre>
    <app-version-panel
      [templateId]="templateId"
      (versionLoaded)="onVersionLoaded($event)" />
  } @else {
    <app-viva-panel [defaultJson]="templateJsonString" />
  }
</div>
```

---

## Refactor 7 — `mapping-studio.ts`

### 7.1 Why split

431 LOC is borderline. **The biggest problem is the 4-branch initialization** in `ngOnInit` which routes by checking edit mode, then wizard state, then direct init, then redirect. The "overlay params" logic in `onConnectionCommit` is also non-trivial.

### 7.2 Minimal proposed split

Don't break the component apart yet — extract the init logic into a service:

```
mapping-studio/
├── mapping-studio.ts                       (~280 LOC after extraction)
├── mapping-studio.html
├── mapping-studio.css
├── mapping-studio-init.service.ts          ← NEW (~150 LOC)
├── bridge-commit-builder.ts                ← NEW (~50 LOC)
└── … existing sub-components
```

### 7.3 Before — init logic in the component

```ts
ngOnInit(): void {
  this.loadCrosswalks();
  const projectId = this.route.snapshot.paramMap.get('id')
                 ?? this.route.snapshot.queryParamMap.get('id') ?? '';

  if (this._handleEditMode(projectId))                return;
  const state = this.newProjectState.state();
  if (this._handleNewTemplateFromWizard(projectId, state)) return;
  if (this._handleDirectInitialization(projectId))    return;
  this._redirectToSchemaSetup();
}

private _handleEditMode(projectId: string): boolean { /* 15 LOC */ }
private _handleLoadSuccess(loaded: boolean): void   { /* … */ }
private _handleLoadError(err: any): void            { /* … */ }
private _handleNewTemplateFromWizard(projectId: string, state: any): boolean { /* 30 LOC */ }
private _handleDirectInitialization(projectId: string): boolean { /* … */ }
private _redirectToSchemaSetup(): void              { /* … */ }
private hasInMemoryTemplateData(state: any): boolean { /* … */ }
```

### 7.4 After — init service

```ts
// mapping-studio-init.service.ts (NEW)
import { Injectable, inject } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { Observable, of } from 'rxjs';
import { map, catchError } from 'rxjs/operators';

export type InitOutcome = 'edit-loaded' | 'edit-failed' | 'wizard' | 'direct' | 'redirect';

@Injectable()  // scoped to component (not 'root')
export class MappingStudioInitService {
  private route   = inject(ActivatedRoute);
  private router  = inject(Router);
  private studio  = inject(StudioStateService);
  private wizard  = inject(NewTemplateStateService);

  /** Returns the chosen initialization branch and performs it. */
  initialize(): Observable<InitOutcome> {
    const projectId = this.route.snapshot.paramMap.get('id')
                   ?? this.route.snapshot.queryParamMap.get('id') ?? '';
    const isEditMode = this.route.snapshot.queryParamMap.get('edit') === 'true';

    if (isEditMode && projectId) return this._initEditMode(projectId);

    const wizardState = this.wizard.state();
    if (this._hasInMemoryTemplateData(wizardState)) {
      this._initFromWizard(projectId, wizardState);
      return of('wizard');
    }

    if (projectId) {
      this.studio.init(projectId, 'New Template', [], [], '', '', '', '', '', '');
      return of('direct');
    }

    this.router.navigate(['/templates/new/schema-setup']);
    return of('redirect');
  }

  private _initEditMode(id: string): Observable<InitOutcome> {
    this.wizard.reset();
    return this.studio.loadFromBackend(id).pipe(
      map(loaded => {
        if (loaded) return 'edit-loaded' as const;
        this.router.navigate(['/templates/new/schema-setup']);
        return 'edit-failed' as const;
      }),
      catchError(() => {
        this.router.navigate(['/templates/new/schema-setup']);
        return of('edit-failed' as const);
      })
    );
  }

  private _initFromWizard(id: string, state: any): void {
    const srcNodes = state.source ? schemaToNodes(state.source) : [];
    const tgtNodes = state.target ? schemaToNodes(state.target) : [];
    let tgtFmt = (state.target?.summary?.detectedFormat ?? '').toUpperCase();

    if (state.target?.summary?.detectedFormat === 'json' && state.target?.rawText) {
      try {
        const parsed = JSON.parse(state.target.rawText);
        if (typeof parsed?.resourceType === 'string' && parsed.resourceType) {
          tgtFmt = parsed.resourceType;
        }
      } catch { /* keep fallback */ }
    }

    this.studio.init(
      id,
      state.title || 'Mapping Template',
      srcNodes, tgtNodes,
      (state.source?.summary?.detectedFormat ?? '').toUpperCase(),
      tgtFmt,
      state.source?.rawText ?? '',
      state.target?.rawText ?? '',
      state.sourceSystem || '',
      state.targetSystem || '',
    );
  }

  private _hasInMemoryTemplateData(s: any): boolean {
    return !!(s.source || s.target || (s.title && s.title.trim().length > 0));
  }
}
```

```ts
// bridge-commit-builder.ts (NEW — pure function, no Angular)
import { Bridge, TransformKind } from '../../models/studio.models';

export interface CommitInput { /* … */ }
export interface ExistingBridge { params: Record<string, string> | undefined }

/**
 * Pure builder for bridge updates. Preserves derived params (recordType,
 * sourceArrayPath, collectionTypeField) that the dialog doesn't manage.
 */
export function buildBridgeUpdate(event: CommitInput, existing?: ExistingBridge): {
  sourcePath: string;
  targetPath: string;
  transform: Bridge['transform'];
} {
  const ruleSetIds = event.ruleSetId
    ? event.ruleSetId.split(',').map(id => id.trim()).filter(Boolean)
    : [];
  const newParams: Record<string, string> = { ...(existing?.params ?? {}) };

  if (event.ruleSetId !== undefined)     newParams['ruleSetId']     = event.ruleSetId     || '';
  if (event.crosswalkName !== undefined) newParams['crosswalkName'] = event.crosswalkName || '';
  if (event.defaultValue !== undefined)  newParams['defaultValue']  = event.defaultValue  || '';
  if (event.valueMapRules !== undefined) newParams['valueMapRules'] = event.valueMapRules || '';
  if (event.sourceArrayPath !== undefined) newParams['sourceArrayPath'] = event.sourceArrayPath || '';

  const transform: Bridge['transform'] = event.transform !== 'none'
    ? { kind: event.transform, ruleSetIds, params: newParams }
    : { kind: 'none' };

  return { sourcePath: event.sourcePath, targetPath: event.targetPath, transform };
}
```

```ts
// mapping-studio.ts (NEW — ~280 LOC)
@Component({
  selector: 'app-mapping-studio',
  standalone: true,
  providers: [MappingStudioInitService],   // ← scoped service
  imports: [/* … */],
  templateUrl: './mapping-studio.html',
})
export class MappingStudioComponent implements OnInit, OnDestroy {
  constructor(
    private initSvc: MappingStudioInitService,
    readonly studio: StudioStateService,
    private dialog: MatDialog,
    private router: Router,
    private crosswalkService: CrosswalkService,
  ) {}

  ngOnInit(): void {
    this.loadCrosswalks();
    this.initSvc.initialize()
      .pipe(takeUntilDestroyed())
      .subscribe(outcome => {
        this._isEditMode.set(outcome === 'edit-loaded');
        if (outcome === 'edit-loaded') this._showToast('Template loaded');
        if (outcome === 'edit-failed') this._showToast('Failed to load template');
      });
  }

  onConnectionCommit(event: CommitInput): void {
    const existing = this.editingBridgeId
      ? this.studio.bridges().find(b => b.id === this.editingBridgeId)
      : undefined;

    const update = buildBridgeUpdate(event, existing?.transform
      ? { params: existing.transform.params }
      : undefined);

    if (this.editingBridgeId) {
      this.studio.updateBridge(this.editingBridgeId, update);
      this._showToast('Connection updated');
    } else {
      this.studio.addBridge(update.sourcePath, update.targetPath);
      const last = this.studio.bridges().at(-1);
      if (last) this.studio.updateBridge(last.id, { transform: update.transform });
      this._showToast('Connection created');
    }
    this._resetDialog();
  }

  // … the rest of the component stays similar but is now much smaller
}
```

The component shrinks by ~150 LOC and the unit-testable bits (init logic, commit builder) become **testable without Angular's TestBed**.

---

## Refactor 8 — `template-builder.component.ts`

### 8.1 Why enhance

274 LOC is moderate, but the file has **the worst single anti-pattern in the codebase** (direct DOM manipulation of a textarea), AND it duplicates concepts from the Mapping Studio (crosswalk picker, rule-set picker).

```ts
@ViewChild('templateEditor') templateEditor!: ElementRef;

insertSnippet(code: string): void {
  const textarea = this.templateEditor?.nativeElement;
  if (!textarea) return;
  const start = textarea.selectionStart;
  const end = textarea.selectionEnd;
  const text = textarea.value;
  const newText = text.substring(0, start) + code + text.substring(end);
  this.templateForm.patchValue({ templateContent: newText });
  setTimeout(() => {
    textarea.focus();
    textarea.setSelectionRange(start + code.length, start + code.length);
  }, 0);
}
```

### 8.2 Proposed enhancements

**Option A — Minimum-viable refactor**: extract a `<app-scriban-editor>` directive/component that owns the textarea + snippet insertion via `Renderer2`.

```ts
// shared/scriban-editor/scriban-editor.directive.ts
import { Directive, ElementRef, HostListener, Renderer2, inject } from '@angular/core';

@Directive({
  selector: 'textarea[appScribanEditor]',
  standalone: true,
})
export class ScribanEditorDirective {
  private el = inject(ElementRef<HTMLTextAreaElement>);
  private renderer = inject(Renderer2);

  insertAtCursor(snippet: string): void {
    const ta = this.el.nativeElement;
    const start = ta.selectionStart;
    const end   = ta.selectionEnd;
    const text  = ta.value;
    const next  = text.substring(0, start) + snippet + text.substring(end);

    // Use Renderer2 so SSR/zoneless are safe; trigger input event so [(ngModel)] sees the change
    this.renderer.setProperty(ta, 'value', next);
    ta.dispatchEvent(new Event('input', { bubbles: true }));
    queueMicrotask(() => {
      ta.focus();
      ta.setSelectionRange(start + snippet.length, start + snippet.length);
    });
  }
}
```

Then in `template-builder.component.ts`:

```ts
@ViewChild(ScribanEditorDirective) editor!: ScribanEditorDirective;

insertSnippet(code: string): void {
  this.editor.insertAtCursor(code);
}
```

Template:

```html
<textarea
  appScribanEditor
  [formControl]="templateForm.controls.templateContent"
  rows="20"></textarea>
```

The directive is **reusable** — the Mapping Studio's edit-mode template view can use it too.

**Option B — Full editor swap**: replace the `<textarea>` with **Monaco** or **CodeMirror**. Both give:
- Syntax highlighting natively (no `bypassSecurityTrustHtml` workarounds).
- Line numbers, code folding, find/replace.
- Multi-cursor, undo/redo with history.
- Bracket matching for Scriban `{{ … }}`.

Estimated effort: 1 day for a working Monaco integration, 2-3 days for full polish (Scriban tokenizer registration, custom autocomplete for crosswalks/rulesets).

### 8.3 Also — wire up `saveTemplate()`

Currently:

```ts
saveTemplate(): void {
  if (this.templateForm.invalid) return;
  const template = { ...this.templateForm.value, crosswalks: this.selectedCrosswalks, ruleSets: this.selectedRuleSets };
  console.log('Save template:', template);                          // ← logs only
  this.snackBar.open('✅ Template saved', 'Close', { duration: 2000 });   // ← lies
}
```

Either remove the feature (the snackbar lies — nothing is saved) or wire it to the real `ScribanTemplateService.saveScribanTemplate(...)`.

---

## Cross-Cutting Enhancements

These don't fit any single file but apply across the codebase.

### CC1 — Adopt `OnPush` everywhere

Today **zero components use `OnPush`**. Combined with signals already in use, the migration is mostly mechanical:

```ts
// Before
@Component({ selector: 'app-x', templateUrl: '…' })

// After
@Component({
  selector: 'app-x',
  templateUrl: '…',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
```

The exceptions where you'll need to change patterns:

- Components using **getters** in templates that compute on every change cycle (e.g. `ExportBundleComponent.crosswalkCount`). Convert these to `computed()` or signals.
- Components with `setTimeout(... markForCheck …)` patterns (some of the dialogs). With OnPush + signals these become unnecessary; signal updates schedule CD automatically.

### CC2 — Subscription discipline

Today: 54 `.subscribe()` calls, only 6 `takeUntilDestroyed`/`ngOnDestroy`. Many are HTTP observables (auto-complete, safe) but ones to fix:

| File | Where | Risk |
|---|---|---|
| `app.ts` | `constructor` — `router.events` | Low (long-lived component) |
| `new-template.ts` | `ngOnInit` — `route.queryParams` | Medium |
| `fhir-render-preview.ts` | `ngOnInit` — `_sourceChange$` debounce | **High** — leaks on modal close |

Adopt one pattern across the codebase:

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
// …
this._sourceChange$
  .pipe(debounceTime(800), distinctUntilChanged(), takeUntilDestroyed())
  .subscribe(() => this.render());
```

### CC3 — Replace `any` with discriminated unions

197 `: any` annotations, concentrated in backend-mapping layers. Pattern to follow — define raw types per endpoint, then a `to*` mapper:

```ts
// models/backend/template-raw.ts
export interface BackendTemplateRaw {
  id?: string;
  Id?: string;                          // PascalCase from .NET
  title?: string;
  Title?: string;
  bridges?: BackendBridgeRaw[];
  Bridges?: BackendBridgeRaw[];
  templateMetadata?: BackendMetadataRaw;
  templatemetadata?: BackendMetadataRaw;
  // …
}

// services/backend/template-mapper.ts
export function toMappingProject(raw: BackendTemplateRaw): MappingProject {
  const id = raw.id ?? raw.Id ?? '';
  const title = raw.title ?? raw.Title ?? 'Untitled';
  const meta = raw.templateMetadata ?? raw.templatemetadata ?? {};
  return {
    id,
    title,
    sourceSystem: meta.sourceSystem ?? meta.sourcesystem ?? 'Unknown',
    // …
  };
}
```

Now `_createMappingProject(t: any)` in `DashboardService` becomes `toMappingProject(t: BackendTemplateRaw)` — type-safe, IDE-completing.

### CC4 — Reusable `FileDownloadService`

The "blob → URL → anchor.click → revokeObjectURL" pattern is duplicated **4 times**:

| File | Lines |
|---|---|
| `dashboard.component.ts` | 274-279 (exportTemplate) |
| `export-bundle.service.ts` | 46-55 (triggerDownload) |
| `template-builder.component.ts` | 258-263 (exportTemplate) |
| `validation-preview.ts` | 680-686 (exportTemplate) |

Extract:

```ts
// services/file-download.service.ts (~30 LOC)
@Injectable({ providedIn: 'root' })
export class FileDownloadService {
  /**
   * Trigger a browser download for a Blob with the given filename.
   * Handles URL.createObjectURL lifecycle and clean-up.
   */
  download(blob: Blob, filename: string): void {
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement('a');
    anchor.href = url;
    anchor.download = filename;
    anchor.style.display = 'none';
    document.body.appendChild(anchor);
    anchor.click();
    document.body.removeChild(anchor);
    URL.revokeObjectURL(url);
  }

  downloadText(text: string, filename: string, type = 'application/json'): void {
    this.download(new Blob([text], { type }), filename);
  }
}
```

Then call `this.fileDownload.downloadText(JSON.stringify(template), 'template.json')` everywhere.

### CC5 — Lazy load routes

Currently every route is statically imported in `app.routes.ts`. Switch to:

```ts
// app.routes.ts (NEW)
export const routes: Routes = [
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
  { path: 'dashboard',       loadComponent: () => import('./features/dashboard/dashboard.component').then(m => m.DashboardComponent) },
  { path: 'system',          loadComponent: () => import('./features/system/system').then(m => m.System) },
  { path: 'crosswalk',       loadComponent: () => import('./features/crosswalk/crosswalk-tables.component').then(m => m.CrosswalkTablesComponent) },
  { path: 'template-builder', loadComponent: () => import('./features/template-builder/template-builder.component').then(m => m.TemplateBuilderComponent) },
  { path: 'export-bundle',   loadComponent: () => import('./features/export-bundle/export-bundle.component').then(m => m.ExportBundleComponent) },
  { path: 'templates/new/schema-setup', loadComponent: () => import('./features/template-wizard/new-template').then(m => m.NewTemplateComponent) },
  { path: 'templates/:id/studio',      loadComponent: () => import('./features/mapping-studio/mapping-studio').then(m => m.MappingStudioComponent) },
  { path: 'render-preview', outlet: 'modal',
    loadComponent: () => import('./features/mapping-studio/fhir-render-preview/fhir-render-preview').then(m => m.FhirRenderPreviewComponent) },
  // legacy redirects (unchanged)
  { path: 'home', redirectTo: 'dashboard', pathMatch: 'full' },
  { path: 'projects/new/schema-setup', redirectTo: 'templates/new/schema-setup', pathMatch: 'full' },
  { path: 'projects/:id/studio', redirectTo: 'templates/:id/studio', pathMatch: 'full' },
  { path: '**', redirectTo: 'dashboard' }
];
```

Expected: the initial bundle drops by a large margin because the 1,864-LOC `FieldConnectionDialog` and its 819-LOC `ValidationPreview` only load when the user actually opens the Mapping Studio.

### CC6 — Add `HttpInterceptor` infrastructure

The plumbing (`provideHttpClient(withInterceptorsFromDi())`) is already in `app.config.ts`. Add three interceptors:

```ts
// interceptors/error.interceptor.ts
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(private snack: MatSnackBar) {}
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError(err => {
        if (err.status === 0)  this.snack.open('Network error — backend unreachable', 'Close', { duration: 5000 });
        if (err.status >= 500) this.snack.open(`Server error (${err.status})`, 'Close', { duration: 5000 });
        return throwError(() => err);
      })
    );
  }
}

// interceptors/correlation-id.interceptor.ts
@Injectable()
export class CorrelationIdInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req.clone({ setHeaders: { 'X-Correlation-Id': crypto.randomUUID() } }));
  }
}

// interceptors/auth.interceptor.ts (if/when auth is added)
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private auth: AuthService) {}
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.auth.token;
    return next.handle(token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req);
  }
}
```

Register in `app.config.ts`:

```ts
providers: [
  // …
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: CorrelationIdInterceptor, multi: true },
  // { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
]
```

With this, every `console.error(...)` + manual `snackBar.open(...)` pair in services can be deleted; the interceptor handles them uniformly.

### CC7 — `CanDeactivate` guard for unsaved bridges

Today `MappingStudioComponent.goBack()` manually opens a `ConfirmDialog` if `studio.bridges().length > 0`. This only triggers on the in-app back button, not on URL changes, browser back, accidental refresh, etc.

```ts
// guards/unsaved-changes.guard.ts
export interface CanDeactivateUnsaved {
  canDeactivate(): boolean | Observable<boolean> | Promise<boolean>;
}

export const unsavedChangesGuard: CanDeactivateFn<CanDeactivateUnsaved> = (component) =>
  component.canDeactivate ? component.canDeactivate() : true;
```

```ts
// mapping-studio.ts (add interface)
export class MappingStudioComponent implements OnInit, OnDestroy, CanDeactivateUnsaved {
  canDeactivate(): Observable<boolean> {
    if (!this.studio.hasUnsavedChanges()) return of(true);
    return this.dialog.open(ConfirmDialogComponent, {
      data: {
        title: 'Leave Mapping Studio?',
        message: 'You have unsaved bridge mappings. Are you sure?',
        confirmText: 'Yes, Leave', cancelText: 'Stay Here',
        icon: 'warning', warn: true
      },
      width: '460px'
    }).afterClosed().pipe(map(confirmed => !!confirmed));
  }
}
```

```ts
// app.routes.ts
{ path: 'templates/:id/studio',
  loadComponent: () => import('./features/mapping-studio/mapping-studio').then(m => m.MappingStudioComponent),
  canDeactivate: [unsavedChangesGuard] },
```

Now the guard fires for **every** navigation away from the studio, including direct URL changes.

---

## Recommended Folder Layout (Target State)

After all refactors:

```
src/app/
├── app.config.ts
├── app.routes.ts                          ← lazy-loaded
├── app.ts / app.html / app.css
│
├── core/
│   ├── interceptors/
│   │   ├── error.interceptor.ts           ← NEW
│   │   ├── correlation-id.interceptor.ts  ← NEW
│   │   └── auth.interceptor.ts            ← NEW (optional)
│   ├── guards/
│   │   └── unsaved-changes.guard.ts       ← NEW
│   └── layout/
│       └── sidebar/                       (header/ deleted)
│
├── features/
│   ├── dashboard/                          (mostly unchanged)
│   ├── crosswalk/                          (mostly unchanged)
│   ├── system/                             (rename class System → SystemListComponent)
│   ├── template-builder/                   (refactored editor; saveTemplate wired)
│   ├── template-wizard/                    (mostly unchanged)
│   ├── export-bundle/                      (uses FileDownloadService)
│   │
│   └── mapping-studio/
│       ├── mapping-studio.ts                ← ~280 LOC
│       ├── mapping-studio-init.service.ts   ← NEW
│       ├── bridge-commit-builder.ts         ← NEW
│       ├── bridge-card/                     (mostly unchanged)
│       ├── schema-tree/                     (unchanged)
│       ├── field-connection-dialog/         ← FULLY SPLIT (Refactor 1)
│       │   ├── field-connection-dialog.ts   ← ~250 LOC
│       │   ├── state/
│       │   │   ├── connection-form.store.ts
│       │   │   └── connection-form.types.ts
│       │   ├── tabs/
│       │   │   ├── source-tab/
│       │   │   ├── target-tab/
│       │   │   ├── crosswalk-tab/
│       │   │   ├── transform-tab/
│       │   │   ├── value-map-rules/
│       │   │   └── date-card/
│       │   └── scriban-operators.constants.ts
│       └── validation-preview/              ← SPLIT (Refactor 6)
│           ├── validation-preview.ts        ← ~200 LOC
│           ├── template-json-builder.service.ts
│           ├── scriban-json-stringifier.service.ts
│           ├── scriban-syntax-highlighter.service.ts
│           └── parts/
│               ├── version-panel.component.ts
│               └── viva-panel.component.ts
│
├── models/                                 (unchanged + backend/ subfolder)
│   ├── backend/                            ← NEW
│   │   ├── template-raw.ts
│   │   └── bridge-raw.ts
│   ├── studio.models.ts
│   ├── template.models.ts
│   └── crosswalk.models.ts
│
├── services/
│   ├── file-download.service.ts            ← NEW (CC4)
│   ├── studio/                             ← SPLIT (Refactor 3)
│   │   ├── studio-state.service.ts          ← facade
│   │   ├── studio.store.ts
│   │   ├── studio-template.api.ts
│   │   ├── studio-scriban.api.ts
│   │   ├── studio-persistence.service.ts
│   │   └── studio-domain.service.ts
│   ├── scriban/
│   │   ├── scriban-template.service.ts      (facade — already exists)
│   │   ├── bridge-enrichment.service.ts     (rename of scriban-bridge-enrichment)
│   │   ├── expression-builder.service.ts    (rename)
│   │   ├── path-parser.service.ts           (rename)
│   │   ├── transform-preview.service.ts     (rename)
│   │   └── emitter/                         ← SPLIT (Refactor 5)
│   │       ├── scriban-template-emitter.service.ts
│   │       ├── target-tree.ts
│   │       ├── serialize-tree.ts
│   │       └── strategies/
│   │           ├── emission-strategy.ts
│   │           ├── single-collection.strategy.ts
│   │           ├── multi-collection.strategy.ts
│   │           └── multi-record-type.strategy.ts
│   ├── crosswalk.service.ts                 (unchanged)
│   ├── system.service.ts                    (remove MOCK_SYSTEMS fallback)
│   ├── dashboard.service.ts                 (uses typed mappers, no `any`)
│   ├── export-bundle.service.ts             (uses FileDownloadService)
│   ├── fhir-schema-loader.service.ts        (unchanged)
│   ├── bridge-normalization.service.ts      (unchanged)
│   ├── new-template-state.service.ts        (unchanged)
│
│   ⛔ DELETED (Refactor cleanup):
│   ⛔   rule-engine.service.ts              ← unused
│   ⛔   ruleset.service.ts                  ← deprecated stub
│   ⛔   scriban-utility.service.ts          ← unused
│   ⛔   template-validation.service.ts      ← unused
│   ⛔   mock-systems.data.ts                ← prod fallback removed
│
├── shared/
│   ├── confirm-dialog/                     (unchanged)
│   ├── scriban-editor/                     ← NEW (Refactor 8)
│   │   └── scriban-editor.directive.ts
│   │
│   ⛔ DELETED:
│   ⛔   header/                              (in core/layout, was already dead)
│   ⛔   status-badge/                        (never used)
│   ⛔   tag-badge/                           (never used)
│   ⛔   progress-bar/                        (never used)
│
└── utils/
    ├── scriban/                            ← SPLIT (Refactor 2)
    │   ├── index.ts
    │   ├── scriban.types.ts
    │   ├── convert-template.ts
    │   ├── path-helpers.ts
    │   ├── label-helpers.ts
    │   ├── variables.ts
    │   ├── formatting.ts
    │   ├── collection-pattern/
    │   │   ├── detect-collection.ts
    │   │   ├── build-collection.ts
    │   │   ├── build-nested.ts
    │   │   └── emit-siblings.ts
    │   └── transforms/
    │       ├── crosswalk-kind.ts
    │       ├── date-format-kind.ts
    │       ├── custom-script-kind.ts
    │       └── null-safe.ts
    ├── schema-parsers/                     ← SPLIT (Refactor 4)
    │   ├── schema-to-nodes.ts
    │   ├── id-generator.ts
    │   ├── json/
    │   │   ├── json-to-nodes.ts
    │   │   ├── infer-type.ts
    │   │   └── preview.ts
    │   ├── xml/
    │   │   └── xml-to-nodes.ts
    │   └── fhir/
    │       ├── fhir-structure-def.ts
    │       ├── fhir-types.ts
    │       ├── expand-complex.ts
    │       ├── type-resolution.ts
    │       └── tree-helpers.ts
    ├── path-context.util.ts                (unchanged)
    ├── validation.util.ts                  (unchanged)
    └── date-format.util.ts                 (unchanged)
```

---

## Summary — Numbers After Refactor

| Metric | Before | After |
|---|---:|---:|
| Largest file | 1,864 LOC | ~300 LOC |
| Files over 500 LOC | 6 | 0 |
| Total files in `services/` | 19 | ~22 |
| Total files in `utils/` | 6 | ~24 |
| Total files in `features/mapping-studio/` | 7 components | ~20 components |
| `any` annotations | 197 | < 30 |
| `console.*` calls | 103 | < 10 (or 0 with proper logger) |
| Initial bundle (estimate) | All routes eager | ~40-50% smaller via lazy load |
| Components with `OnPush` | 0 | All feature components |

---

## Suggested Refactor Order (Risk-Adjusted)

If you can only do this in 4 sprints, do it in this order — each sprint produces a deliverable, safe-to-ship state:

**Sprint 1 — Cleanup & infrastructure**
- Delete dead components/services (analysis doc §6.6).
- Move `mongoose`, `cors`, `express` out of `dependencies`.
- Add `FileDownloadService` and refactor 4 call sites.
- Add `unsavedChangesGuard` for Mapping Studio.
- Add `ErrorInterceptor` + `CorrelationIdInterceptor`.
- Lazy-load all routes.

**Sprint 2 — Split `studio-state.service.ts`**
- Lowest-risk split because facade preserves API.
- Adopt `OnPush` on top 4 components.
- Replace `: any` in `DashboardService` with typed mappers.

**Sprint 3 — Split `template-to-scriban.util.ts` and `schema-to-nodes.util.ts`**
- Pure utility split — no consumers change.
- Run existing specs after each move to catch regressions.

**Sprint 4 — Split `field-connection-dialog.ts` and `validation-preview.ts`**
- The biggest, riskiest work. Save for last so all the infrastructure (store pattern, child-component testing patterns) is well-established.

---

*End of refactoring playbook.*