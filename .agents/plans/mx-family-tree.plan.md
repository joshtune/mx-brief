# Implementation Plan: mx-family-tree

> **PRD:** `.agents/prds/mx-family-tree.prd.md`
> **Created:** 2026-04-05
> **Strategy:** Team (parallel agent workstreams per phase)
> **Confidence:** 8/10

---

## Overview

A family tree web app for digitizing paper-based records into a flexible graph data model. Persons are nodes, relationships are typed directed edges. Supports all real-world family complexity: multiple marriages, adoption, orphans, half-siblings, step-parents, same-sex couples, and disconnected subtrees.

**Target:** MVP covering all Must Have requirements (M1–M12) plus S1 (half-sibling inference), S6 (validation warnings), and S7 (gender-agnostic relationships).

---

## Architecture

```
src/
  lib/
    components/
      ui/                   # shadcn-svelte primitives (button, input, dialog, etc.)
      person/               # PersonForm, PersonCard, PersonDetail
      relationship/         # RelationshipForm, RelationshipList, RelationshipBadge
      tree/                 # FamilyTree (Cytoscape wrapper), TreeControls, TreeLegend
      layout/               # Nav, SearchBar, FilterPanel, Sidebar
    services/
      persons.ts            # Supabase CRUD for person table
      relationships.ts      # Supabase CRUD for relationship table
      graph.ts              # Transform DB records → Cytoscape elements
      inference.ts          # Half-sibling / step-sibling inference logic
      validation.ts         # Relationship validation warnings
    types/
      database.ts           # Auto-generated Supabase types
      graph.ts              # Cytoscape node/edge type definitions
    stores/
      tree-store.ts         # Reactive state: persons, relationships, selected node, filters
    supabase.ts             # Browser + server client initialization
  routes/
    +layout.svelte          # Root layout (Nav, auth guard)
    +layout.server.ts       # Session loader
    +page.svelte            # Dashboard → redirect to /tree
    auth/
      login/+page.svelte
      signup/+page.svelte
      logout/+page.server.ts
      callback/+server.ts
    persons/
      +page.svelte          # Person list with search
      +page.ts              # Load persons
      new/+page.svelte      # Create person form
      [id]/
        +page.svelte        # Person detail view
        +page.ts            # Load person + relationships
        edit/+page.svelte   # Edit person form
    tree/
      +page.svelte          # Full-page interactive tree visualization
      +page.ts              # Load all persons + relationships for graph
supabase/
  migrations/
    001_create_person.sql
    002_create_relationship.sql
    003_rls_policies.sql
  config.toml               # Port assignments for mx-family-tree
```

---

## Phase 0: Scaffold

**Goal:** Bootable SvelteKit project with full toolchain configured.

| # | Task | Files | Details |
|---|------|-------|---------|
| 0.1 | Init SvelteKit project | `package.json`, `svelte.config.js`, `vite.config.ts`, `tsconfig.json` | `pnpm create svelte@latest mx-family-tree` with TypeScript strict, Vitest, Playwright, ESLint, Prettier |
| 0.2 | Install dependencies | `package.json` | Core: `@sveltejs/kit`, `svelte`, `@tailwindcss/vite`, `tailwindcss`, `bits-ui`, `clsx`, `tailwind-merge`, `tailwind-variants`, `@supabase/supabase-js`, `@supabase/ssr`, `cytoscape`, `@lucide/svelte`. Dev: `vitest`, `playwright`, `eslint`, `prettier`, `svelte-check`, `typescript` |
| 0.3 | Configure Tailwind CSS 4 | `src/app.css`, `vite.config.ts` | Tailwind v4 setup via `@tailwindcss/vite` plugin |
| 0.4 | Initialize shadcn-svelte | `components.json`, `src/lib/components/ui/` | `pnpm dlx shadcn-svelte@latest init` — add button, input, label, dialog, select, card, badge, separator, tooltip, dropdown-menu, avatar, sheet |
| 0.5 | Configure Supabase | `supabase/config.toml`, `.env.example`, `.env.local` | Port block: API 54821, DB 54822, Studio 54823, Inbucket 54824, Realtime 54825, Auth 54826, Storage 54827, Edge Functions 54828, Analytics 54829, Shadow DB 54820. Dev server: 5176, Preview: 4176 |
| 0.6 | Supabase client setup | `src/lib/supabase.ts` | Browser + server clients following `@supabase/ssr` pattern from mx-au-mane |
| 0.7 | Root layout + hooks | `src/routes/+layout.svelte`, `src/routes/+layout.server.ts`, `src/hooks.server.ts` | Auth session loader, Supabase client creation in hooks |
| 0.8 | Project CLAUDE.md | `CLAUDE.md` | Project-specific conventions, port assignments, quality commands |
| 0.9 | Validate scaffold | — | Run `pnpm dev`, `pnpm check`, `pnpm lint`, `pnpm format` — all must pass clean |

**Exit criteria:** `pnpm dev` serves at localhost:5176, `pnpm check` passes, `supabase start` boots on assigned ports.

---

## Phase 1: Database & Auth

**Goal:** Person + relationship tables with RLS, auth flows working.

| # | Task | Files | Details |
|---|------|-------|---------|
| 1.1 | Person table migration | `supabase/migrations/001_create_person.sql` | All columns per PRD data model. `id` uuid PK default `gen_random_uuid()`. `user_id` uuid NOT NULL FK to `auth.users`. `first_name` and `last_name` required. `updated_at` trigger. |
| 1.2 | Relationship table migration | `supabase/migrations/002_create_relationship.sql` | All columns per PRD. `type` as text with CHECK constraint against enum values: `biological_parent`, `adoptive_parent`, `step_parent`, `foster_parent`, `guardian`, `spouse`, `partner`. Unique constraint on `(person_a_id, person_b_id, type)`. Self-referencing FK prevention: CHECK `person_a_id != person_b_id`. |
| 1.3 | RLS policies | `supabase/migrations/003_rls_policies.sql` | Enable RLS on both tables. Policies: SELECT/INSERT/UPDATE/DELETE where `user_id = auth.uid()`. Helper function `get_user_id()` for consistency. |
| 1.4 | Generate TypeScript types | `src/lib/types/database.ts` | `supabase gen types typescript --local > src/lib/types/database.ts` |
| 1.5 | Auth pages | `src/routes/auth/login/+page.svelte`, `signup/+page.svelte`, `logout/+page.server.ts`, `callback/+server.ts` | Email/password auth. shadcn form components. Redirect to `/tree` on success. |
| 1.6 | Auth guard | `src/hooks.server.ts` | Redirect unauthenticated users to `/auth/login`. Allow `/auth/*` routes without auth. |
| 1.7 | Seed data (dev only) | `supabase/seed.sql` | 10-15 sample persons with varied relationship types for development/testing. Cover: nuclear family, adoption, multiple marriages, orphan node, same-sex couple. |

**Exit criteria:** Can sign up, log in, see empty dashboard. Database tables exist with correct schema. RLS prevents cross-user access.

---

## Phase 2: Person CRUD

**Goal:** Full create/read/update/delete for person records.

| # | Task | Files | Details |
|---|------|-------|---------|
| 2.1 | Person service layer | `src/lib/services/persons.ts` | Type-safe Supabase queries: `getPersons()`, `getPerson(id)`, `createPerson(data)`, `updatePerson(id, data)`, `deletePerson(id)`. Deletion cascades relationships (DB-level CASCADE). Search by name support built in (`ilike` on first_name, last_name, maiden_name). |
| 2.2 | Person form component | `src/lib/components/person/PersonForm.svelte` | shadcn form with all PRD fields. Date pickers for birth/death. Gender select. Maiden name conditional on context. Svelte 5 runes for form state. Validation: first_name + last_name required, death_date > birth_date if both present. |
| 2.3 | Person card component | `src/lib/components/person/PersonCard.svelte` | Compact card: avatar (photo or initials), full name, birth–death year range, gender icon. Click navigates to detail. |
| 2.4 | Person list page | `src/routes/persons/+page.svelte`, `+page.ts` | Grid of PersonCards. Search input (debounced, filters by name). "Add person" button → `/persons/new`. Sort by last name (default), birth date, created date. |
| 2.5 | Person create page | `src/routes/persons/new/+page.svelte` | PersonForm in create mode. On submit → redirect to person detail. |
| 2.6 | Person detail page | `src/routes/persons/[id]/+page.svelte`, `+page.ts` | Full biographical data display. Relationships section (grouped by type, links to related persons). Edit/Delete buttons. Relationship count badges. |
| 2.7 | Person edit page | `src/routes/persons/[id]/edit/+page.svelte` | PersonForm in edit mode, pre-populated. |
| 2.8 | Delete confirmation | — | shadcn AlertDialog on delete. Warn about cascading relationship deletion. |
| 2.9 | Unit tests | `src/lib/__tests__/persons.test.ts` | Service layer tests with mocked Supabase client. |

**Exit criteria:** Can create, view, edit, delete persons. Search works. All form validations pass.

---

## Phase 3: Relationship Management

**Goal:** Create and manage typed relationships between persons.

| # | Task | Files | Details |
|---|------|-------|---------|
| 3.1 | Relationship service layer | `src/lib/services/relationships.ts` | `getRelationships(personId?)`, `createRelationship(data)`, `updateRelationship(id, data)`, `deleteRelationship(id)`. Query returns joined person names for display. |
| 3.2 | Person picker component | `src/lib/components/relationship/PersonPicker.svelte` | Searchable dropdown/combobox (shadcn Command/Popover pattern). Search persons by name. Shows avatar + name. Excludes current person from results. |
| 3.3 | Relationship form component | `src/lib/components/relationship/RelationshipForm.svelte` | Two person pickers (A and B), relationship type select, optional start/end dates, notes. Dynamic label help: when type is `biological_parent`, show "Person A is the parent of Person B". Gender-agnostic labels (S7). |
| 3.4 | Relationship badge component | `src/lib/components/relationship/RelationshipBadge.svelte` | Color-coded badge per relationship type. Click to edit. |
| 3.5 | Relationship list component | `src/lib/components/relationship/RelationshipList.svelte` | Grouped display: Parents (biological, adoptive, step, foster, guardian), Children, Spouses/Partners. Each entry shows related person name + badge + dates. |
| 3.6 | Add relationship from person detail | `src/routes/persons/[id]/+page.svelte` | "Add relationship" button opens dialog with RelationshipForm. Pre-fills current person as one side. |
| 3.7 | Validation warnings | `src/lib/services/validation.ts` | Non-blocking warnings (S6): circular ancestry, birth after death, parent younger than child, duplicate relationships. Display as toast/alert on save. |
| 3.8 | Half-sibling inference | `src/lib/services/inference.ts` | Given a person, derive half-siblings (share exactly one biological parent) and step-siblings (parent's spouse's children). Display in person detail under inferred section. |
| 3.9 | Unit tests | `src/lib/__tests__/relationships.test.ts`, `src/lib/__tests__/inference.test.ts`, `src/lib/__tests__/validation.test.ts` | Edge cases: orphan with no rels, person with 3 marriages, adopted child with bio + adoptive parents, circular reference detection. |

**Exit criteria:** Can create all 7 relationship types. Validation warnings fire correctly. Half-sibling inference works. Person detail shows complete relationship picture.

---

## Phase 4: Tree Visualization

**Goal:** Interactive graph rendering of the full family network.

| # | Task | Files | Details |
|---|------|-------|---------|
| 4.1 | Graph transform service | `src/lib/services/graph.ts` | Transform `person[]` + `relationship[]` → Cytoscape `ElementDefinition[]`. Nodes: id, label (name), thumbnail, living/deceased flag. Edges: source, target, type, label, style class. Handle disconnected subgraphs. |
| 4.2 | Cytoscape type definitions | `src/lib/types/graph.ts` | TypeScript types for node data, edge data, layout options, style definitions. |
| 4.3 | FamilyTree component | `src/lib/components/tree/FamilyTree.svelte` | Cytoscape.js wrapper. Initialize on mount, destroy on unmount. Reactive updates when persons/relationships change. Layout: dagre (hierarchical, top-down) as default. Zoom/pan built-in. |
| 4.4 | Node styling | — | Nodes: rounded rectangle with name + birth/death years. Photo thumbnail as background image if available. Border color by gender. Deceased nodes dimmed. Orphan nodes have dashed border. |
| 4.5 | Edge styling | — | Color-coded by relationship type: biological_parent (solid blue), adoptive_parent (dashed green), step_parent (dotted orange), foster_parent (dotted purple), guardian (dotted gray), spouse (solid red, bidirectional), partner (dashed red, bidirectional). Line width varies by type. |
| 4.6 | Tree controls component | `src/lib/components/tree/TreeControls.svelte` | Zoom in/out/fit buttons. Layout toggle (dagre vs force-directed). Fullscreen toggle. Reset view. |
| 4.7 | Tree legend component | `src/lib/components/tree/TreeLegend.svelte` | Visual key mapping colors/styles to relationship types. Collapsible. |
| 4.8 | Node click → detail | — | Click node opens person detail in a side sheet (shadcn Sheet). Allows viewing/editing without leaving tree. Double-click centers + zooms to node. |
| 4.9 | Tree page | `src/routes/tree/+page.svelte`, `+page.ts` | Full-viewport tree. Load all persons + relationships. FilterPanel in sidebar. SearchBar highlights matching nodes. |
| 4.10 | Performance optimization | — | Batch Cytoscape updates. Debounce layout recalculations. For 200+ nodes: disable animations, simplify node rendering, use `textureOnViewport` for zoom. |
| 4.11 | Unit tests | `src/lib/__tests__/graph.test.ts` | Transform tests: empty graph, single node, nuclear family, disconnected subtrees, complex topology with all relationship types. |

**Exit criteria:** Tree renders 200 nodes without lag. All relationship types visually distinguishable. Zoom/pan/click work. Disconnected subtrees render correctly.

---

## Phase 5: Search & Filter

**Goal:** Find and focus on specific persons or branches in the tree.

| # | Task | Files | Details |
|---|------|-------|---------|
| 5.1 | Search bar component | `src/lib/components/layout/SearchBar.svelte` | Debounced text input. Searches first name, last name, maiden name. Results dropdown with PersonCards. Click result → navigate or highlight in tree. |
| 5.2 | Filter panel component | `src/lib/components/layout/FilterPanel.svelte` | Filters: surname (multi-select from existing surnames), living/deceased toggle, relationship type checkboxes. Filters apply to both person list and tree visualization. |
| 5.3 | Tree filtering integration | `src/routes/tree/+page.svelte` | Active filters hide non-matching nodes/edges in Cytoscape (not remove — toggle visibility). Filtered-out nodes shown as ghosts (low opacity) for context. |
| 5.4 | Person list filtering | `src/routes/persons/+page.svelte` | Same filter state applies to person list grid. URL query params sync for shareable filtered views. |
| 5.5 | Tree store | `src/lib/stores/tree-store.ts` | Centralized reactive state: `$state` for persons, relationships, search query, active filters, selected person. `$derived` for filtered sets. |

**Exit criteria:** Can search by any name field. Filters narrow both list and tree views. Tree highlights search matches.

---

## Phase 6: Polish & Quality

**Goal:** Responsive layout, keyboard nav, full quality gate pass.

| # | Task | Files | Details |
|---|------|-------|---------|
| 6.1 | Responsive layout | All route pages | Desktop: sidebar + main content. Tablet: collapsible sidebar. Mobile: stacked, tree gets full width with overlay controls. Breakpoints via Tailwind. |
| 6.2 | Nav component | `src/lib/components/layout/Nav.svelte` | App navigation: Tree (primary), Persons, Settings (future). User menu with logout. Active route indicator. |
| 6.3 | Keyboard navigation (S8) | `src/routes/tree/+page.svelte` | Arrow keys traverse connected nodes in tree. Enter opens detail sheet. Escape closes sheet/dialogs. `/` focuses search. |
| 6.4 | Empty states | Various | No persons yet → onboarding prompt. No relationships → suggestion to add. Empty search → helpful message. |
| 6.5 | Loading states | Various | Skeleton loaders for person cards, tree, detail view. Spinner for Cytoscape layout calculation. |
| 6.6 | Error handling | Various | Toast notifications for CRUD success/failure. Network error boundaries. Invalid person ID → 404 page. |
| 6.7 | Code quality pass | — | Run `pnpm format`, `pnpm lint`, `pnpm check`. Fix all errors and warnings to 0. |
| 6.8 | E2E tests | `tests/` | Playwright flows: sign up → create person → add relationship → view tree → search → filter. Auth redirect flow. CRUD operations. |
| 6.9 | Final validation | — | Full `pnpm code-quality` pipeline. Manual test with seed data (15+ persons, all relationship types). Verify 200-node performance target. |

**Exit criteria:** `pnpm code-quality` passes with 0 errors, 0 warnings. All E2E tests pass. Responsive on desktop + tablet.

---

## Dependency Graph

```
Phase 0 (Scaffold)
    │
    v
Phase 1 (Database & Auth)
    │
    ├──────────────────┐
    v                  v
Phase 2 (Person CRUD)  Phase 3* (Relationship Mgmt)
    │                  │
    │    ┌─────────────┘
    v    v
Phase 4 (Tree Visualization)
    │
    v
Phase 5 (Search & Filter)
    │
    v
Phase 6 (Polish & Quality)
```

*Phase 3 depends on Phase 2 for PersonPicker but relationship service/form can start in parallel with person UI.

---

## Agent Workstream Strategy

For **team** execution, work can be parallelized as follows:

| Workstream | Agent | Phases |
|------------|-------|--------|
| **Infrastructure** | Agent A | Phase 0 → Phase 1 (migrations, auth, types) |
| **Person UI** | Agent B | Phase 2 (after Phase 1 types are ready) |
| **Relationship Logic** | Agent C | Phase 3 services (inference, validation) can start once types exist; UI after Phase 2 PersonPicker |
| **Visualization** | Agent D | Phase 4 (after Phase 1 data model is final) — graph transform + Cytoscape can be built with mock data |
| **Integration** | Agent A | Phase 5 + Phase 6 (brings all workstreams together) |

---

## File Count Estimates

| Category | New Files | Modified Files |
|----------|-----------|----------------|
| Config / scaffold | 10 | 0 |
| Database migrations | 4 | 0 |
| Auth pages | 5 | 0 |
| Services | 6 | 0 |
| Types | 2 | 0 |
| Stores | 1 | 0 |
| Components | 15 | 0 |
| Route pages | 12 | 0 |
| Tests (unit) | 5 | 0 |
| Tests (e2e) | 2 | 0 |
| **Total** | **~62** | **0** |

---

## Test Scenarios

| # | Scenario | Type | Phase |
|---|----------|------|-------|
| T1 | Create person with all fields | Unit | 2 |
| T2 | Create person with minimal fields (first + last name only) | Unit | 2 |
| T3 | Update person preserves unchanged fields | Unit | 2 |
| T4 | Delete person cascades relationship deletion | Unit | 2 |
| T5 | Search person by first name, last name, maiden name | Unit | 2 |
| T6 | Create biological_parent relationship | Unit | 3 |
| T7 | Create adoptive_parent relationship | Unit | 3 |
| T8 | Create spouse relationship (gender-agnostic) | Unit | 3 |
| T9 | Multiple marriages: person with 3 sequential spouses | Unit | 3 |
| T10 | Child out of wedlock: child → 2 bio parents with no spousal link | Unit | 3 |
| T11 | Orphan: person with zero relationships | Unit | 3 |
| T12 | Prevent self-referencing relationship | Unit | 3 |
| T13 | Prevent duplicate relationship (same A, B, type) | Unit | 3 |
| T14 | Validation warning: parent younger than child | Unit | 3 |
| T15 | Validation warning: birth date after death date | Unit | 3 |
| T16 | Half-sibling inference: two children sharing one parent | Unit | 3 |
| T17 | Step-sibling inference: parent's spouse's children | Unit | 3 |
| T18 | Graph transform: empty dataset → empty elements | Unit | 4 |
| T19 | Graph transform: nuclear family → correct nodes + edges | Unit | 4 |
| T20 | Graph transform: disconnected subtrees preserved | Unit | 4 |
| T21 | Graph transform: all 7 relationship types → correct edge styles | Unit | 4 |
| T22 | E2E: Sign up → create person → add relationship → view in tree | E2E | 6 |
| T23 | E2E: Search person → navigate to detail | E2E | 6 |
| T24 | E2E: Filter tree by surname → verify hidden nodes | E2E | 6 |
| T25 | E2E: Auth redirect: unauthenticated user → login page | E2E | 6 |

---

## Open Decisions

1. **GEDCOM import (S4):** Deferred to post-MVP. High value but large parsing effort. Plan for it in data model (no schema changes needed) but don't build the importer.
2. **Graph layout default:** Start with dagre (hierarchical top-down) as primary. Add force-directed as toggle. User can switch.
3. **Photo upload (S3):** Defer Supabase Storage upload to post-MVP. MVP uses `photo_url` field (manual URL entry). Schema already supports it.
4. **Timeline view (S2):** Post-MVP. No schema changes needed.

---

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Cytoscape.js performance at 200+ nodes | High | Medium | Phase 4.10 optimizations: batch updates, disable animations, `textureOnViewport` |
| Complex relationship topologies break layout | High | Medium | Use dagre layout which handles DAGs well; fall back to force-directed for extreme cases |
| Svelte 5 + Cytoscape.js integration issues | Medium | Medium | Cytoscape runs in its own DOM container; Svelte just passes data. Minimal framework coupling. |
| shadcn-svelte component coverage gaps | Low | Low | bits-ui covers all needed primitives; can build custom components on top |
