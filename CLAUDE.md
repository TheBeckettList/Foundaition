# Foundaition — CLAUDE.md

## What this is
Foundaition helps homeowners take on any level of project management for their own
residential construction, from approving decisions while a GC runs everything to acting
as a full owner-builder. Long term, every house built on Foundaition leaves construction
with a verified as-built model that becomes a living digital twin, with agents per system
and sensor data over the life of the home.

Build the foundation properly. Ship features incrementally on top of it.

## Architectural principles (non-negotiable)
1. **One building model is the source of truth.** Homes, spaces, components, products,
   relationships and facts. Every feature reads and writes it. Homes outlive projects.
   See `docs/specs/building-model.md`.
2. **Every component has an identity.** A water heater is an object with an ID, specs,
   installer, install date and history — never just a line item.
3. **History is append-only.** Changes are recorded as events in `events`. Nothing that
   happened is ever overwritten or deleted. This is the audit trail for warranty and
   dispute claims and the timeline for the digital twin.
4. **The signed agreement is the configuration.** Who does what on a project is defined
   by a signed responsibility agreement. Agent behavior, permissions and risk guardrails
   are derived from it. Never hardcode a "mode". See `docs/specs/responsibility-agreements.md`.
5. **Models are behind a gateway.** Code calls tasks (`extractQuote`, `structureAgreement`),
   never a specific model. `lib/ai/gateway.ts` routes tasks to providers.
6. **Every AI task has evals.** Golden inputs with expected outputs in `evals/`. A change
   to a prompt or model must not lower eval scores.
7. **AI never writes legal text.** Agreements are assembled deterministically from
   attorney-approved, versioned clauses. AI may build the responsibility matrix, propose
   variable values, and write labeled plain-language explanations. Nothing AI-generated
   ever appears in a signed document. If no approved clause covers a case, block it.

## Current phase: Foundation
In scope now:
- Auth and multi-party projects (owner, GC, PM, sub, viewer)
- Core schema, including the event log and responsibility catalog
- Building model core: homes, spaces, components (tree), products and barcode lookup,
  facts with provenance, relationships (build order steps 1–5 in the building-model spec)
- Responsibility agreement builder (see spec)
- Schedule with inspection gates (addition template)
- Quote extraction and comparison
- Photo log (every photo is an event)

Out of scope for now: sensors and readings, rules and findings, simulation, digital twin
UI, plan reading, synthetic plan factory, 3D, per-trade agent swarm, payments processing,
native mobile app.

## Stack
- Next.js (App Router) + TypeScript, strict mode
- Supabase: Postgres, Auth, Storage; Row Level Security ON for all tables
- Anthropic TypeScript SDK, called only through `lib/ai/gateway.ts`, server-side only
- Zod for validating all AI output before it touches the database
- Vitest for unit tests; Tailwind for layout
- E-signature via a third-party provider behind `lib/esign/` (provider TBD — keep an interface)

## Environment variables (`.env.local`, never commit)
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=   # server-side only
ANTHROPIC_API_KEY=           # server-side only
```

## Session workflow
- Read `docs/STATUS.md` first, then the session prompt from `docs/SESSIONS.md`.
- Plan before coding. Stay inside the session's scope.
- Never edit `docs/specs/` or this file to fit your code; raise an open question instead.
- Never edit an applied migration; add a new one.
- Before opening a PR: `npm run typecheck && npm run lint && npm test && npm run test:db`.
- End every session by updating `docs/STATUS.md`.

## Conventions
- DB and AI calls only in server actions or route handlers.
- Migrations in `supabase/migrations/` (plain SQL, ordered). Seeds in `supabase/seed.sql`.
- Pure logic in `lib/` with unit tests: schedule math, quote comparison, agreement
  gap/conflict checks, effective-responsibility resolution. UI stays thin.
- Any write that changes project state also writes an event, in the same transaction.
- Prefer simple and readable over clever.

## Data model (core)
- `organizations` — GC/builder companies
- `users` — via Supabase Auth; `profiles` for display info
- `homes`, `spaces` — see building-model spec
- `projects` — id, home_id, name, project_type, start_date
- `project_members` — project_id, user_id, party ('owner' | 'gc' | 'pm' | 'sub' | 'viewer'), org_id
- `products`, `product_identifiers`, `components`, `component_relationships`, `facts` —
  see building-model spec. Components belong to the home, not the project.
- `phases`, `inspections` — as in the addition template below
- `quotes`, `quote_line_items`, `quote_exclusions`
- `photos` — id, project_id, storage_path, sha256, taken_at (EXIF), uploaded_at, phase_id, component_id
- `responsibilities` — fixed catalog (see spec)
- `agreements`, `agreement_assignments`, `agreement_signatures`, `agreement_templates` (see spec)
- `events` — append-only: id, project_id, occurred_at, recorded_at, actor_user_id, type,
  subject_type, subject_id, payload (jsonb), content_hash, prev_hash.
  No UPDATE or DELETE permitted (enforce with RLS and a trigger).

## Domain rules
- **Inspection gates block progress.** Phases depending on a gated phase show as blocked
  until the inspection is 'passed'.
- Schedules use working days (Mon–Fri). No holidays or weather yet.
- Never present anything as confirmation of code compliance or as legal advice. Code
  items say to confirm with the local inspector; agreements say to have an attorney review.
- Agreements are **responsibility addenda** to a construction contract, not replacements.

## Addition phase template (placeholder durations, working days)
| # | Phase | Days | Depends on | Inspection after |
|---|-------|------|-----------|------------------|
| 1 | Site prep & excavation | 3 | — | — |
| 2 | Footings | 2 | 1 | Footing |
| 3 | Foundation / slab | 5 | 2 | Foundation |
| 4 | Framing (floor, walls, roof) | 10 | 3 | — |
| 5 | Roofing & dry-in (windows, doors, wrap) | 5 | 4 | — |
| 6 | Rough plumbing | 3 | 5 | Rough plumbing |
| 7 | Rough electrical | 3 | 5 | Rough electrical |
| 8 | Rough HVAC | 2 | 5 | Rough mechanical |
| 9 | Framing inspection | 1 | 6, 7, 8 | Framing |
| 10 | Insulation | 2 | 9 | Insulation |
| 11 | Drywall | 7 | 10 | — |
| 12 | Exterior siding | 5 | 5 | — |
| 13 | Interior trim & paint | 7 | 11 | — |
| 14 | Flooring | 4 | 13 | — |
| 15 | Finish plumbing, electrical, HVAC | 3 | 14 | — |
| 16 | Final | 1 | 12, 15 | Final |

A phase starts on the next working day after all its dependencies end.

## Quote extraction
- Send the PDF as a base64 `document` block; instruct the model to return only JSON.
- Validate with Zod. On failure, show the raw response and an error; save nothing partial.
- Schema: `contractor_name`, `trade`, `total`, `payment_terms`, `valid_until`,
  `line_items[]` (`category`, `description`, `quantity`, `unit`, `unit_price`, `line_total`),
  `exclusions[]` (include "by owner", "not included", allowances).
- `category` normalized to a fixed list per trade so quotes can be compared.
- Comparison (`lib/compare-quotes.ts`): align by category, flag missing categories and
  exclusions, show totals and differences. Never declare a winner.
