# Mapping Studio UI — Cleanup Guide

> Three actions, one document:
>
> 1. **DELETE** — code that's dead and can be removed today
> 2. **MERGE** — multiple files that should become one (duplication, near-duplication)
> 3. **SPLIT** — files too large to stay as one
>
> Every entry shows real code from the current repo, the proposed change, and where to find it.

---

## Executive Summary

| Action | Items | Estimated LOC removed/saved | Risk |
|---|---|---|---|
| **DELETE — unused files** | 4 components, 3 services, 1 duplicate env, mock data | ~1,000 LOC + their spec files | 🟢 None — nothing references them |
| **DELETE — dead in-file code** | 103 `console.*` calls, 1 unused type alias, commented-out code blocks, legacy routes | ~150 LOC + cleaner output | 🟢 None |
| **DELETE — wrong dependencies** | `mongoose`, `cors`, `express` from `dependencies` | smaller bundle, faster install | 🟢 None — never imported |
| **MERGE — 3 delete dialogs into the existing `ConfirmDialog`** | 6 files (3 components × 2 files each) → 0 (use existing) | ~330 LOC removed | 🟡 Touches 3 features (15 min each) |
| **MERGE — `SystemCard` + `CrosswalkCard` patterns** | 2 components share a base | ~120 LOC saved | 🟡 Small refactor |
| **MERGE — 5 environment files → 2** | `environment.dev.ts`/`environment.prod.ts` are duplicates; `environment.production.ts` is unreferenced | 3 files removed | 🟢 Verify `angular.json` |
| **MERGE — `mock-systems.data.ts` → into service (or delete)** | 1 file | 53 LOC | 🟢 Removing the prod fallback is the right move |
| **SPLIT — 6 oversized files** | See Refactoring Playbook for details | — | 🔴 High effort, see playbook |

**Total** safe deletions + simple merges = roughly **1,500-1,800 lines of dead or redundant code removed**, plus 3 dependencies and 3 environment files. None of this requires architectural changes — it's all just cleanup.

---

# Part 1 — DELETE

## 1.1 Unused Components (4)

These components are declared, compile, have spec files, but **no template anywhere references their selector**. Safe to delete with their HTML, CSS, and spec files.

| Folder to delete | Selector | Why it's dead |
|---|---|---|
| `src/app/core/layout/header/` | `app-header` | `grep` for `<app-header` returns only the definition. No template uses it. |
| `src/app/shared/status-badge/` | `app-status-badge` | Not referenced. Dashboard uses Material `mat-chip` for the same UI. |
| `src/app/shared/tag-badge/` | `app-tag-badge` | Not referenced. |

### Verification before deleting

```bash
# Confirm no template uses the selector before deleting
grep -rn "<app-header" src/         # 0 matches → safe
grep -rn "<app-status-badge" src/   # 0 matches → safe
grep -rn "<app-tag-badge" src/      # 0 matches → safe
grep -rn "<app-progress-bar" src/   # 0 matches → safe
```

### The Header component (full file)

```ts
// src/app/core/layout/header/header.ts — declared, never used
import { Component } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-header',
  imports: [],
  templateUrl: './header.html',
  styleUrl: './header.css',
})
export class Header {
  constructor(private router: Router) {}

  goToNewTemplate(): void {
    this.router.navigate(['/templates/new/schema-setup']);
  }
}
```

`goToNewTemplate()` exists, but no template instantiates `<app-header>` to display the button that would call it. The Dashboard has its own "New Template" button, which makes this component redundant.

---

## 1.2 Unused Services (3)

These services are exported from `@Injectable({ providedIn: 'root' })` but **no component or service injects them**.

| File | LOC | What its spec says |
|---|---|---|
| `src/app/services/rule-engine.service.ts` | 255 | `RuleEngineService` is only referenced in its own spec file. No consumer. |
| `src/app/services/scriban-utility.service.ts` | 167 | Mentioned only in a code comment inside `rule-engine.service.ts`. Never injected. |
| `src/app/services/template-validation.service.ts` | ~150 | `TemplateValidationService` is exported but never imported. |

### Verification

```bash
# Confirm no consumer
grep -rn "RuleEngineService"            src/ --include="*.ts" | grep -v "rule-engine.service" | grep -v ".spec.ts"
grep -rn "ScribanUtilityService"        src/ --include="*.ts" | grep -v "scriban-utility.service"
grep -rn "TemplateValidationService"    src/ --include="*.ts" | grep -v "template-validation.service"
# Each returns 0 lines → safe
```

### Files to delete

```
src/app/services/rule-engine.service.ts
src/app/services/scriban-utility.service.ts
src/app/services/template-validation.service.ts
src/app/services/spec/rule-engine.service.spec.ts
src/app/services/spec/scriban-utility.service.spec.ts        (if exists)
src/app/services/spec/template-validation.service.spec.ts    (if exists)
```

---

## 1.3 Deprecated Stub Service (1)

| File | LOC | Status |
|---|---|---|
| `src/app/services/ruleset.service.ts` | 19 | `@deprecated` annotated; methods return `of([])` / `of(null)` |

It's still injected by `TemplateBuilderComponent` and `FieldConnectionDialogComponent`, so you can't delete the file outright. The full content:

```ts
// src/app/services/ruleset.service.ts — currently stubbed
/**
 * @deprecated RuleSet concept has been removed. This stub exists for backwards
 * compatibility while legacy components are migrated. Always returns empty collections.
 */
@Injectable({ providedIn: 'root' })
export class RuleSetService {
  /** @deprecated Returns empty array. RuleSets are no longer used. */
  getAllRuleSets(): Observable<any[]> {
    return of([]);
  }
  /** @deprecated Returns null. RuleSets are no longer used. */
  getRuleSetById(_id: string): Observable<any | null> {
    return of(null);
  }
}
```

### Cleanup steps

1. Find every `ruleSetService.*` call. Two consumers:
   - `TemplateBuilderComponent` — `loadCrosswalksAndRuleSets()`. The `getAllRuleSets()` result feeds `availableRuleSets`, which feeds the (always empty) right-rail Ruleset panel. Just delete the call and the panel UI.
   - `FieldConnectionDialogComponent` — `loadRuleSets()`. Same — feeds `availableRuleSets`, `collectionRuleSets`, `transformRuleSets` arrays plus the "RuleSet picker" UI block. Delete those arrays, the picker, and the methods that touch them.
2. Delete `src/app/services/ruleset.service.ts` and its spec.
3. Delete the related `ruleSetIds` plumbing in `Bridge.transform.ruleSetIds` if no other code reads it.

---

## 1.4 Duplicate / Unreferenced Environment Files

There are **5 environment files** but `angular.json` only references 3 of them. Worse, two of them are byte-identical.

```bash
$ wc -l src/environments/*.ts
   8 environment.dev.ts
   8 environment.prod.ts
  13 environment.production.ts
   8 environment.qa.ts
  12 environment.ts
```

The actual contents:

```ts
// environment.dev.ts
export const environment = {
  production: true,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && (window as any).__env?.API_URL)
      ? (window as any).__env.API_URL
      : 'https://tiestemplateengine.bluewave-80119095.westus3.azurecontainerapps.io/api';
  }
};

// environment.prod.ts                  ← IDENTICAL to environment.dev.ts (same URL too!)
export const environment = {
  production: true,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && (window as any).__env?.API_URL)
      ? (window as any).__env.API_URL
      : 'https://tiestemplateengine.bluewave-80119095.westus3.azurecontainerapps.io/api';
  }
};

// environment.qa.ts                    ← differs ONLY in the URL
…
      : 'https://tiestemplateengine.mangograss-9690bc81.westus3.azurecontainerapps.io/api';

// environment.production.ts            ← NOT referenced anywhere
//   Uses empty-string fallback (the correct "build once, deploy anywhere" pattern).
export const environment = {
  production: true,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && window.__env?.API_URL)
      ? window.__env.API_URL
      : '';
  }
};

// environment.ts                       ← dev/local fallback
export const environment = {
  production: false,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && window.__env?.API_URL)
      ? window.__env.API_URL
      : 'https://localhost:7000/api';
  }
};
```

### Recommended cleanup

1. **Delete `environment.production.ts`** outright. It's not referenced from `angular.json` and the file with the cleaner pattern (empty-string fallback) isn't the one used in production builds anyway.
2. **Delete `environment.prod.ts`** OR **delete `environment.dev.ts`** — they're identical. Keep the one that `angular.json` references.
3. See the **MERGE** section (§2.3) for consolidating the rest with a per-env config object instead of file duplication.

---

## 1.5 Wrong Dependencies in `package.json`

Three Node-only packages live in `dependencies` (not `devDependencies`). They will increase `npm install` time and — critically — `mongoose` could potentially get bundled into the browser build if accidentally imported.

```jsonc
// package.json — current
"dependencies": {
  "@angular/animations": "21.1.0",
  …
  "cors":     "^2.8.5",            ← Node-only Express middleware. Wrong section.
  "express":  "^5.1.0",            ← Node-only HTTP server. Wrong section.
  "mongoose": "^9.2.4",            ← Node-only MongoDB ORM. MUST NOT BUNDLE.
  …
}
```

### Verification

```bash
grep -rn "from 'mongoose'\|from \"mongoose\"" src/   # 0
grep -rn "from 'express'\|from \"express\""   src/   # 0
grep -rn "from 'cors'\|from \"cors\""         src/   # 0
```

Nothing in the app actually imports any of these. The Docker build uses `nginx` to serve static files; there's no Node server. These dependencies are leftovers from some earlier experiment.

### Fix

```bash
npm uninstall mongoose cors express
```

Or, if any tooling actually needs them, move them to `devDependencies`:

```jsonc
"devDependencies": {
  …
  "cors":     "^2.8.5",
  "express":  "^5.1.0"
  // mongoose should just be removed — there's no scenario where a frontend repo needs it
}
```

---

## 1.6 Unused Type Aliases

```ts
// src/app/models/template.models.ts — last line
// Keep the old name for backward compatibility
export type NewProjectState = NewTemplateState;
```

```bash
$ grep -rn "NewProjectState" src/ --include="*.ts" | grep -v ".spec.ts"
# 0 matches — only the alias declaration itself
```

Nothing imports the type as `NewProjectState`. Safe to delete:

```ts
// AFTER
// (the alias is gone — keep just NewTemplateState)
export type SchemaType = 'source' | 'target';
export interface UploadedSchema { /* … */ }
export interface NewTemplateState { /* … */ }
```

---

## 1.7 Dead Console Statements (103 calls!)

```bash
$ grep -rn "console\." src/app --include="*.ts" | grep -v ".spec.ts" | wc -l
103
```

These statements ship in production. Worst offenders:

| File | Offence |
|---|---|
| `services/dashboard.service.ts` | `console.warn` on **every** template fetched in `computeCoverage` — fires for every dashboard load |
| `features/mapping-studio/mapping-studio.ts` | `console.log('🆕 NEW TEMPLATE: …')`, `console.log('✏️ EDIT TEMPLATE: …')` on every init |
| `features/dashboard/dashboard.component.ts` | A whole `console.group()` block for "analytics" on every load |
| `services/studio-state.service.ts` | Multiple `console.log` for save/load progress |

### Quick win — strip in production builds

Option A — One-liner Angular CLI option:

```jsonc
// angular.json (under projects.<name>.architect.build.configurations.production)
"optimization": {
  "scripts": true,
  "styles": { "minify": true, "inlineCritical": true },
  "fonts": true
}
```

(Optimization already drops most consts, but `console.*` is preserved by default. Use a `terser` option via custom webpack config to drop them.)

Option B — Proper fix, introduce a logger:

```ts
// src/app/core/logger.service.ts (NEW)
import { Injectable, isDevMode } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(...args: unknown[]):  void { if (isDevMode()) console.log(...args); }
  info(...args: unknown[]): void { if (isDevMode()) console.info(...args); }
  warn(...args: unknown[]): void { console.warn(...args); }                    // always show
  error(...args: unknown[]): void { console.error(...args); }                  // always show
  group(label: string): void   { if (isDevMode()) console.group(label); }
  groupEnd(): void              { if (isDevMode()) console.groupEnd(); }
}
```

Then replace every `console.log(...)` with `this.logger.log(...)`. Warnings and errors stay visible in production; chatty `log()` calls vanish.

---

## 1.8 Commented-Out Code Blocks

A few spots have commented-out stat tiles, snippets, or import lines that should just be deleted (Git has the history if you ever need them back):

```html
<!-- src/app/core/layout/sidebar/sidebar.html lines 39-44 -->
<!--<a class="nav-item" routerLink="/rule-builder" routerLinkActive="active">
  <svg class="nav-icon" …>…</svg>
  <span>Rule Builder</span>
</a>-->
```

```ts
// src/app/features/dashboard/dashboard.component.ts — statTiles signal
// Commented-out tiles for "Crosswalks" and "Rule Sets" suggest features that were removed.
// Delete the comments.
```

---

## 1.9 Legacy Routes (audit before deleting)

```ts
// src/app/app.routes.ts
{ path: 'home',                        redirectTo: 'dashboard',                  pathMatch: 'full' },
{ path: 'projects/new/schema-setup',   redirectTo: 'templates/new/schema-setup', pathMatch: 'full' },
{ path: 'projects/:id/studio',         redirectTo: 'templates/:id/studio',       pathMatch: 'full' }
```

If access-log analysis shows zero traffic to `/home` or `/projects/*` in the last 90 days, delete the three redirects.

---

## 1.10 The `MOCK_SYSTEMS` Production Fallback

```ts
// src/app/services/system.service.ts (excerpt)
getAllSystems(): Observable<System[]> {
  return this.http.get<System[]>(`${this.apiUrl}/system`).pipe(
    catchError(() => {
      console.warn('Backend unavailable, returning mock systems');
      return of(MOCK_SYSTEMS);                ← silently masks backend outages in production
    })
  );
}
```

This is a **production code smell**. When the backend goes down, every user silently sees the same 5 fake systems ("Epic EHR", "Cerner Millennium", etc.) with no indication anything is wrong. They try to map data using systems that don't really exist.

### Fix

```ts
// AFTER
getAllSystems(): Observable<System[]> {
  return this.http.get<System[]>(`${this.apiUrl}/system`);
  // Let the error propagate. The Error HttpInterceptor (see Refactoring Playbook §CC6)
  // surfaces backend outages to the user via a snackbar.
}
```

Then delete `src/app/services/mock-systems.data.ts` entirely.

---

# Part 2 — MERGE

## 2.1 Merge 3 Delete Dialogs into the Existing `ConfirmDialogComponent`

**The biggest single cleanup win in the project.**

Today there are **three near-identical delete-confirmation dialog components**, each with its own `.ts`, `.html`, `.scss`, and `.spec.ts`:

```
features/dashboard/delete-template-dialog/        — 37 LOC component + html + scss + spec
features/system/delete-system-dialog/             — 37 LOC component + html + scss + spec
features/crosswalk/delete-crosswalk-dialog/       — 36 LOC component + html + scss + spec
```

**All three have identical logic:**

```ts
// Each component is essentially this same shell:
@Component({ selector: 'app-delete-X-dialog', standalone: true, imports: [CommonModule, MatDialogModule, MatButtonModule, MatIconModule], … })
export class DeleteXDialogComponent {
  constructor(
    public dialogRef: MatDialogRef<DeleteXDialogComponent>,
    @Inject(MAT_DIALOG_DATA) public data: DeleteXDialogData
  ) {}
  cancel():  void { this.dialogRef.close(false); }
  confirm(): void { this.dialogRef.close(true);  }
}
```

The only differences are:
- The `DialogData` interface shape (templateName / systemName+systemType / crosswalkName)
- The wording in the HTML body

### The trick — `ConfirmDialogComponent` already exists and does this

```ts
// src/app/shared/confirm-dialog/confirm-dialog.component.ts — already in the codebase
export interface ConfirmDialogData {
  title: string;
  message: string;
  confirmText?: string;
  cancelText?: string;
  icon?: string;
  warn?: boolean;
}
```

It accepts a `title` and a `message`. The Mapping Studio already uses it for "Delete bridge?", "Reset all bridges?", "Leave the studio?".

The 3 delete dialogs are **redundant** — they could all be replaced by `ConfirmDialog` with appropriate data.

### Before — Dashboard (4 files to maintain)

```ts
// dashboard.component.ts
import { DeleteTemplateDialogComponent } from './delete-template-dialog/delete-template-dialog.component';

deleteTemplate(t: MappingProject, event: Event): void {
  event.stopPropagation();
  const dialogRef = this.dialog.open(DeleteTemplateDialogComponent, {
    data: { templateName: t.title },
    width: '480px'
  });
  dialogRef.afterClosed().subscribe(confirmed => {
    if (confirmed) {
      this.dashboardService.deleteProject(t.id).subscribe(/* … */);
    }
  });
}
```

### After — Dashboard (no separate dialog needed)

```ts
// dashboard.component.ts
import { ConfirmDialogComponent } from '../../shared/confirm-dialog/confirm-dialog.component';

deleteTemplate(t: MappingProject, event: Event): void {
  event.stopPropagation();
  this.dialog.open(ConfirmDialogComponent, {
    data: {
      title: 'Delete Template?',
      message: `Are you sure you want to delete "${t.title}"? This cannot be undone.`,
      confirmText: 'Delete',
      cancelText: 'Cancel',
      icon: 'delete',
      warn: true
    },
    width: '480px'
  }).afterClosed().subscribe(confirmed => {
    if (confirmed) {
      this.dashboardService.deleteProject(t.id).subscribe(/* … */);
    }
  });
}
```

### Cleanup checklist

1. Update each callsite (3 total) to open `ConfirmDialogComponent` with appropriate data.
2. Delete these folders entirely:
   ```
   src/app/features/dashboard/delete-template-dialog/
   src/app/features/system/delete-system-dialog/
   src/app/features/crosswalk/delete-crosswalk-dialog/
   ```
3. Remove their imports from the parent components.

### Result

- **~330 LOC removed** (3 components × ~110 LOC counting `.scss` and `.spec.ts` each)
- **3 fewer files to maintain**
- One place to change the look of every confirmation dialog in the app

---

## 2.2 `SystemCardComponent` and `CrosswalkCardComponent` Share a Base

The two card components have nearly identical structure:

```ts
// system-card.component.ts
@Component({
  selector: 'app-system-card',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatIconModule, MatButtonModule, MatTooltipModule, MatChipsModule, MatDividerModule],
  templateUrl: './system-card.component.html',
  styleUrls: ['./system-card.component.scss']
})
export class SystemCardComponent {
  @Input({ required: true }) system!: SystemModel;
  @Output() edit   = new EventEmitter<SystemModel>();
  @Output() delete = new EventEmitter<SystemModel>();

  onEdit():   void { this.edit.emit(this.system); }
  onDelete(): void { this.delete.emit(this.system); }
}
```

```ts
// crosswalk-card.component.ts — same imports, same Input/Output pattern
@Component({
  selector: 'app-crosswalk-card',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatIconModule, MatButtonModule, MatTooltipModule, MatChipsModule, MatDividerModule],
  templateUrl: './crosswalk-card.component.html',
  styleUrls: ['./crosswalk-card.component.scss']
})
export class CrosswalkCardComponent {
  @Input({ required: true }) crosswalk!: Crosswalk;
  @Input() count: number = 0;
  @Output() edit   = new EventEmitter<Crosswalk>();
  @Output() delete = new EventEmitter<Crosswalk>();
  @Output() view   = new EventEmitter<Crosswalk>();    // ⚠ declared but never emitted
}
```

The HTML/CSS differs (one shows database icon + "System Type" badge, the other shows crosswalk-specific info), but they could share a generic shell with content projection.

### Option A — generic `EntityCard` with content projection

```ts
// src/app/shared/entity-card/entity-card.component.ts (NEW)
@Component({
  selector: 'app-entity-card',
  standalone: true,
  imports: [CommonModule, MatCardModule, MatIconModule, MatButtonModule, MatTooltipModule, MatDividerModule],
  template: `
    <mat-card class="entity-card">
      <div class="card-header">
        <div class="title-group">
          <ng-content select="[card-icon]"></ng-content>
          <div class="title-content">
            <div class="title">{{ title }}</div>
            <div class="subtitle">{{ subtitle }}</div>
          </div>
        </div>
        <div class="actions">
          <button mat-icon-button (click)="edit.emit()" matTooltip="Edit"><mat-icon>edit</mat-icon></button>
          <button mat-icon-button (click)="delete.emit()" matTooltip="Delete"><mat-icon>delete</mat-icon></button>
        </div>
      </div>
      <mat-divider></mat-divider>
      <ng-content></ng-content>  <!-- per-card content slot -->
    </mat-card>
  `,
  styleUrls: ['./entity-card.component.scss']
})
export class EntityCardComponent {
  @Input({ required: true }) title!: string;
  @Input() subtitle = '';
  @Output() edit   = new EventEmitter<void>();
  @Output() delete = new EventEmitter<void>();
}
```

### After — `SystemCardComponent` becomes a thin wrapper (or disappears)

```html
<!-- system.html — used to look like this -->
<app-system-card *ngFor="let s of systems" [system]="s" (edit)="openEditDialog(s)" (delete)="openDeleteDialog(s)" />

<!-- After — directly use EntityCard, no wrapper needed -->
<app-entity-card *ngFor="let s of systems"
  [title]="s.name"
  [subtitle]="(s.type | titlecase) + ' System'"
  (edit)="openEditDialog(s)"
  (delete)="openDeleteDialog(s)">
  <svg card-icon class="database-icon" viewBox="0 0 24 24">…</svg>   <!-- icon slot -->
  <div class="info-band">…system-specific info…</div>                <!-- main content -->
</app-entity-card>
```

### Cleanup result

- Delete `SystemCardComponent` (38 LOC) and its spec (735 LOC).
- Delete `CrosswalkCardComponent` (46 LOC) and its spec (445 LOC).
- The new `EntityCardComponent` adds ~30 LOC.
- Net: **~1,200 LOC removed** (mostly because the original cards each carried ridiculously long spec files for what is, essentially, an `@Input` + `@Output` shell).

### Risks

- The CSS in `system-card.component.scss` and `crosswalk-card.component.scss` is non-trivial (134 + 303 LOC). Some will move to `entity-card.component.scss`, the rest stays with the parent feature.
- The `view` output on `CrosswalkCardComponent` is declared but never used — delete it too.

> **Trade-off:** Option A merges aggressively. **Option B** is to keep both wrappers but have them extend a common base class. Less drastic, less LOC saved, but smaller risk surface.

---

## 2.3 Consolidate Environment Files (5 → 2)

### Current

```
src/environments/environment.ts            ← dev (production: false, localhost)
src/environments/environment.dev.ts        ← identical to .prod.ts
src/environments/environment.prod.ts       ← identical to .dev.ts (!)
src/environments/environment.qa.ts         ← differs only in URL
src/environments/environment.production.ts ← NOT referenced from angular.json
```

### Proposed

Keep **two files**:

```
src/environments/environment.ts            ← dev/local default (production: false)
src/environments/environment.production.ts ← production (production: true, empty fallback)
```

Move per-environment URLs to a **single config file**, switched at build time via `fileReplacements` in `angular.json`:

```ts
// src/environments/api-urls.ts (NEW — shared)
export const API_URLS = {
  dev:  'https://tiestemplateengine.bluewave-80119095.westus3.azurecontainerapps.io/api',
  qa:   'https://tiestemplateengine.mangograss-9690bc81.westus3.azurecontainerapps.io/api',
  local: 'https://localhost:7000/api',
};
```

```ts
// src/environments/environment.ts (LOCAL DEV)
import { API_URLS } from './api-urls';
export const environment = {
  production: false,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && window.__env?.API_URL)
      ? window.__env.API_URL
      : API_URLS.local;
  }
};
```

```ts
// src/environments/environment.production.ts (PRODUCTION)
import { API_URLS } from './api-urls';
export const environment = {
  production: true,
  get apiUrl(): string {
    return (typeof window !== 'undefined' && window.__env?.API_URL)
      ? window.__env.API_URL
      : '';   // production runtime MUST provide window.__env.API_URL via docker-entrypoint.sh
  }
};
```

Then update `angular.json` `configurations`:

```jsonc
"dev": {
  "fileReplacements": [
    { "replace": "src/environments/environment.ts", "with": "src/environments/environment.production.ts" }
  ]
}
```

(Or keep the dev fallback URL by replacing just the URL inside the existing files.)

### Files to delete after migration

```
src/environments/environment.dev.ts        ← duplicate
src/environments/environment.prod.ts       ← duplicate
src/environments/environment.qa.ts         ← URL moved into api-urls.ts
```

---

## 2.4 Move Mock Data Into Its Service — Or Just Remove It

`mock-systems.data.ts` is a **53-LOC array of fake data** used as a fallback when the backend errors:

```ts
// src/app/services/mock-systems.data.ts
export const MOCK_SYSTEMS: System[] = [
  { id: '1', name: 'Epic EHR',           type: 'source', /* … */ },
  { id: '2', name: 'Cerner Millennium',  type: 'source', /* … */ },
  { id: '3', name: 'FHIR Server',        type: 'target', /* … */ },
  { id: '4', name: 'HL7 v2',             type: 'both',   /* … */ },
  { id: '5', name: 'Data Warehouse',     type: 'target', /* … */ }
];
```

### Recommended (best): just delete it

As discussed in §1.10, the production fallback to mock data is a code smell — it silently masks backend outages. Delete the import from `SystemService`, delete the file, let the proper error handler surface the failure.

### Alternative (if you really want test-only data): move into spec file

If `MOCK_SYSTEMS` is genuinely useful for *tests*, move it into the spec file itself:

```ts
// system.service.spec.ts (or test fixtures file)
const MOCK_SYSTEMS_FIXTURE: System[] = [/* … */];
```

Then it's clearly scoped to testing and won't pollute the production bundle.

---

## 2.5 Optional — Consolidate Model Files With a Barrel Export

Currently:

```
src/app/models/
├── crosswalk.models.ts   (19 LOC)
├── studio.models.ts      (241 LOC)
└── template.models.ts    (29 LOC)
```

Imports across the codebase are verbose:

```ts
import { Bridge, SchemaNode, FieldType } from '../../models/studio.models';
import { UploadedSchema, NewTemplateState } from '../../models/template.models';
import { Crosswalk } from '../../models/crosswalk.models';
```

### Proposal — add a barrel export

Keep the per-domain files but add an index:

```ts
// src/app/models/index.ts (NEW)
export * from './studio.models';
export * from './template.models';
export * from './crosswalk.models';
```

Then consumers do:

```ts
import { Bridge, SchemaNode, FieldType, UploadedSchema, NewTemplateState, Crosswalk } from '../../models';
```

### Risk

`studio.models.ts` (241 LOC) is the only one large enough to keep separated. If you ever want **one** file:

```ts
// src/app/models/index.ts (alternative — single file)
// All model types in one place. Keep separators clear.

// ── Studio ───────────────────────────────────────────────────────────
export type FieldType = 'string' | 'number' | 'boolean' | 'date' | 'array' | 'object' | 'unknown';
export interface SchemaNode { /* … */ }
export interface Bridge { /* … */ }
// … the rest of studio.models.ts

// ── Template ─────────────────────────────────────────────────────────
export type SchemaType = 'source' | 'target';
export interface UploadedSchema { /* … */ }
export interface NewTemplateState { /* … */ }

// ── Crosswalk ────────────────────────────────────────────────────────
export interface CrosswalkMapping { /* … */ }
export interface Crosswalk { /* … */ }
```

> **Recommendation:** Stick with **3 files + barrel**. Merging into one 289-line file makes maintenance harder, not easier.

---

# Part 3 — SPLIT

The big files need to be split. This is covered in detail in `Refactoring_Playbook.md`. Summary table:

| File | LOC | Split into | Why |
|---|---:|---|---|
| `field-connection-dialog.ts` | 1,864 | A scoped `ConnectionFormStore` + 6 tab sub-components | 130+ methods, 40+ fields, 6 distinct responsibilities in one class |
| `template-to-scriban.util.ts` | 1,270 | Folder of 7 cohesive modules (collection-pattern/, transforms/, etc.) | 38 functions across two completely separate code paths (collection vs default) |
| `studio-state.service.ts` | 1,123 | Facade + Store + 2 API services + Persistence + Domain | Three responsibilities (state / HTTP / domain) jammed together |
| `schema-to-nodes.util.ts` | 843 | Per-format folders (json/, xml/, fhir/) + extract `FHIR_DATA_TYPES` | Three parsers in one file; the 250-line `FHIR_DATA_TYPES` const dominates |
| `scriban-template-emitter.service.ts` | 829 | Strategy pattern: 3 strategy classes + target-tree helpers | Emission has three modes (single/multi/cross-collection) all in one switch |
| `validation-preview.ts` | 819 | UI shell + 3 services + 2 child components | Mixes UI orchestration with JSON stringify, syntax highlight, version mgmt, Viva mode |

See the **Refactoring Playbook** for concrete before/after code for each split, the migration steps, and the risk analysis.

---

# Part 4 — Cleanup Order (4 Pull Requests)

If you can only spend a week or two on cleanup, do it in this order. Each PR is **independently shippable**.

## PR 1 — Pure deletions (1-2 hours)

No behavior changes; no risk. Just remove dead files.

1. Delete 4 unused components (`Header`, `StatusBadge`, `TagBadge`, `ProgressBar`).
2. Delete 3 unused services (`RuleEngineService`, `ScribanUtilityService`, `TemplateValidationService`).
3. Delete `environment.production.ts` (not referenced) and either `environment.dev.ts` or `environment.prod.ts` (duplicate of the other).
4. Delete `NewProjectState` type alias from `template.models.ts`.
5. Delete commented-out sidebar nav item (`<!--<a routerLink="/rule-builder">…-->`).
6. `npm uninstall mongoose cors express`.

**Result:** ~1,000 LOC + 3 npm dependencies removed.

## PR 2 — Production hygiene (half-day)

1. Add `LoggerService`, replace ~50 of the most visible `console.*` calls (especially `DashboardService.computeCoverage`).
2. Delete `MOCK_SYSTEMS` and its fallback in `SystemService` — let real errors surface.
3. Delete `MappingStudioComponent` console-log strings like `'🆕 NEW TEMPLATE: …'`.
4. Audit & either delete or wire up the unused `view` output on `CrosswalkCardComponent`.

**Result:** No more silent fallbacks in production, dramatically cleaner browser console.

## PR 3 — Merge the 3 delete dialogs into ConfirmDialog (half-day)

1. Update each callsite (Dashboard, System, Crosswalk) to open `ConfirmDialogComponent` with appropriate data.
2. Delete the 3 delete-dialog folders and their specs.

**Result:** ~330 LOC + 3 components + 3 spec files removed. All confirmation dialogs now look identical.

## PR 4 — Finalize RuleSet deprecation (1 day)

1. Remove `ruleSetService` injection from `FieldConnectionDialogComponent` and `TemplateBuilderComponent`.
2. Remove the related UI (`availableRuleSets`, `collectionRuleSets`, `transformRuleSets`, the picker, the panel).
3. Delete `ruleset.service.ts` and its spec.
4. Audit `Bridge.transform.ruleSetIds` references — if no downstream code reads them, delete the field too.

**Result:** Complete removal of a deprecated feature.

---

## After PR 4 — The Big Splits

Once the codebase is cleaner, tackle the splits in this order (see `Refactoring_Playbook.md`):

1. Split `StudioStateService` (facade preserves all 50 consumer APIs — lowest risk).
2. Split `template-to-scriban.util.ts` (pure functions, no consumer changes via `index.ts` barrel).
3. Split `schema-to-nodes.util.ts` (same — pure functions).
4. Split `scriban-template-emitter.service.ts` (strategy pattern; uses other splits done by then).
5. Split `validation-preview.ts` (services + child components).
6. Split `field-connection-dialog.ts` (the biggest, riskiest — save for last).

---

## Quick "What to Delete" Checklist

Copy this into your PR description:

```
PR 1 — Pure deletions
- [ ] Delete src/app/core/layout/header/                       (unused component)
- [ ] Delete src/app/shared/status-badge/                      (unused component)
- [ ] Delete src/app/shared/tag-badge/                         (unused component)
- [ ] Delete src/app/shared/progress-bar/                      (unused component)
- [ ] Delete src/app/services/rule-engine.service.ts + spec    (unused service)
- [ ] Delete src/app/services/scriban-utility.service.ts + spec (unused service)
- [ ] Delete src/app/services/template-validation.service.ts + spec (unused service)
- [ ] Delete src/environments/environment.production.ts        (not referenced)
- [ ] Delete src/environments/environment.dev.ts OR .prod.ts   (duplicates)
- [ ] Delete `NewProjectState` alias from template.models.ts   (unused)
- [ ] Delete commented-out sidebar nav item                    (rule-builder)
- [ ] Run `npm uninstall mongoose cors express`                (wrong stack)

PR 2 — Production hygiene
- [ ] Add LoggerService
- [ ] Replace top 50 console.* with logger.* calls
- [ ] Delete MOCK_SYSTEMS fallback in SystemService
- [ ] Delete src/app/services/mock-systems.data.ts

PR 3 — Dialog merge
- [ ] Update Dashboard.deleteTemplate to use ConfirmDialogComponent
- [ ] Update System.deleteSystem to use ConfirmDialogComponent
- [ ] Update Crosswalk.deleteCrosswalk to use ConfirmDialogComponent
- [ ] Delete src/app/features/dashboard/delete-template-dialog/
- [ ] Delete src/app/features/system/delete-system-dialog/
- [ ] Delete src/app/features/crosswalk/delete-crosswalk-dialog/

PR 4 — RuleSet deprecation
- [ ] Remove ruleSetService from FieldConnectionDialogComponent
- [ ] Remove ruleSetService from TemplateBuilderComponent
- [ ] Remove ruleset UI (panels, pickers, arrays)
- [ ] Delete src/app/services/ruleset.service.ts + spec
- [ ] Audit & remove Bridge.transform.ruleSetIds usages
```

---

*End of cleanup guide.*