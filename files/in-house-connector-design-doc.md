# Design Doc — In-house HubSpot ⇄ Salesforce Sync

**Owner:** Marketing Ops & Analytics
**Status:** Draft
**Audience:** you / a data engineer picking this up

---

## 1. Purpose & scope

**What:** Replace the paid native HubSpot↔Salesforce connector with a
warehouse-mediated sync. All data flows *through* BigQuery, never
platform-to-platform.

**Why:** The native connector's documented failure modes (exact-match-only
dedup, no pre-sync filtering, no queryable error log, duplicate/orphan
creation on re-sync) are exactly the things a warehouse-mediated design
lets us own. The warehouse becomes the single control point for cleaning,
matching, and logging.

**Scope:** Contacts/Leads first. Accounts, Opportunities, Campaign Members
are read-only into the warehouse for now (analytics), not written back.

**Non-goals (v1):** real-time event streaming, writing Opportunities from
HubSpot, multi-region.

---

## 2. Architecture

Two **independent one-way feeds**, both mediated by BigQuery:

```
Feed A:  HubSpot  ──API──▶  BigQuery  ──clean/match──▶  Salesforce (upsert)
Feed B:  Salesforce ─API─▶  BigQuery  ──curate──────▶  HubSpot   (batch upsert)
```

They are never a single bidirectional job. Each feed is extract → transform
(in warehouse) → load. Change detection is a modified-time watermark, not a
webhook (v1).

---

## 3. Identity model (the keys)

The whole design stands or falls on stable keys. Three of them:

| Key | Lives in | Type | Purpose |
|---|---|---|---|
| `HubSpot_Contact_ID__c` | Salesforce (Lead + Contact) | Text, **External ID, Unique** | Lets Feed A `upsert()` update-or-create idempotently |
| `salesforce_lead_id` / `salesforce_contact_id` | HubSpot (custom contact property) | string | Lets Feed B batch-upsert idempotently |
| `master_id` | Warehouse (`marts.contact_match`) | UUID | Internal resolved-identity key; one per real person |

**Company/account key:** normalized email domain (`company_domain`), derived
— strip protocol/www, lowercase. Used for account-level rollup and as the
last-resort match key. Never match two *people* on company alone.

**Match priority (highest precision first):**
1. `email` exact (case-insensitive)
2. `company_domain` + last name
3. normalized company name (fuzzy) — flag for review, don't auto-merge people

---

## 4. Data model (BigQuery)

### Datasets
- `staging` — raw landings + logs
- `marts` — resolved/curated tables the feeds read from

### Tables

**Raw landings (WRITE_TRUNCATE per run, v1):**
- `staging.hubspot_contacts_raw`
- `staging.salesforce_leads_raw`
- `staging.salesforce_contacts_raw`
- `staging.salesforce_accounts_raw`
- `staging.salesforce_opportunities_raw`

**Resolved / curated:**
- `marts.stg_hubspot_contacts` — normalized (`company_domain`,
  `company_normalized`, lowercased email)
- `marts.stg_salesforce_leads` — same normalization
- `marts.contact_match` — the identity graph:
  `master_id, hubspot_id, salesforce_id, email, company_domain,
   match_type, match_confidence, updated_at`
- `marts.match_exceptions` — anything `no_match` / one-sided / ambiguous
- `marts.sf_outbound` — **exact columns Salesforce upsert expects**, one row
  per contact, HubSpot-owned fields only
- `marts.hs_outbound` — **exact HubSpot property internal names**, one row
  per contact, Salesforce-owned fields only
- `staging.sync_log` — every write attempt (both feeds):
  `run_id, feed, target_id, external_key, op, status, http_code, error,
   ts` (WRITE_APPEND)

---

## 5. Field ownership matrix

The single most important table in this doc. **No field is authoritative in
both systems.** This is what makes the loop impossible.

| Field | Source of truth | Direction | Lands in |
|---|---|---|---|
| `ga_client_id` | HubSpot (from GA4) | HS → SF | `sf_outbound` |
| `lifecyclestage` / lead score | HubSpot | HS → SF | `sf_outbound` |
| self-reported source | HubSpot | HS → SF | `sf_outbound` |
| Opportunity Stage | Salesforce | SF → HS | `hs_outbound` |
| Lead/Deal Status | Salesforce | SF → HS | `hs_outbound` |
| Owner (sales rep) | Salesforce | SF → HS | `hs_outbound` |
| email, name | either, normalized centrally | match key only | neither writes over |

If a field isn't in this table, it doesn't sync. Adding a field = adding a
row here first.

---

## 6. Feed A — HubSpot → Salesforce

**When:** scheduled incremental (e.g. hourly). Watermark:
`hs.lastmodifieddate > @last_run`.

**How:**
1. `extract_hubspot_contacts()` — paginate `/crm/v3/objects/contacts`,
   filter by watermark → `staging.hubspot_contacts_raw`.
2. dbt/SQL builds `stg_hubspot_contacts` → `contact_match` →
   `sf_outbound` (HubSpot-owned fields only, keyed by `hubspot_id`).
3. `push_to_salesforce()` — for each row,
   `sf.Lead.upsert('HubSpot_Contact_ID__c/{hubspot_id}', {...})`.
   Composite external-ID syntax = update-or-create, never duplicate.
4. Log every result to `staging.sync_log` (feed = `A`).

**Idempotency:** guaranteed by the External ID upsert. Re-running is safe.

---

## 7. Feed B — Salesforce → HubSpot

**When:** scheduled incremental. Watermark:
`sf.LastModifiedDate > @last_run` (SOQL).

**How:**
1. `extract_salesforce()` — `simple_salesforce.query_all()` for Leads
   (and Contacts) modified since watermark → `staging.salesforce_leads_raw`.
2. SQL builds `hs_outbound`: **exact HubSpot property internal names** as
   columns (`salesforce_lead_id`, `hs_lead_status`, `dealstage`, ...),
   Salesforce-owned fields only, one row per contact, keyed by email or
   `salesforce_lead_id`.
3. `push_to_hubspot()` — HubSpot batch upsert
   `POST /crm/v3/objects/contacts/batch/upsert` with `idProperty` set to
   `email` (or the custom `salesforce_lead_id` property). Batches of ≤100.
4. Log every result to `staging.sync_log` (feed = `B`).

**Idempotency:** HubSpot batch upsert keyed on a unique property =
update-or-create. Re-running is safe.

---

## 8. Loop prevention

- Each field is single-direction (Section 5), so Feed B never writes a field
  Feed A owns, and vice versa. A write in one system can't echo back.
- Belt-and-suspenders: exclude records whose only change since last run was a
  sync-written field (compare against `sync_log` last-write ts) before
  pushing.

---

## 9. Observability

- `staging.sync_log` — the queryable error report the native connector
  lacks. "Show all failed syncs today" = one `WHERE status='failed'`.
- `marts.match_exceptions` — standing dedup/orphan audit, refreshed each run.
- **Source freshness test** (dbt `source freshness` or scheduled query):
  did each raw table land, and is row count within a trailing-average band?
  Alert to Slack/email on miss.

---

## 10. Orchestration & security

- **Orchestration:** two Airflow DAGs (or Cloud Scheduler + Cloud Run) —
  `feed_a_hs_to_sf`, `feed_b_sf_to_hs`. Extract → dbt run → push, each with
  its own watermark stored in a `staging.sync_state` table.
- **Auth:** HubSpot private app token (least-privilege: read now, write when
  Feed B lands); `simple_salesforce` username/password/token for the demo,
  OAuth 2.0 JWT bearer in production.
- **Secrets:** `.env` locally (gitignored); Secret Manager in production.
  Never in code or the repo.

---

## 11. Open questions / future

- Move raw landings from truncate to partitioned append for history.
- Webhook-driven (true on-change) vs scheduled incremental — upgrade when
  volume justifies the listener endpoint.
- Extend write-back to Accounts/Opportunities if HubSpot needs deal context
  richer than stage.
- Reverse-ETL tooling (Census / Hightouch) could replace the hand-rolled
  push jobs while keeping the warehouse matching layer — evaluate vs.
  maintenance cost.

---

## 12. Known risks & mitigations

- **Existing duplicates stay in the CRM.** The warehouse knows which records
  are the same person, but old copies still live in Salesforce. If one person
  has three records, an upsert can only update one. Mitigation: one-time
  merge of existing duplicates in Salesforce before go-live; the warehouse
  keeps new ones from forming after that.
- **Native connector left on.** Its main damage is *creating* duplicates,
  which a later overwrite doesn't undo, and two syncs writing the same field
  can flip values every run. Mitigation: turn the native connector off, or
  limit it to fields this feed never touches.
- **Conflicts in a two-way sync.** If Marketing and Sales edit the same field,
  one system has to win. Mitigation: the field ownership matrix (Section 5).
- **Batch delay.** An hourly job means a hot demo request can wait up to an
  hour before a rep sees it. Mitigation: a fast path (webhook or more
  frequent job) for high-intent leads; everything else stays on the batch
  schedule.

---

## 13. Where this fits — a layered approach

Messy contact data is handled in layers. This connector is one of them.

1. **Prevent bad data at entry (cheapest, biggest win)**
   - Form validation: block free/invalid emails; picklists instead of free
     text for country, job function, etc.
   - Enrichment on form submit (e.g. Clearbit, ZoomInfo) to stamp the
     official company domain and a company ID — matching on IDs beats
     fuzzy-matching company names.
   - Salesforce duplicate rules set to block or warn on create.
2. **Match inside the CRM in real time**
   - Lead-to-account matching and routing (e.g. LeanData) and dedup/merge
     tools (e.g. RingLead, DemandTools) run the moment a record arrives —
     solving the batch-delay gap above.
3. **Warehouse as source of truth for analytics, audit, and bulk fixes**
   - This design. Strongest for attribution, duplicate/exception reporting,
     and large backfills or cleanups. Reverse-ETL tools (Hightouch, Census)
     can replace the hand-rolled push jobs.

**Alternative:** remove the sync entirely by running one system — HubSpot as
the CRM, or Salesforce plus its own marketing tool (Account Engagement,
formerly Pardot). A large organizational decision, but sometimes the right
answer.

**Positioning:** in production this connector pairs with prevention at entry
and in-CRM matching for speed; the warehouse is where we measure whether
those layers are working.
