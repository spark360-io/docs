# Spark360 Outbound Webhooks — Implementation Reference

> **Repo path:** `docs/webhooks/WEBHOOKS_REFERENCE.md`  
> **Prepared:** April 2026  
> **Status:** Pre-implementation reference — authoritative spec for all CC sessions

---

## Guiding Principles

- The **Leads table** is the canonical trigger source — not the inbound Quantiv webhook handler.
- Outbound webhook failures are **silent and non-blocking**. They never disrupt lead creation or core platform flows.
- `lead.enriched` fires **only after** Spark360 enrichment has completed and been written back — never on raw Quantiv data alone.
- Spark360 converts Quantiv 1–5 integer scores to 1–10 scale. All outbound payloads carry the **converted S360 values only**.
- `flowId` and `leadId` are carried in **every** lead event payload for subscriber deduplication and CRM record matching.
- Webhook signing secrets are stored in **Supabase Vault** — never in application columns.

---

## 1. Event Model

| Quantiv Event | `leadStatus` | S360 Outbound Event | Trigger | Availability |
|---|---|---|---|---|
| `lead.created` | `address_info_provided` | `lead.address_preview` | Leads INSERT | Optional |
| `lead.updated` | `project_details_provided` | *(no outbound event)* | — | — |
| `lead.updated` | `personal_info_and_estimate_provided` | `lead.enriched` | Enrichment write-back complete | **REQUIRED — locked** |
| `lead.updated` | `personal_info_and_estimate_provided` + `ownerOccupied=false` | `lead.rental_alert` | Same as above, occupancy check | Optional |
| `lead.updated` | `contact_info_provided` | `lead.contact_requested` | Leads UPDATE | Optional |
| `widget.updated` | n/a | `widget.status_changed` | `organizations.widget_status` UPDATE | Optional |

**Notes:**
- `lead.enriched` is mandatory for every webhook subscription. It cannot be deselected in the UI. A webhook subscription with no `lead.enriched` serves no purpose.
- `lead.rental_alert` fires **alongside** `lead.enriched` when `ownerOccupied = false`. They are independent subscribable events.
- `project_details_provided` does **not** update the S360 leads table and produces no outbound event. The data it carries (roof reason, start time) is included in subsequent payloads.
- `lead.address_preview` carries address only — no name, no contact info. Present as opt-in in the UI with clear messaging.

---

## 2. Widget Version & Source Tracking

Two fields are included in **all outbound lead payloads** from day one. Default values are used until the premium multi-widget and source tracking features ship.

```json
"widget_version": "01",
"source_tracking": {
  "source_code": "01"
}
```

- `widget_version: "01"` — default for single-widget organizations. When the premium multi-widget feature ships, this will reflect the specific widget that generated the lead.
- `source_code: "01"` — default for primary website source. Will expand when lead source tracking is implemented.

These fields must be stubbed with defaults now so subscriber CRM integrations can be built without a breaking schema change later.

---

## 3. Score Conversion

| Score | Quantiv Scale | S360 Scale | Conversion |
|---|---|---|---|
| `lead_integrity_score` | 1–10 | 1–10 | None required |
| `elevate_score` | 1–10 | 1–10 | None required |
| `surepay_score` | 1–5 | 1–10 | Raw × 2 (e.g. raw `2` → stored/outbound `4`) |

The `surepay_score` conversion is handled by an existing S360 function and written to the `leads` table before the outbound payload is built. The `enqueueWebhookEvent` helper reads the stored value directly — **no additional math at dispatch time**.

---

## 4. Payload Schemas

### 4.1 Common Envelope

All events share this outer structure:

```json
{
  "event":           "lead.enriched",
  "event_id":        "uuid-v4",
  "timestamp":       "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-228d-4f0f-b5db-28b49cf4069e",
  "data": { }
}
```

- `event_id` is unique per delivery attempt. Subscribers should use it for idempotency.
- `timestamp` is ISO 8601 UTC.
- `organization_id` is the S360 organization UUID (same as Quantiv `referenceId`).

---

### 4.2 `lead.address_preview`

Fires on `lead.created` / `address_info_provided`. Address only — no name, no contact, no scores.

```json
{
  "event": "lead.address_preview",
  "event_id": "uuid-v4",
  "timestamp": "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-...",
  "data": {
    "lead_id":       "b543b8ca-87ff-45e9-8d04-65c61853c853",
    "flow_id":       "ad64cf83-45d9-4f72-a57a-c028f0678fdc",
    "lead_status":   "address_info_provided",
    "widget_version": "01",
    "source_tracking": { "source_code": "01" },
    "address": {
      "formatted":     "4355 Candacraig, Johns Creek, GA",
      "street_number": "4355",
      "street":        "Candacraig",
      "city":          "Alpharetta",
      "state":         "GA",
      "zip":           "30022",
      "lat":           34.0324253,
      "lng":           -84.2327157
    }
  }
}
```

---

### 4.3 `lead.enriched`

The primary high-value event. Fires **only after** S360 enrichment write-back completes. This is the same trigger point as the existing lead notification email.

```json
{
  "event": "lead.enriched",
  "event_id": "uuid-v4",
  "timestamp": "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-...",
  "data": {
    "lead_id":       "b543b8ca-87ff-45e9-8d04-65c61853c853",
    "flow_id":       "ad64cf83-45d9-4f72-a57a-c028f0678fdc",
    "lead_status":   "personal_info_and_estimate_provided",
    "widget_version": "01",
    "source_tracking": { "source_code": "01" },
    "contact": {
      "first_name":      "Rhett",
      "last_name":       "Butler",
      "email":           "rhett.butler@email.com",
      "phone":           "(770) 448-5252",
      "contact_consent": true
    },
    "address": {
      "formatted":     "4355 Candacraig, Johns Creek, GA",
      "street_number": "4355",
      "street":        "Candacraig",
      "city":          "Alpharetta",
      "state":         "GA",
      "zip":           "30022",
      "lat":           34.0324253,
      "lng":           -84.2327157
    },
    "property": {
      "county":         "Gwinnett",
      "subdivision":    "JAMESON MILL",
      "type":           "Residential",
      "built_year":     "2010",
      "years_in_home":  "16",
      "owner_occupied": true,
      "rental_alert":   false,
      "primary_owner":  "John Joanna",
      "accuracy_code":  "H"
    },
    "roof_estimate": {
      "estimate_number": "R5101UT33",
      "cost_min":        24667,
      "cost_max":        32456,
      "roof_area":       5193.46,
      "roof_material":   "asphalt shingle",
      "complexity":      "moderate",
      "average_pitch":   41.83,
      "accuracy_alerts": {
        "roof_area":  "HIGH_AREA",
        "roof_pitch": "NO_ALERTS"
      }
    },
    "project_details": {
      "reason":     "It's leaking or has water spots",
      "start_time": "In the next few weeks"
    },
    "spark360_scores": {
      "lead_integrity_score":    7,
      "surepay_score":           4,
      "elevate_score":           6,
      "lead_integrity_defects":  ["surname"]
    },
    "sales_hacks": {
      "elevate_hack": "Cautiously upsell — present modest upgrades tied to efficiency or long-term savings.",
      "surepay_hack": "Low reliability — recommend deposit up front and strong financing options.",
      "buyer_insight_hacks": {
        "popular_picks_hack":       "Show what's trending in their area. Reinforce their decision is in line with other happy customers.",
        "quality_first_buyer_hack": "Focus on material quality, expert installation, and long-term value."
      }
    }
  }
}
```

**Notes:**
- `sales_hacks` contains only the **two highest-scoring, actionable** buyer insight hacks. Null hacks are omitted entirely — not sent as null keys.
- `elevate_hack` and `surepay_hack` are always included when available.
- `buyer_insight_hacks` contains the top two scoring insight hacks only.
- Raw `buyerInsights` scores from the Quantiv enrichment blob are **not** included in the outbound payload (MVP scope).
- `surepay_score` in the payload is the S360-converted value (already stored in `leads` table).

---

### 4.4 `lead.rental_alert`

Fires **alongside** `lead.enriched` when `property.owner_occupied = false`. Focused subset for contractors who want a dedicated rental trigger.

```json
{
  "event": "lead.rental_alert",
  "event_id": "uuid-v4",
  "timestamp": "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-...",
  "data": {
    "lead_id":  "b543b8ca-87ff-45e9-8d04-65c61853c853",
    "flow_id":  "ad64cf83-45d9-4f72-a57a-c028f0678fdc",
    "widget_version": "01",
    "source_tracking": { "source_code": "01" },
    "contact": {
      "first_name": "Rhett",
      "last_name":  "Butler",
      "address":    "4355 Candacraig, Johns Creek, GA"
    },
    "property": {
      "owner_occupied": false,
      "rental_alert":   true
    },
    "spark360_scores": {
      "lead_integrity_score": 7
    }
  }
}
```

---

### 4.5 `lead.contact_requested`

Fires when `contact_info_provided` is received. Always arrives **after** `lead.enriched` — never in an earlier payload. Same trigger point as the existing "contact requested" notification email.

```json
{
  "event": "lead.contact_requested",
  "event_id": "uuid-v4",
  "timestamp": "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-...",
  "data": {
    "lead_id":    "b543b8ca-87ff-45e9-8d04-65c61853c853",
    "flow_id":    "ad64cf83-45d9-4f72-a57a-c028f0678fdc",
    "lead_status": "contact_info_provided",
    "widget_version": "01",
    "source_tracking": { "source_code": "01" },
    "contact": {
      "first_name": "Rhett",
      "last_name":  "Butler",
      "email":      "rhett.butler@email.com",
      "phone":      "(770) 448-5252"
    },
    "contact_preference": {
      "phone_call":           "Weekend works best",
      "appointment":          null,
      "asked_not_to_contact": false
    }
  }
}
```

---

### 4.6 `widget.status_changed`

Fires from the existing inbound Quantiv `widget.updated` handler. No new database trigger required.

```json
{
  "event": "widget.status_changed",
  "event_id": "uuid-v4",
  "timestamp": "2026-04-06T14:22:00Z",
  "organization_id": "b8ff2073-...",
  "data": {
    "widget_status":   "inactive",
    "previous_status": "active",
    "changed_by":      "system"
  }
}
```

- `changed_by: "system"` — driven by subscription status change
- `changed_by: "admin"` — driven by SuperAdmin manual toggle

---

## 5. Database Schema

### 5.1 `webhook_subscriptions`

One row per endpoint per organization. An organization may have multiple subscriptions.

```sql
CREATE TABLE public.webhook_subscriptions (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id bigint NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  description     text,            -- subscriber label e.g. 'HubSpot CRM'
  endpoint_url    text NOT NULL,   -- https:// enforced at application layer
  secret_id       uuid NOT NULL,   -- references vault.secrets (NOT the secret itself)
  is_active       boolean NOT NULL DEFAULT true,
  event_types     text[] NOT NULL, -- validated against allowed enum at app layer
  created_at      timestamptz NOT NULL DEFAULT now(),
  updated_at      timestamptz NOT NULL DEFAULT now()
);
```

**RLS:** Organizations can only read/write their own rows. Use service role for admin operations.

---

### 5.2 `webhook_deliveries`

Async delivery queue. One row per delivery attempt group.

```sql
CREATE TABLE public.webhook_deliveries (
  id                      uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_subscription_id uuid NOT NULL REFERENCES webhook_subscriptions(id),
  event_type              text NOT NULL,
  payload                 jsonb NOT NULL,
  status                  text NOT NULL DEFAULT 'pending'
    CHECK (status IN ('pending', 'delivered', 'failed', 'abandoned')),
  attempt_count           int NOT NULL DEFAULT 0,
  last_attempted_at       timestamptz,
  next_retry_at           timestamptz NOT NULL DEFAULT now(),
  response_status         int,    -- HTTP status from subscriber endpoint
  response_body           text,   -- truncated to 1000 chars
  created_at              timestamptz NOT NULL DEFAULT now()
);

-- Required: cron worker queries on (status, next_retry_at)
CREATE INDEX idx_webhook_deliveries_queue
  ON webhook_deliveries (status, next_retry_at)
  WHERE status IN ('pending', 'failed');
```

**RLS:** `webhook_deliveries` is service-role only — never exposed to subscriber JWT queries.

---

### 5.3 Supabase Vault

Signing secrets are stored in Supabase Vault. `webhook_subscriptions.secret_id` holds only the Vault UUID — never the raw secret.

- Read secret at dispatch time via Vault API using `secret_id`
- Secret is generated on subscription creation: `crypto.randomBytes(32).toString('hex')`
- Displayed once to subscriber at creation; masked thereafter
- Regeneration invalidates previous secret immediately

---

### 5.4 Vault Wrapper Functions

Three `SECURITY DEFINER` wrapper functions in `public` provide service-role-safe Vault access through RPC (instead of direct `vault` schema table access):

- `public.create_vault_secret(secret_value text, secret_name text, secret_description text DEFAULT null) RETURNS uuid`
- `public.read_vault_secret(secret_id uuid) RETURNS text`
- `public.delete_vault_secret(secret_id uuid) RETURNS void`

Usage:
- `create_vault_secret` when creating a webhook subscription secret
- `read_vault_secret` when dispatching deliveries and for Test Endpoint secret read
- `delete_vault_secret` when deleting a subscription or replacing a regenerated secret

Permissions are service-role only:
- `REVOKE` from `public`, `anon`, and `authenticated`
- `GRANT` execute to `service_role`

---

## 6. Trigger Architecture

### 6.1 Lead Events — Application Layer

All lead webhook events fire from the **application layer**, not Postgres triggers. This is consistent with the existing lead notification email pattern.

| Trigger point | Event(s) fired | Location in codebase |
|---|---|---|
| Leads INSERT with `address_info_provided` | `lead.address_preview` | `src/app/api/quantiv/` — existing handler |
| Enrichment write-back complete (scores + buyer insights saved) | `lead.enriched` + conditionally `lead.rental_alert` | Enrichment write-back function — same point existing notification email fires |
| Leads UPDATE with `contact_info_provided` | `lead.contact_requested` | `src/app/api/quantiv/` — existing handler |

**Sequencing gate:** `lead.enriched` must never fire on the raw `personal_info_and_estimate_provided` Quantiv event. It fires only after the enrichment API response has been successfully written to both `leads` and `quantiv_product_responses`.

### 6.2 Widget Status Event

`widget.status_changed` fires from the **existing** inbound Quantiv `widget.updated` handler at `src/app/api/quantiv/`. When that handler updates `organizations.widget_status`, it calls `enqueueWebhookEvent('widget.status_changed', orgId, payload)`. No new database trigger needed.

---

## 7. enqueueWebhookEvent — Core Helper

**File:** `src/lib/webhooks/enqueue-webhook-event.ts`

**Signature:**
```typescript
enqueueWebhookEvent(
  eventType:      string,
  organizationId: string,
  payload:        object,
  leadId?:        string,
  flowId?:        string,
  client?:        SupabaseClient
): Promise<void>
```

> Lead events should pass `leadId` and `flowId`. Non-lead events (e.g. `widget.status_changed`) pass `undefined` for both.

Client handling:
- Route handler context (`src/app/api/quantiv/webhook/route.ts`): pass the route's existing admin client as the last argument
- Server action context: omit `client` and let helper fallback to `getSupabaseServerActionClient({ admin: true })`

**Responsibilities:**
1. Build complete outbound payload from leads row, enrichment data, and org context
2. Query `webhook_subscriptions` for all active subscriptions where `organization_id` matches and `event_types` contains the event being fired
3. Write one row to `webhook_deliveries` per matching subscription (`status = 'pending'`, `next_retry_at = now()`)
4. Return silently on any failure — **never throw, never block the calling function**

---

## 8. Delivery Worker

**Supabase Edge Function:** `process-webhook-deliveries`  
**Schedule:** pg_cron every 60 seconds

### Processing loop

```sql
SELECT * FROM webhook_deliveries
WHERE status IN ('pending', 'failed')
  AND next_retry_at <= now()
FOR UPDATE SKIP LOCKED;
```

> `FOR UPDATE SKIP LOCKED` is essential — prevents concurrent cron runs from double-processing the same row.

### Response handling

| HTTP Response | Action |
|---|---|
| `2xx` | `status = 'delivered'` |
| `4xx` | `status = 'abandoned'` immediately — subscriber config error, retrying won't help |
| `5xx` or timeout | `status = 'failed'`, increment `attempt_count`, set `next_retry_at` per backoff |
| `attempt_count >= 5` | `status = 'abandoned'`, trigger abandonment email |

Worker implementation requirements:
- Edge Function secret reads must use wrapper RPC: `supabase.rpc('read_vault_secret', { secret_id })`
- Do **not** query `vault.decrypted_secrets` directly from Edge Function PostgREST (`Invalid schema: vault`)
- Both the row fetch query and `guardedUpdateDelivery` status guard must accept `pending` and `failed`
- `guardedUpdateDelivery` guard should use `.in('status', ['pending', 'failed'])` to allow successful retries to transition failed rows to delivered

### Request headers on every delivery

```
X-Spark360-Signature: sha256=<hmac_sha256(secret, raw_body)>
X-Spark360-Event:     lead.enriched
X-Spark360-Delivery:  <delivery_uuid>
```

Signature is computed as: `HMAC-SHA256(secret, raw_request_body)`

---

## 9. Failure Notification Emails

Uses existing Resend transactional email infrastructure. Recipients: organization owner + all admin-role members.

| Attempt | Action | Email content |
|---|---|---|
| 1–3 | Silent retry only | — |
| 4 (warning) | Retry + send warning email | "Your webhook endpoint is failing. It will be paused in approximately 4 hours if delivery continues to fail. Please check your endpoint configuration." Include: endpoint URL, HTTP response code, link to Settings > Webhooks. |
| 5 → Abandoned | Set `status = abandoned` + send abandonment email | "Your webhook endpoint has been paused after 5 consecutive failures. Re-enable it in Settings > Webhooks once your endpoint is restored." Include: endpoint URL, HTTP response code, link to Settings > Webhooks. |

> Notification emails must never block the delivery worker loop. Wrap Resend calls in try/catch and fail silently.

---

## 10. Retry Schedule

| Attempt | Delay after previous |
|---|---|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 10 minutes |
| 4 | 1 hour |
| 5 | 4 hours → abandoned |

---

## 11. Security

### HMAC-SHA256 Signing
```
X-Spark360-Signature: sha256=<hmac_sha256(secret, raw_body)>
```
Public developer docs will include verification samples in JavaScript, PHP, and Python.

### HTTPS Enforcement
Reject any `endpoint_url` not beginning with `https://`. Enforced at subscription creation and edit with inline validation error.

### Secret Lifecycle
- Generated: `crypto.randomBytes(32).toString('hex')`
- Stored: Supabase Vault (only UUID stored in `webhook_subscriptions.secret_id`)
- Displayed: once on creation, masked thereafter with Reveal option
- Regeneration: requires confirmation, immediately invalidates previous secret

---

## 12. Subscriber Settings UI — Webhooks Tab

Located in subscriber Settings alongside the existing Zapier tab.

### Event type checkboxes

| Event | Default | Notes |
|---|---|---|
| `lead.address_preview` | Off (opt-in) | Address only — no name or contact info |
| `lead.enriched` | **REQUIRED — cannot be disabled** | Core event — mandatory for all subscriptions |
| `lead.rental_alert` | Off (opt-in) | Rental properties only |
| `lead.contact_requested` | Off (opt-in) | Contact preference submitted |
| `widget.status_changed` | Off (opt-in) | Widget toggled active/inactive |

`lead.enriched` must be rendered as a locked checked state — not a checkbox the user can toggle.

### Endpoint card fields
- Endpoint URL (HTTPS enforced, inline validation)
- Description (optional label e.g. "HubSpot CRM production")
- Event type checkboxes (per above)
- Signing secret — masked with Reveal toggle + Regenerate button (confirmation required)
- Test Endpoint button — sends sample `lead.enriched` payload, shows HTTP response code inline within 10 seconds
- Enable/Disable toggle — pauses delivery without deleting
- Delete button — confirmation required, hard deletes subscription and all delivery history

### Delivery log (per endpoint)
- Last 50 deliveries: timestamp, event type, status badge, HTTP response code
- Expandable row: full payload JSON sent + response body received (truncated to 1000 chars)
- Failed and Abandoned rows highlighted in red-600/red-700 (existing color convention)

### Empty state
Clear onboarding message, link to developer documentation, "Add Endpoint" CTA in `#FF622C` orange.

---

## 13. SuperAdmin Tooling

Additions to the SuperAdmin Organizations view:

- Webhook subscription count badge on each organization row
- "View Webhooks" in row ellipsis menu — slide-over with all subscriptions + aggregate delivery stats (reference: beta feedback queue slide-over as pattern)
- "Disable All Webhooks" in row ellipsis menu — bulk toggle for misbehaving organizations

---

## 14. Testing

### Recommended tools

| Tool | URL | Best for |
|---|---|---|
| webhook.site | https://webhook.site | Quick ad-hoc testing — instant HTTPS endpoint, real-time request inspection, no account required |
| Pipedream | https://pipedream.com | Persistent testing — stable endpoint across sessions, can configure forced 4xx/5xx responses to test retry and abandonment behavior |

### Test scenarios

| Scenario | How to test | Expected result |
|---|---|---|
| Successful delivery | Add webhook.site URL; trigger test lead through widget | `status = delivered`; payload visible with correct headers and valid HMAC |
| HMAC verification | Copy `X-Spark360-Signature` from delivered event; verify against secret | Signature validates correctly |
| 4xx abandonment | Pipedream endpoint returning 403 | `status = abandoned` after first attempt; no retries; delivery log shows 403 |
| 5xx retry sequence | Pipedream endpoint returning 500 | Retries on schedule; `attempt_count` increments; warning email on attempt 4; abandoned + email on attempt 5 |
| `lead.enriched` mandatory | Create subscription with only optional events | `lead.enriched` always fires regardless |
| Rental alert co-fire | Trigger lead where `ownerOccupied = false` | Both `lead.enriched` and `lead.rental_alert` delivered |
| Test Endpoint button | Click test button in Settings UI | Sample `lead.enriched` payload delivered within 10 seconds; response code shown inline |

---

## 15. Claude Code Implementation Phases

Execute phases sequentially. Do not begin a phase until the prior phase is confirmed working.

### Phase 1 — Database & Vault Setup
- Create `webhook_subscriptions` migration (schema per Section 5.1)
- Create `webhook_deliveries` migration with queue index (schema per Section 5.2)
- Enable Supabase Vault extension if not already active
- Add RLS: organizations read/write own `webhook_subscriptions` rows; `webhook_deliveries` service-role only
- Reference: `docs/makerkit/` for migration patterns; `CLAUDE.md` for service role key pattern

### Phase 2 — enqueueWebhookEvent Helper
- Create `src/lib/webhooks/enqueue-webhook-event.ts`
- Implement per Section 7 — explicit `leadId` and `flowId` parameters, silent failure, no throws
- Unit test in isolation before wiring to any triggers

### Phase 3 — Trigger Wiring
- Wire `lead.address_preview` into existing `address_info_provided` handler at `src/app/api/quantiv/`
- Wire `lead.enriched` + `lead.rental_alert` into enrichment write-back function (same point existing notification email fires)
- Wire `lead.contact_requested` into `contact_info_provided` handler
- Wire `widget.status_changed` into existing `widget.updated` handler at `src/app/api/quantiv/`
- Read existing handler files before modifying. Match existing code style.

### Phase 4 — Delivery Worker
- Create Supabase Edge Function: `process-webhook-deliveries`
- Implement `FOR UPDATE SKIP LOCKED` query pattern
- Implement Vault secret retrieval
- Implement HMAC-SHA256 signing (Section 11)
- Implement HTTP POST with headers `X-Spark360-Signature`, `X-Spark360-Event`, `X-Spark360-Delivery`
- Implement 2xx/4xx/5xx response handling per Section 8
- Implement retry backoff per Section 10
- Implement failure notification emails (warning attempt 4, abandonment attempt 5) via Resend per Section 9
- Configure pg_cron: every 60 seconds

### Phase 5 — Subscriber Settings UI
- Add Webhooks tab to subscriber Settings (alongside existing Zapier tab)
- Implement endpoint card (URL, description, event checkboxes, secret, test button, enable/disable, delete)
- `lead.enriched` rendered as locked/required — not a toggleable checkbox
- Implement delivery log component
- Implement empty state with "Add Endpoint" CTA
- Reference: `docs/uxpilot-references/` for visual style; `docs/makerkit/` for Settings tab patterns. Do not copy UX Pilot HTML inline styles.

### Phase 6 — SuperAdmin Tooling
- Add webhook subscription count badge to Organizations table rows
- Add "View Webhooks" and "Disable All Webhooks" to organization row ellipsis menu
- Implement slide-over with subscription list and delivery stats
- Reference: existing beta feedback queue slide-over as pattern

---

## 16. Reference Files for CC Prompts

| File | Purpose |
|---|---|
| `docs/quantiv/quantiv_webhook_lead_payload_examples.txt` | Exact Quantiv inbound payload examples for all four lead stages |
| `docs/quantiv/Quantiv_Score_API_JSONB.txt` | Enrichment API response — source of scores, salesHacks, buyerInsights, property fields |
| `docs/quantiv/quantiv_widget_updated.txt` | `widget.updated` payload — `object.status` field location for `widget.status_changed` trigger |
| `docs/makerkit/` | MakerKit Settings tab and component patterns |
| `docs/uxpilot-references/` | Visual reference only — do not copy inline styles |
| `CLAUDE.md` | Service role key pattern, leads table as canonical source, silent failure principle |
| `src/app/api/quantiv/` | Existing inbound webhook handler — pattern to follow for all application-layer triggers |

---

*Score scale reminder: `lead_integrity_score` and `elevate_score` are native 1–10 from Quantiv. `surepay_score` is raw 1–5 from Quantiv; S360 converts it (raw × 2) and stores the result. The `enqueueWebhookEvent` helper reads the stored value — no math at dispatch time.*
