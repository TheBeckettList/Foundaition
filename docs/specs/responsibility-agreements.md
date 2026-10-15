# Spec: Responsibility Agreements

## Summary
A GC, PM or owner creates a signable agreement defining who is responsible for what on a
project. They start from guided questions, free text, or an uploaded existing document.
The AI structures it into a responsibility matrix, flags gaps and conflicts, both parties
review and sign, and the **signed matrix configures the platform**: agent behavior,
permissions and risk guardrails. Amendments require signatures from both parties.

Supports the full range, from "my GC found all the subs, I just approve day to day" to a
full owner-builder with no GC.

## Why
- Turns handshake arrangements into clear, signed responsibilities.
- Gaps in responsibility are where disputes and lawsuits come from; this finds them up front.
- The signed agreement, amendments and event log form an evidentiary record for future
  warranty and dispute claims.
- GCs reuse templates across clients, which brings homeowners onto the platform.

## Feature: Attorney-authored clause library
**Every word of legal text in a Foundaition agreement is written and approved by a
licensed attorney. AI never writes legal language.** AI interprets what the parties want;
a deterministic engine assembles approved clauses like building blocks.

### Division of labor
| Who | Does |
|-----|------|
| Attorneys | Write clauses, define their variables and when they apply, approve versions |
| AI | Turns questions, free text or uploads into the responsibility matrix; proposes variable values; writes plain-language explanations (clearly labeled, never part of the signed text) |
| Assembly engine (deterministic code) | Selects clauses whose conditions match the matrix, fills typed variables, orders sections, renders the document |
| Parties | Review, negotiate the matrix and variable values, sign |

### Clauses
Each clause is a versioned block with:
- `clause_key` (e.g. `TX-LIEN-WAIVER-OWNER-PAYS`), `jurisdiction`, `section`, `sort_order`
- `body` — attorney-written text with typed variables, e.g. `{{owner_name}}`, `{{trade}}`,
  `{{retainage_days: integer}}`, `{{gc_name}}`
- `applies_when` — machine-readable conditions on the responsibility matrix, e.g.
  `paying_subs.party == 'owner'` or `sub_contracting[trade].party == 'owner'`
- `requires` / `excludes` — other clauses that must or must not appear with it
- `variables` — name, type, constraints (min/max, allowed values), source (matrix, project
  data, or party input)
- Status lifecycle: `draft` → `approved` → `deprecated`. Only `approved` versions can be assembled.
- Provenance: `authored_by`, `approved_by` (attorney, bar number, state), `approved_at`

Every jurisdiction has a set of **base clauses** that always appear (parties, relationship
to the underlying construction contract, definitions, amendment process, signatures).

### Assembly rules
1. Build the matrix (from questions, text, upload or template) and pass the gap/conflict checks.
2. Select every approved clause in the project's jurisdiction whose `applies_when` matches.
3. Validate `requires` / `excludes`. Any violation is a blocker.
4. **Coverage check:** every assignment in the matrix must be covered by at least one
   clause. If an arrangement has no approved language, it's a blocker: "There is no
   attorney-approved language for this arrangement in Texas yet." The user can request
   attorney review (below). The engine never improvises.
5. Fill variables. Values must pass type and constraint validation. AI may *propose*
   values from the input; parties confirm them in the review UI.
6. Render. The signed PDF contains only approved clause text and validated variable values.

### Custom terms
Parties can't type legal text into the agreement. If they want something the library
doesn't cover, they submit a **custom clause request**, which routes to a partner attorney
for paid review. Approved custom clauses can be added to the library (with permission),
growing coverage over time.

### Provenance in every document
The signed PDF's appendix lists each clause key, version, and approving attorney. The
agreement record stores the exact clause versions used, so a signed agreement always
reproduces exactly, even after clauses are updated.

### When the law changes
Attorneys publish a new clause version and deprecate the old one. Signed agreements keep
their original versions. Active projects using a deprecated clause get a notice, with the
option to amend to the new version (both parties sign).

### Attorney portal
Separate interface for attorneys: write and edit clauses, define variables and conditions,
preview assembled agreements against sample matrices, approve versions, and handle custom
clause requests. Attorneys are paid per review or retained per state.

### Still shown before signing
"This addendum was assembled from attorney-approved clauses. It does not replace advice
about your specific situation. You can have your own attorney review it before signing."

### Positioning
- The agreement is a **responsibility addendum** to the parties' construction contract,
  not a replacement.
- The AI never tells a party an arrangement is legal or safe. It explains consequences
  and flags risks.
- The record helps document what was agreed and what happened. Never claim it guarantees
  any legal outcome.

## Responsibility catalog (fixed, in `responsibilities` table)
Each entry: `key`, `name`, `category`, `description`, `scopable_by_trade` (bool),
`risk_notes` (plain-language consequences for whoever holds it).

**Pre-construction:** plans_and_design, permit_holder (contractor of record),
financing_coordination, sub_sourcing, sub_vetting (license and insurance),
sub_contracting, materials_selection, materials_procurement

**Construction:** scheduling, site_supervision, site_safety, inspection_scheduling,
inspection_attendance, quality_acceptance, change_order_pricing, change_order_approval,
site_cleanup_disposal, utilities_coordination

**Money:** paying_subs, paying_suppliers, lien_waiver_collection, retainage_holding,
budget_tracking

**Risk and closeout:** builders_risk_insurance, liability_insurance_verification,
workers_comp_verification, warranty, punch_list, as_built_documentation

`scopable_by_trade` items (e.g. sub_sourcing, sub_contracting, paying_subs, warranty) can
be assigned per trade, so the GC can hold electrical while the owner holds flooring.

## Data model
- `agreements` — id, project_id, version, status ('draft' | 'in_review' | 'out_for_signature'
  | 'signed' | 'superseded' | 'void'), source_type ('questions' | 'text' | 'upload' |
  'template'), source_file_path, source_text, rendered_pdf_path, rendered_pdf_sha256,
  amends_agreement_id, created_by, created_at
- `agreement_assignments` — agreement_id, responsibility_key, trade (nullable),
  phase_id (nullable), party ('owner' | 'gc' | 'pm' | 'sub' | 'shared'), member_id
  (nullable), notes, ai_confidence, source_excerpt (for uploads)
- `agreement_signatures` — agreement_id, member_id, party, signed_at, provider,
  provider_envelope_id
- `agreement_templates` — org_id, name, jurisdiction, assignments (jsonb). Templates
  store matrix presets only, never legal text.
- `attorneys` — id, user_id, name, bar_number, bar_state, verified_at
- `clauses` — id, clause_key, jurisdiction, section, sort_order
- `clause_versions` — clause_id, version, body, variables (jsonb), applies_when (jsonb),
  requires (text[]), excludes (text[]), status ('draft' | 'approved' | 'deprecated'),
  authored_by, approved_by (attorney_id), approved_at, change_note. Approved versions
  are immutable.
- `agreement_clauses` — agreement_id, clause_version_id, sort_order, variable_values (jsonb)
- `custom_clause_requests` — agreement_id, requested_by, description, status,
  assigned_attorney_id, resulting_clause_version_id

**Effective responsibilities** = assignments from the latest signed agreement for the
project, resolved most-specific-first (trade + phase, then trade, then project-wide).
Implement as a pure function `resolveResponsibility(projectId, key, trade?, phase?)` with tests.

Every status change, assignment edit, signature and amendment writes an event.

## Flow

### 1. Input
- **Guided questions:** plain-language questions ("Who found your subs?", "Whose name is
  on the permit?", "Who pays the electrician?") map directly to assignments. Start from a
  preset (Approver, Hybrid, Owner-builder) and adjust.
- **Free text:** user describes the arrangement; AI task `structureAgreement` returns
  assignments as JSON.
- **Upload:** existing contract or letter (PDF); AI task `extractAgreement` returns
  assignments with `source_excerpt` for each, so reviewers can see where it came from.
- **Template:** GC picks one of their org's saved templates.

All AI output is validated with Zod. Low-confidence assignments are marked for review,
never silently accepted.

### 2. Checks (pure functions in `lib/agreements/checks.ts`, deterministic, tested)
**Gaps:** any catalog responsibility unassigned, and any trade-scopable responsibility
unassigned for a trade in the project's scope.

**Conflicts and required pairings (examples, extend as a rule list):**
- permit_holder ≠ inspection_scheduling party → require a note on who contacts the inspector.
- Owner holds paying_subs → lien_waiver_collection must be owner; show retainage and
  homestead risk notes for the project's jurisdiction.
- Owner holds sub_contracting for a trade → warranty for that trade must be explicitly
  assigned (never defaults to GC), and liability/workers' comp verification must be assigned.
- change_order_approval must include the owner.
- Same responsibility assigned to two parties without 'shared' → conflict.

Each finding: severity ('blocker' | 'warning' | 'info'), plain-language explanation,
and which parties it affects. Blockers prevent sending for signature.

### 3. Review
- Both parties see the same matrix grouped by category, with each party's risk notes.
- Either party can propose changes; each round creates a new draft version.
- Diff view between versions.

### 4. Sign
- Assemble the agreement from approved clauses (see Assembly rules) in
  `lib/agreements/assemble.ts`, a pure function: (matrix, project data, clause library,
  variable values) → ordered clauses or blockers.
- Render the PDF with a clause provenance appendix; store its SHA-256.
- Send through the e-signature provider via `lib/esign/` interface.
- On completion webhook: store signatures, set status 'signed', supersede prior agreement,
  write events.

### 5. Execute
- On signing, recompute effective responsibilities.
- **Agents:** for each responsibility, an agent is in `drive` mode if the current user's
  party holds it (plans, drafts outreach, prompts next actions) or `inform` mode if
  another party does (explains, reminds, suggests questions to ask).
- **Permissions:** actions tied to a responsibility (e.g. marking an inspection passed,
  approving a change order) are allowed only for the party holding it.
- **Guardrails:** workflows switch on automatically, e.g. lien waiver tracking when the
  owner pays subs, insurance certificate collection when the owner contracts subs.

### 6. Amend
- Any change after signing is a new agreement with `amends_agreement_id`, requiring both
  signatures. The old agreement stays in the record as 'superseded'.

## Evidentiary record
- `events` is append-only with a hash chain (`prev_hash`, `content_hash`).
- Signed PDFs and photos are stored with SHA-256 hashes and server timestamps.
- Export: a "project record" packet (agreements and amendments, signatures, event log,
  inspections, photos with hashes) as a PDF plus a JSON bundle.

## Build order
1. Catalog seed, data model, `resolveResponsibility`, checks, with tests.
2. Clause schema, `assemble.ts` with tests, using clearly marked **placeholder clauses**
   (text: "PLACEHOLDER — NOT ATTORNEY APPROVED") so the engine can be built before an
   attorney is engaged. Placeholder clauses can never reach 'approved' status, and the
   signing step refuses any agreement containing one.
3. Guided questions + presets → matrix → checks → review UI.
4. Attorney portal (authoring, approval, preview). Engage a Texas construction attorney
   to write the real Texas clause set.
5. PDF rendering with provenance + e-sign integration + execute step (agent modes,
   permissions, guardrails).
6. Free-text structuring (with evals).
7. Upload extraction (with evals).
8. Org templates.
9. Custom clause requests.
10. Project record export.

## Acceptance criteria (first release)
- An owner and GC can create the Hybrid arrangement via guided questions, see gaps and
  conflicts, resolve blockers, sign, and see agent modes and permissions change accordingly.
- The signed document contains only approved clause text and validated variable values,
  with a provenance appendix.
- An arrangement with no covering clause is blocked, never improvised.
- An amendment after signing requires both signatures and preserves the prior version.
- Every step appears in the event log, and the hash chain verifies.

## Required tests for `assemble.ts`
- Every matrix assignment is covered by a clause, or a blocker is returned.
- `requires` / `excludes` violations are blockers.
- Invalid variable values are rejected.
- Draft, deprecated or placeholder clauses are never selected for signing.
- The same inputs always produce the identical document (deterministic).

## Evals (`evals/agreements/`)
- 20+ free-text descriptions and 10+ real or realistic uploaded documents with expected
  assignments. Track per-responsibility accuracy; gap detection must be 100% on the golden set.
