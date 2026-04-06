# PRD: mx-family-tree

> **Status:** Draft
> **Created:** 2026-04-05
> **Author:** Joshua Tune (via Claude Code)
> **Stack:** SvelteKit 5, Svelte 5, Tailwind CSS 4, shadcn-svelte, Supabase, TypeScript 5.9

---

## Problem Statement

Family history records stored on paper are fragile, hard to share, and impossible to visualize as a connected network. Existing genealogy tools are either bloated desktop apps, locked behind subscriptions, or enforce rigid tree hierarchies that break down with real-world family complexity (multiple marriages, adoption, unknown parentage, non-traditional structures).

## Solution

A lightweight, self-hosted web app that lets a user digitize paper-based family records into a flexible graph data model and visualize the full family network interactively. The app treats every person as a node and every relationship as a typed, directed edge — supporting any topology without forcing a strict tree hierarchy.

## Target User

A single household archivist (the app owner) who has accumulated paper-based family records and wants to centralize, digitize, and explore them visually. No multi-tenant or collaborative editing in MVP.

---

## User Roles & Expectations

### Role A — Owner (Authenticated User)

The primary (and only) user. Has full CRUD access to all data. Authenticated via Supabase Auth.

### Role C — Viewer (Unauthenticated / Share Link)

A future consideration (post-MVP). Could view a read-only snapshot of the tree via a share link. Not in scope for MVP but the data model should not preclude it.

---

## Requirements

### Must Have

| ID | Requirement |
|----|-------------|
| **M1** | **Person CRUD** — Create, read, update, and delete person records with fields: first name, middle name(s), last name, maiden name, suffix, gender, birth date, birth place, death date, death place, photo URL, and freeform notes. |
| **M2** | **Relationship CRUD** — Create, read, update, and delete directional relationships between any two persons. Supported relationship types: `biological_parent`, `adoptive_parent`, `step_parent`, `foster_parent`, `guardian`, `spouse`, `partner`. Each relationship may carry optional metadata: start date, end date, and notes. |
| **M3** | **Multiple marriages / partners** — A person can have multiple `spouse` or `partner` relationships (sequential or overlapping). The UI must clearly display all partnerships for a given person. |
| **M4** | **Children out of wedlock** — A child can be linked to one or both biological parents without requiring a spousal relationship between those parents. |
| **M5** | **Adoption modeling** — Distinguish biological vs. adoptive parentage. A child may have both biological parent(s) and adoptive parent(s) simultaneously. The UI must visually differentiate these link types. |
| **M6** | **Orphan / unknown parentage** — Person records with zero parent relationships are valid. The tree visualization must render disconnected nodes / subtrees without errors. |
| **M7** | **Interactive tree visualization** — Render the full family network as a zoomable, pannable graph/tree. Nodes represent persons (showing name + thumbnail). Edges represent relationships, color- or style-coded by type. Must handle disconnected subgraphs. |
| **M8** | **Person detail view** — Clicking a node opens a detail panel/page showing all biographical data, all relationships (grouped by type), and navigation to related persons. |
| **M9** | **Search & filter** — Search persons by name (first, last, maiden). Filter the visible tree by surname, living/deceased status, or relationship type. |
| **M10** | **Supabase Auth** — Owner authenticates via email/password (Supabase Auth). All data is protected by RLS policies scoped to the authenticated user. |
| **M11** | **Responsive layout** — Usable on desktop (primary) and tablet. Mobile is secondary but must not be broken. |
| **M12** | **Graph data model** — Use a `person` table + `relationship` edge table (not a hierarchical parent_id column). This is the architectural foundation for all relationship flexibility. |

### Should Have

| ID | Requirement |
|----|-------------|
| **S1** | **Half-sibling / step-sibling inference** — The app should automatically infer and display half-sibling and step-sibling relationships based on shared parents and step-parent links, without requiring explicit sibling edges. |
| **S2** | **Timeline view** — A horizontal timeline showing family events (births, deaths, marriages) in chronological order, filterable by branch. |
| **S3** | **Photo upload** — Upload person photos to Supabase Storage (not just URL). Show thumbnails in tree nodes and full-size in detail view. |
| **S4** | **CSV / GEDCOM import** — Import existing records from CSV or GEDCOM 5.5 format to bootstrap the tree from digitized spreadsheets or exports from other genealogy tools. |
| **S5** | **Data export** — Export the full dataset as JSON or CSV for backup and portability. |
| **S6** | **Relationship validation warnings** — Warn (but do not block) on logically suspicious data: e.g., a person listed as their own ancestor, birth date after death date, parent younger than child. |
| **S7** | **Same-sex couples** — Spouse/partner relationships are gender-agnostic. The UI should not assume heteronormative defaults (e.g., "husband/wife" labels should adapt or use neutral terms). |
| **S8** | **Keyboard navigation** — Arrow keys to traverse the tree, Enter to open detail, Escape to close panels. |

### Could Have (Post-MVP)

| ID | Requirement |
|----|-------------|
| **C1** | **Shareable read-only link** — Generate a public or password-protected link for family members to view (but not edit) the tree. |
| **C2** | **Collaborative editing** — Invite family members to contribute records with approval workflow. |
| **C3** | **AI-assisted OCR** — Upload photos of paper records and auto-extract names, dates, and relationships. |
| **C4** | **DNA / ethnicity integration** — Link to 23andMe / AncestryDNA results for ethnicity overlay on the tree. |
| **C5** | **Print-friendly tree export** — Generate a PDF or SVG of the tree suitable for printing at large format. |
| **C6** | **Realtime collaboration** — Supabase Realtime for live multi-user editing (depends on C2). |
| **C7** | **Story / narrative attachments** — Attach longer narrative text, audio, or video to a person record for oral history preservation. |

### Won't Have (Explicitly Out of Scope)

| ID | Requirement |
|----|-------------|
| **W1** | Multi-tenant / SaaS — This is a single-owner app, not a platform. |
| **W2** | Ancestry.com integration — No API connections to third-party genealogy services in MVP. |
| **W3** | Mobile-native app — Web only. |
| **W4** | Historical record search — No integration with census, immigration, or church record databases. |

---

## Data Model (Conceptual)

### `person`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `user_id` | uuid | FK to auth.users, RLS owner |
| `first_name` | text | Required |
| `middle_names` | text | Optional, space-separated |
| `last_name` | text | Required |
| `maiden_name` | text | Optional |
| `suffix` | text | Jr., Sr., III, etc. |
| `gender` | text | male, female, other, unknown |
| `birth_date` | date | Nullable (unknown) |
| `birth_place` | text | Nullable |
| `death_date` | date | Nullable (still living or unknown) |
| `death_place` | text | Nullable |
| `photo_url` | text | Nullable |
| `notes` | text | Freeform |
| `created_at` | timestamptz | Default now() |
| `updated_at` | timestamptz | Auto-updated |

### `relationship`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `user_id` | uuid | FK to auth.users, RLS owner |
| `person_a_id` | uuid | FK to person — the "from" node |
| `person_b_id` | uuid | FK to person — the "to" node |
| `type` | enum | `biological_parent`, `adoptive_parent`, `step_parent`, `foster_parent`, `guardian`, `spouse`, `partner` |
| `start_date` | date | Nullable (e.g., marriage date) |
| `end_date` | date | Nullable (e.g., divorce date, death) |
| `notes` | text | Freeform |
| `created_at` | timestamptz | Default now() |

**Edge semantics:**
- For parent types: `person_a` is the **parent**, `person_b` is the **child**.
- For spouse/partner: direction is arbitrary; the relationship is logically bidirectional.
- A unique constraint on `(person_a_id, person_b_id, type)` prevents duplicate edges of the same type.

---

## Technical Constraints

| Constraint | Detail |
|-----------|--------|
| Stack | SvelteKit 5, Svelte 5, Tailwind CSS 4, shadcn-svelte (bits-ui), TypeScript 5.9, Vite 7 |
| Database | Supabase Postgres with RLS |
| Auth | Supabase Auth (email/password) |
| Package manager | pnpm |
| Ports | Dev: 5176, Preview: 4176, Supabase block: 54821-54829 |
| Quality gate | `pnpm code-quality` must pass with 0 errors and 0 warnings before any commit |
| TypeScript | Strict mode, no `any`, no type casting, `unknown` + narrowing |
| Svelte | Runes only (`$state`, `$derived`, `$effect`) — no legacy lifecycle hooks |
| Navigation | All paths via `resolve()` from `$app/paths` |
| Commits | Conventional commits required |

---

## Graph Visualization Approach

The family network is a **directed multigraph** (multiple typed edges between the same pair of nodes). Recommended library evaluation for MVP:

1. **D3.js (d3-force)** — Maximum flexibility, well-suited for force-directed layouts of arbitrary graphs. Requires more manual work for zoom/pan/interaction.
2. **Cytoscape.js** — Purpose-built for graph visualization. Supports compound nodes, multiple edge types, built-in layouts (dagre, cola, breadthfirst). Good fit for genealogy graphs.
3. **vis-network** — Simpler API, decent for medium-sized graphs. Less layout control than Cytoscape.

**Recommendation:** Cytoscape.js for MVP — it handles the multigraph topology natively, has built-in hierarchical layouts suitable for family trees, and supports edge styling by type.

---

## Success Criteria

1. Owner can digitize 100+ person records with all relationship types listed in M2.
2. The tree visualization renders a 200-node graph without performance degradation.
3. Disconnected subtrees, orphan nodes, and complex topologies (multiple marriages, adoption chains) render correctly.
4. All data is persisted in Supabase and survives page refresh / relogin.
5. `pnpm code-quality` passes with 0 errors and 0 warnings.

---

## Open Questions

1. Should the MVP support GEDCOM import (S4) or defer to post-MVP? (Large format to parse but high value for bootstrapping.)
2. Preferred graph layout: hierarchical (top-down pedigree) or force-directed (organic network)? Could offer both.
3. Should "living" persons have restricted visibility by default (privacy concern for shared views in C1)?
