# Foundaition — Claude Code Session Plan

This is the build runbook for Claude Code cloud sessions (claude.ai/code). Each session
starts fresh, so everything a session needs lives in this repo: `CLAUDE.md` (rules),
`docs/specs/` (what to build), this file (the order), and `docs/STATUS.md` (where the
last session left off).

---

## One-time setup (≈20 minutes)

1. **Push this folder to a new private GitHub repo** named `foundaition`.
2. **Create a Supabase project** for development (free tier is fine). Copy the project
   URL, anon key and service role key. Keep a separate project for production later.
3. **Get an Anthropic API key** from the Claude Console. This is for the *app's* AI
   features (quote extraction, agreement structuring), separate from your Claude plan.
4. **Open claude.ai/code**, connect GitHub, and select the `foundaition` repo.
5. **Configure the cloud environment** for the repo:
   - Environment variables: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`,
     `SUPABASE_SERVICE_ROLE_KEY`, `ANTHROPIC_API_KEY`
   - Network access: allow package registries (npm) at minimum. Add your Supabase
     project's domain if you want sessions to apply migrations to the dev project.
   - Setup script: `npm ci` (once `package.json` exists after Session 0)

Never paste keys into prompts or commit them. They live only in the environment settings
and in your local `.env.local`.

---

## Rules for every session

Paste nothing but the session prompt. These rules are already in the prompts, but they
are the contract:

- **Start in plan mode.** Review the plan before any code is written.
- **One session = one branch = one pull request.** You review and merge each PR.
- **Tests must pass before the PR.** `npm run typecheck && npm run lint && npm test`.
- **Migrations are tested in the sandbox** against PGlite (in-process Postgres), so no
  session needs a live database to prove its schema works.
- **End by updating `docs/STATUS.md`**: what was built, what's left, decisions made,
  anything the next session must know.
- **If the session hits something the specs don't cover,** it stops and writes the
  question in STATUS.md under "Open questions" rather than guessing.

---

## Session 0 — Repo foundation

**Goal:** a working skeleton every later session builds on.

```
Read CLAUDE.md, docs/SESSIONS.md and both specs in docs/specs/. This is Session 0.
Start in plan mode and show me the plan before writing code.

Build the repo foundation only:
1. Scaffold Next.js (App Router) + TypeScript strict + Tailwind + ESLint.
2. Install @supabase/supabase-js, @supabase/ssr, @anthropic-ai/sdk, zod, vitest,
   @electric-sql/pglite.
3. Scripts: typecheck, lint, test, test:db. Add .env.example matching CLAUDE.md.
4. Migration test harness: tests/db/ helper that boots PGlite, creates a stub `auth`
   schema (auth.users table and an auth.uid() function settable per test), applies every
   file in supabase/migrations/ in order, and exposes helpers to run SQL as a given user.
5. Migration 0001_events.sql: the `events` table from CLAUDE.md with content_hash and
   prev_hash computed by trigger (hash chain per project), and triggers that reject
   UPDATE and DELETE. Tests: insert works, chain links correctly, update/delete fail,
   a tampered row breaks verification. Add lib/events/verify-chain.ts with tests.
6. lib/ai/gateway.ts: a typed task registry (task name → input schema, output schema,
   provider config). Implement one provider (Anthropic) and a fake provider for tests.
   No real tasks yet beyond a trivial `ping` task covered by a test using the fake.
7. GitHub Actions CI running typecheck, lint, test, test:db on every PR.
8. Create docs/STATUS.md and fill it in.

Do not build auth UI, features or pages beyond the default home page.
Work on a branch, open a PR when everything passes.
```

**Done when:** CI is green and the event chain tests pass, including the tamper test.

---

## Session 1 — Identity, projects and access

```
Read CLAUDE.md, docs/STATUS.md and docs/SESSIONS.md. This is Session 1. Plan mode first.

Build:
1. Migrations for organizations, profiles, homes (fields from building-model spec),
   projects (with home_id), project_members (party enum from CLAUDE.md).
2. Row Level Security on every table: members see only their projects; only owners and
   GCs can add members. Write RLS tests in the PGlite harness for each party.
3. Every insert/update on these tables writes an event in the same transaction.
4. Supabase Auth email sign-in (magic link) using @supabase/ssr, a protected /projects
   page listing the user's projects, and a create-project form (home address, project
   type 'addition', start date).
5. Seed: one home, one 'addition' project, an owner and a GC member.

Keep UI plain. Update STATUS.md, open a PR when CI passes.
```

**Done when:** two test users see only their own projects, verified by RLS tests.

---

## Session 2 — Building model core: spaces and components

```
Read CLAUDE.md, docs/STATUS.md, docs/specs/building-model.md. Session 2. Plan mode first.

Implement building-model build order step 1:
1. Migrations: spaces (hierarchical), the fixed category list, components (tree via
   parent_component_id, lifecycle status, replaced_by_id, is_placeholder,
   installed_in_project_id). Components belong to the home.
2. lib/building-model/: pure functions for tree operations (move, refine placeholder,
   remove, replace) with the required tests from the spec. Nothing is ever deleted.
3. Every change writes an event.
4. RLS consistent with Session 1.
5. A simple home page per home: space tree and component tree, add/move/replace actions.

Update STATUS.md, open a PR.
```

---

## Session 3 — Products, facts and relationships

```
Read CLAUDE.md, docs/STATUS.md, docs/specs/building-model.md. Session 3. Plan mode first.

Implement building-model build order steps 2–4:
1. products, product_identifiers (unique), product_documents.
2. Scan flow, found path only: enter or scan a barcode (use the browser BarcodeDetector
   API with manual entry fallback), look up the product, create a component, pick a space.
3. facts table with source and confidence enums from the spec, superseded_by_id, and
   lib/building-model/resolve-value.ts (component fact → product spec fallback) with tests.
4. component_relationships with the relation enum from the spec.
5. Show each component's resolved values with their source.

Seed ~20 common residential products (water heater, faucets, panel, furnace) with
placeholder specs clearly marked as seed data. Update STATUS.md, open a PR.
```

---

## Session 4 — Schedule with inspection gates

```
Read CLAUDE.md, docs/STATUS.md. Session 4. Plan mode first.

1. Migrations: phases and inspections per CLAUDE.md.
2. lib/schedule/: pure functions to instantiate the addition template from CLAUDE.md,
   compute dates on working days (start the next working day after all dependencies end),
   and mark phases blocked behind unpassed inspection gates. Thorough tests, including
   parallel phases (6, 7, 8) and the final phase depending on 12 and 15.
3. Editing a duration or start date recomputes downstream dates.
4. Marking an inspection passed or failed writes an event. For now any project member
   may do it; Session 6 restricts this by responsibility.
5. Project schedule page: table of phases, dates, gates, blocked status.

Update STATUS.md, open a PR.
```

---

## Session 5 — Responsibility matrix and checks

```
Read CLAUDE.md, docs/STATUS.md, docs/specs/responsibility-agreements.md. Session 5.
Plan mode first.

Implement agreement build order step 1:
1. Seed the responsibility catalog exactly as listed in the spec, with risk_notes.
2. Migrations: agreements, agreement_assignments, agreement_signatures,
   agreement_templates (matrix presets only).
3. lib/agreements/resolve.ts: resolveResponsibility (trade+phase → trade → project-wide).
4. lib/agreements/checks.ts: gap detection and every conflict/pairing rule in the spec,
   as a rule list, returning findings with severity. Tests for each rule.
5. Presets as data: Approver, Hybrid, Owner-builder.

No UI yet. Update STATUS.md, open a PR.
```

---

## Session 6 — Clause library and assembly engine

```
Read CLAUDE.md (principle 7), docs/STATUS.md, docs/specs/responsibility-agreements.md.
Session 6. Plan mode first.

Implement agreement build order step 2:
1. Migrations: attorneys, clauses, clause_versions (approved versions immutable via
   trigger), agreement_clauses, custom_clause_requests.
2. A small declarative condition language for applies_when, evaluated in
   lib/agreements/conditions.ts, with tests.
3. lib/agreements/assemble.ts as specified, returning ordered clauses or blockers.
   Implement every test in "Required tests for assemble.ts".
4. Seed Texas PLACEHOLDER clauses covering the Hybrid preset. Every body begins with
   "PLACEHOLDER — NOT ATTORNEY APPROVED". Add a database constraint and a test proving
   placeholder clauses can never reach 'approved' status.
5. Enforce responsibility-based permissions for marking inspections (from Session 4).

Never write realistic legal language, even as a placeholder. Update STATUS.md, open a PR.
```

---

## Session 7 — Agreement builder UI

```
Read CLAUDE.md, docs/STATUS.md, docs/specs/responsibility-agreements.md. Session 7.
Plan mode first.

Implement agreement build order step 3:
1. Guided questions flow: start from a preset, plain-language questions that set
   assignments, per-trade overrides for trade-scopable responsibilities.
2. Review screen: matrix grouped by category, findings from checks.ts with blockers
   preventing "send for signature", each party's risk notes.
3. Draft versions and a diff view between versions.
4. Preview of the assembled document (placeholder clauses clearly watermarked).
   Signing stays disabled while any placeholder clause is present.

Update STATUS.md, open a PR.
```

**Done when:** the Hybrid arrangement can be built through questions, shows gaps and
conflicts, and previews an assembled (placeholder) addendum.

---

## Session 8 — Quote extraction and comparison

```
Read CLAUDE.md, docs/STATUS.md. Session 8. Plan mode first.

1. Migrations: quotes, quote_line_items, quote_exclusions. Quote PDFs in Supabase
   Storage with sha256 recorded.
2. Gateway task `extractQuote` per the "Quote extraction" section of CLAUDE.md, with Zod
   output schema and per-trade category lists. Unit tests use the fake provider.
3. evals/quotes/: harness that runs extractQuote on golden PDFs and scores line items,
   totals and exclusions. Add 3 synthetic sample quotes you generate as PDFs; I will add
   real ones later. Evals run on demand (npm run eval:quotes), not in CI.
4. lib/compare-quotes.ts with tests.
5. Upload page and side-by-side comparison page.

Update STATUS.md, open a PR.
```

---

## Session 9 — Photo log

```
Read CLAUDE.md, docs/STATUS.md. Session 9. Plan mode first.

1. photos table per CLAUDE.md; upload from phone camera via file input; compute sha256
   server-side; read EXIF taken_at when present.
2. Tag by phase, space and component. Every photo writes an event.
3. Facts can reference a photo as evidence (evidence_type 'photo'); marking a fact
   build_verified requires a photo or a passed inspection.
4. Mobile-friendly timeline view per project, filterable by phase and space.

Update STATUS.md, open a PR.
```

---

## Session 10 — Categories and pricing

```
Read CLAUDE.md, docs/STATUS.md, docs/specs/building-model.md. Session 10. Plan mode first.

Implement building-model build order step 5:
1. home_subscriptions and a pricing config table (size tiers, category prices, scale
   factors). Placeholder amounts only.
2. lib/pricing/compute-price.ts with the tests listed in the spec.
3. Price preview on the home page that updates when categories or scale factors change.

No payment processing. Update STATUS.md, open a PR.
```

---

## After Session 10

At this point the foundation phase in CLAUDE.md is built. Next sessions come from the
remaining build orders in the specs, in this sequence:

1. Agreements: PDF rendering with provenance + e-sign interface + execute step
2. Agreements: free-text structuring, then upload extraction (each with evals)
3. Building model: draft products from photos (`extractProductSpecs`, with evals)
4. Building model: materials, compatibility, failure modes seed data
5. Building model: rules engine with schedule and recall rules, findings
6. Sensors: streams, readings store, Home Assistant integration
7. Attorney portal (engage the Texas construction attorney before this)

Write each as a session prompt in this file before starting it, following the same shape:
read the docs, plan mode, numbered scope, tests, STATUS.md, PR.

---

## When a session goes wrong

- **It drifts outside scope:** stop it, point to the session prompt and CLAUDE.md.
- **Tests are flaky or it wants to skip them:** don't merge. Ask it to fix the test, not
  delete it.
- **It edits a spec to match its code:** reject. Specs change only when you decide.
- **A schema decision looks wrong after merging:** fix it with a new migration, never by
  editing an applied one.
