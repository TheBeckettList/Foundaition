# Spec: Building Model

## Summary
The building model is the single source of truth for a home. It starts during a
construction project, becomes the verified as-built record, and continues as the living
digital twin for the life of the house. Every feature reads and writes it: schedules,
quotes, agreements, photos, sensors, findings and agents.

Detail is variable. A home can be recorded coarsely ("HVAC system") or down to every
scanned item, and detail can deepen at any time without restructuring.

## Core decisions
1. **Homes outlive projects.** A `home` persists for decades; `projects` (the original
   build, later renovations) belong to a home. Components belong to the home, and record
   which project installed them.
2. **Products vs. components.** A *product* is a shared catalog entry (a model of water
   heater). A *component* is one installed item in one home.
3. **Detail is a tree.** Components nest to any depth. Placeholders can be refined later.
4. **Relationships drive predictions.** Contact, location, supply and power relationships
   between components are first-class data.
5. **Every fact has a source and a confidence.**
6. **Sensors are components.** Their readings live in a time-series store, not the model.
7. **Findings come from versioned, human-authored rules.** AI explains findings and may
   draft rule proposals for human review. AI never invents a rule that fires in production.
8. **Price = home size + categories.** Depth within a category is *derived* from installed
   sensors and available simulation inputs, never purchased per object.
9. **Agents run on demand.** No always-on agent per object. An agent is assembled with the
   relevant context when something happens.
10. **All changes write events** (append-only, hash-chained; see CLAUDE.md).

## Entities

### Homes and spaces
- `homes` — id, address, jurisdiction, climate_zone, year_built, conditioned_sqft,
  total_sqft, created_at
- `projects` — id, home_id, name, project_type, start_date, … (see CLAUDE.md)
- `spaces` — id, home_id, parent_space_id, name, space_type ('building' | 'floor' |
  'room' | 'attic' | 'crawlspace' | 'garage' | 'wall_cavity' | 'exterior_zone' | …),
  conditioned (bool), area_sqft
  Spaces are hierarchical: home → building → floor → room. Wall cavities and exterior
  zones are spaces too, so sensors and components can be located in them.

### Categories
Fixed list: `core_structure`, `envelope_roof`, `plumbing`, `electrical`, `hvac`,
`appliances`, `water_systems`, `outdoor`, `energy`, `specialty`.
`core_structure`, `envelope_roof`, `plumbing` and `electrical` form the Core bundle.

### Products (shared catalog, not per home)
- `products` — id, manufacturer, model_name, model_number, product_type, category,
  specs (jsonb: operating ranges, dimensions, ratings), expected_life_years,
  maintenance (jsonb: task, interval), warranty (jsonb), status ('draft' | 'verified')
- `product_identifiers` — product_id, scheme ('gtin' | 'upc' | 'ean' | 'mpn'), value (unique)
- `product_documents` — product_id, doc_type ('spec_sheet' | 'manual' | 'install_guide'
  | 'warranty'), storage_path, sha256
- `product_recalls` — product_id, source, recall_id, published_at, summary, url
- `product_materials` — product_id, material_id, role (e.g. 'body', 'seal', 'finish')
- `product_fleet_stats` — product_id, installs_observed, failures_observed,
  failure_mode_counts (jsonb), computed_at. Aggregated only from homes that consented.

Specs extracted from documents by the AI task `extractProductSpecs` are stored with
source `ai_extracted` until verified by a person.

### Materials and failure modes (the building blocks)
- `materials` — id, name, material_class ('metal' | 'polymer' | 'stone' | 'wood' |
  'ceramic' | 'composite' | …), properties (jsonb)
- `material_compatibility` — material_a_id, material_b_id, interaction ('galvanic' |
  'chemical' | 'thermal' | …), severity, conditions (jsonb, e.g. moisture present), notes
- `failure_modes` — id, key, name, applies_to ('material' | 'product_type' | 'product'),
  target_id, drivers (jsonb: conditions that accelerate it, e.g. humidity > 70%),
  indicators (jsonb: observable signals), typical_onset (jsonb), consequence, severity

Custom or rare objects are modeled by composing materials and their failure modes, then
adding anything unique. No object is modeled from scratch.

### Components (installed items)
- `components` — id, home_id, parent_component_id, product_id (nullable),
  component_type, category, name, space_id, status ('planned' | 'installed' | 'removed'
  | 'replaced'), replaced_by_id, serial_number, installed_at, installed_by_member_id,
  installed_in_project_id, is_placeholder (bool)
- `component_materials` — component_id, material_id, role, notes. Overrides or adds to
  the product's materials; required for custom objects without a product.

Systems are components too (e.g. a component of type `hvac_system` whose children are
the furnace, condenser, ducts and thermostat). The tree is `parent_component_id`.

Lifecycle: nothing is deleted. Removal sets status and writes an event; replacement links
old to new via `replaced_by_id`.

### Relationships
- `component_relationships` — id, from_id, from_type ('component' | 'space'), to_id,
  to_type, relation, attributes (jsonb), source, confidence, valid_from, valid_to

Relations (extendable): `in_contact_with`, `connected_to`, `located_above`,
`powered_by` (circuit), `fed_by` (supply), `drains_to`, `serves` (e.g. HVAC zone serves
space), `observes` (sensor → component or space).

Containment is the component tree, not a relationship.

### Facts and provenance
- `facts` — id, subject_type ('component' | 'space' | 'relationship' | 'home'),
  subject_id, key, value (jsonb), unit, source, confidence (0–1), evidence_type,
  evidence_id (photo, document, reading, scan), recorded_by, recorded_at, superseded_by_id

Sources: `build_verified` (confirmed during construction, ideally with photo),
`scanned`, `catalog`, `spec_sheet`, `user_entered`, `ai_extracted`, `ai_inferred`,
`simulated`.

Component values resolve as: most recent non-superseded fact on the component, falling
back to the product's specs. Every value shown in the UI can display its source.

### Sensors and readings
- A sensor is a component whose product has `capabilities` in its specs (metrics it
  measures: temperature, humidity, moisture, leak, power, current, flow, pressure,
  vibration, …).
- `observes` relationships link each sensor to what it measures.
- `sensor_streams` — id, sensor_component_id, metric, unit, sampling_interval_s,
  integration ('home_assistant' | 'vendor_api' | 'manual' | …), external_ref, active
- `readings` — stream_id, observed_at, value. Stored in a dedicated time-series store
  (partitioned by time), never in `events`. Downsampled rollups (hourly, daily) kept
  long term; raw retention period configurable.

### Simulation inputs
Used when no sensor covers something:
- Local weather history and forecast for the home's location
- Climate zone
- Component age and product specs
- Typical usage profiles (by product type and household size)
- Material behavior from `failure_modes` and `material_compatibility`

- `simulated_conditions` — home_id, space_id, metric, method, estimate (jsonb: expected
  value and range), confidence, computed_at

Anything derived from simulation carries basis `simulated` and lower confidence than
sensor-based results.

### Rules and findings
- `rules` — id, rule_key, category
- `rule_versions` — rule_id, version, applies_to (jsonb: product types, materials,
  relationships), conditions (jsonb, a small declarative condition language evaluated
  in code), basis_required ('schedule' | 'simulated' | 'sensor' | 'fleet'), severity,
  explanation_template, failure_mode_id, status ('draft' | 'approved' | 'deprecated'),
  authored_by, reviewed_by, approved_at. Approved versions are immutable.
- `findings` — id, home_id, rule_version_id, subject_type, subject_id, severity,
  basis ('schedule' | 'simulated' | 'sensor' | 'fleet'), confidence, evidence (jsonb:
  fact ids, reading windows, rule version, product document), status ('open' |
  'acknowledged' | 'resolved' | 'dismissed'), explanation (AI-written, labeled as such),
  created_at, resolved_at

Rule families:
- **Schedule:** maintenance intervals, warranty expiry, end of expected life.
- **Recall:** component's product matches a recall.
- **Operating range:** readings (or simulated conditions) outside the product's rated range.
- **Compatibility:** `in_contact_with` components whose materials interact, amplified by
  measured or simulated conditions (e.g. galvanic pair + humidity).
- **Fleet:** failure likelihood from `product_fleet_stats` given age and conditions.

The rule engine is deterministic and tested. Findings are deduplicated per rule and
subject while open.

### Depth (derived, never stored as a purchase)
`computeDepth(component)` returns the highest level available:

| Depth | Requires |
|-------|----------|
| `record` | Component exists |
| `scheduled` | Product or facts provide maintenance, warranty or life data |
| `simulated` | Simulation inputs apply to its failure modes |
| `monitored` | A sensor `observes` it or its space for a metric its rules use |
| `predicted` | Monitored, plus fleet data or a validated material model for its failure modes |

Only categories the home subscribes to produce findings. Unsubscribed categories stay at
`record`.

**Coverage map:** per category and space, shows depth achieved and which rules are
running on simulated vs. sensor basis.

**Sensor recommendations:** for each candidate sensor type and space, compute how many
rules or failure modes would move from `simulated` to `monitored`. Rank and show the top
recommendations. During construction, surface these while walls are open.

### Categories and pricing
- `home_subscriptions` — home_id, category, active_from, active_to
- Price is computed by `computePrice(home, categories)` from `conditioned_sqft` tier,
  selected categories, and category scale factors (e.g. number of HVAC systems,
  presence of pool). All amounts live in a config table, not in code.
- Adding components that change a scale factor (a new HVAC system, a pool) recalculates
  the price and notifies the owner before it takes effect.

## Key flows

### Scan an item
1. Scan barcode → look up `product_identifiers`.
2. Found → create component linked to product, ask for space and (optionally) serial number.
3. Not found → photo of label and item, AI extracts manufacturer, model and specs into a
   draft product (source `ai_extracted`); user confirms; draft enters catalog review.
4. Write events; recompute depth and findings for the component.

### Custom object
Create a component without a product, add `component_materials`, facts, photos and
relationships. Material-level failure modes apply automatically.

### Build capture
During construction, components are created from quotes, inspections and photos, and
facts are marked `build_verified` when tied to photos or passed inspections. At project
completion, the as-built record is the home's starting model.

### On-demand agents
Triggers: a finding opens, a reading leaves range, a user asks about an object, a
scheduled review runs. The agent context is assembled from the component subtree,
relationships, resolved facts with sources, open findings, recent readings summary and
the home's responsibility assignments (during projects). Agents explain and recommend.
They never change facts without recording themselves as the source.

## Consent and catalog contribution
- Homes opt in or out of contributing anonymized data to fleet statistics and the shared
  catalog. Default and wording to be decided (see open questions).
- New products created by a user enter the shared catalog only after review, without any
  home-identifying information.

## Build order
1. `homes`, `spaces`, `components` (tree, lifecycle), events, with tests.
2. `products`, `product_identifiers`, scan flow (found path).
3. `facts` with provenance and value resolution.
4. `component_relationships`.
5. Categories, `home_subscriptions`, `computePrice`.
6. Draft product creation from photos (`extractProductSpecs`, with evals).
7. `materials`, `material_compatibility`, `failure_modes` seed data.
8. Rules engine with schedule and recall rules; `findings`.
9. Sensors, streams, readings store, one integration (Home Assistant is a good first target).
10. Operating-range and compatibility rules.
11. Simulation inputs (weather first), `computeDepth`, coverage map, sensor recommendations.
12. Fleet statistics and fleet rules.
13. On-demand agents.

## Required tests
- Component tree: move, refine placeholder, remove, replace; history preserved.
- Fact resolution: component facts override product specs; superseded facts ignored.
- `computeDepth` for each level, including sensor added and removed.
- Rule engine: each rule family fires and does not fire on fixture homes; open findings
  deduplicated; deterministic output.
- `computePrice` across size tiers, category sets, and scale-factor changes.
- Sensor recommendations rank by rules unlocked.

## Open questions
- Who owns the home record, and how it transfers when the home is sold.
- Consent defaults and wording for fleet data and catalog contribution.
- Who authors and reviews rules (trade experts, inspectors, engineers) and how they're paid.
- Time-series store choice and raw-reading retention period.
- Sensor integrations beyond the first (vendor APIs, Matter).
- Sources for product catalog seeding and recall data.
