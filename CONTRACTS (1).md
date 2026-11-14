# KrishiMarket: Shared Contracts

> The single source of truth for everything the four workstreams share.
> **Rule:** if your code touches something listed here, follow this file exactly. If you need a change, edit this file in a tiny PR and tell the team. Never silently deviate.
> Companion files: `improvement.md` (what to build), `PROMPTS.md` (what to paste into Antigravity).
>
> The repo's real framework and paths were not visible when this was written. Code below assumes a Python backend (FastAPI-style routers, SQL models, pytest) and a React frontend. **The agent must adapt names and paths to the existing conventions** but keep the behaviour and shapes identical.

---

## 1. Global conventions

| Topic | Rule |
|---|---|
| Weight | Store in **kg** (numeric). Convert quintal etc. only in UI. |
| Money | Store in rupees as `NUMERIC(12,2)` for all **new** tables. Existing columns keep their type. |
| Time | UTC, ISO 8601 in APIs (`2026-11-14T10:30:00Z`). |
| IDs | Follow the existing primary key type of the `transactions` table for foreign keys. |
| Errors | Always `{ "error": { "code": "SOME_CODE", "message": "human text" } }` with the correct HTTP status. |
| Auth | Reuse the existing auth dependency. Do not write a second auth system. |
| Flags | Boolean env vars, default **off** in production, listed in section 12. |
| Labels | Anything synthetic, demo or estimated is flagged in data (`is_demo`, `source`, `is_estimate`) and in UI. |

---

## 2. Roles and permissions

Existing roles: `farmer`, `buyer`, `admin` (and transporter if present). **M1 adds `hub_operator`.**

| Action | farmer | buyer | hub_operator | admin |
|---|---|---|---|---|
| Fire `LOT_DISPATCHED` | owner of lot | no | no | yes |
| Fire `HUB_CHECKIN`, `HUB_VERIFIED` | no | no | yes (assigned hub only) | yes |
| Fire `REQUOTE_PROPOSED` | no | no | system (on variance) | yes |
| Fire `REQUOTE_ACCEPTED` / reject | party to the deal (both must accept) | party to the deal | no | yes |
| Fire `DELIVERED` | no | buyer of the deal | no | yes |
| Open dispute | party to the deal | party to the deal | no | yes |
| Resolve dispute | no | no | no | yes |
| Create `hub_operator` accounts | no | no | no | yes |

Violations return `403 FORBIDDEN_EVENT`.

---

## 3. Delivery events

### 3.1 Enum (owned by M1, file: `shared/events.py`)

```python
from enum import Enum

class EventType(str, Enum):
    LOT_DISPATCHED   = "LOT_DISPATCHED"
    HUB_CHECKIN      = "HUB_CHECKIN"
    HUB_VERIFIED     = "HUB_VERIFIED"
    REQUOTE_PROPOSED = "REQUOTE_PROPOSED"
    REQUOTE_ACCEPTED = "REQUOTE_ACCEPTED"
    REQUOTE_REJECTED = "REQUOTE_REJECTED"
    DELIVERED        = "DELIVERED"
    DISPUTE_OPENED   = "DISPUTE_OPENED"
    DISPUTE_RESOLVED = "DISPUTE_RESOLVED"
```

### 3.2 Payload schemas (`payload` JSON per event)

| Event | Required payload fields |
|---|---|
| `LOT_DISPATCHED` | `listed_weight_kg`, `listed_grade` |
| `HUB_CHECKIN` | `hub_id`, `measured_weight_kg`, `measured_grade`, `photo_ref` |
| `HUB_VERIFIED` | `verified_weight_kg`, `verified_grade`, `variance_pct`, `within_tolerance` (bool) |
| `REQUOTE_PROPOSED` | `old_price_per_kg`, `new_price_per_kg`, `reason`, `variance_pct` |
| `REQUOTE_ACCEPTED` / `REQUOTE_REJECTED` | `by_party` (`farmer` or `buyer`) |
| `DELIVERED` | `confirmed_by` |
| `DISPUTE_OPENED` | `opened_by`, `reason`, `evidence_refs` (list) |
| `DISPUTE_RESOLVED` | `outcome` (`favor_farmer`, `favor_buyer`, `mutual`, `no_fault`), `split_farmer_pct` (only for `mutual`) |

### 3.3 State flow

```
LOT_DISPATCHED -> HUB_CHECKIN -> HUB_VERIFIED -> DELIVERED
                                     |
                          (variance > tolerance)
                                     v
                  REQUOTE_PROPOSED -> REQUOTE_ACCEPTED -> DELIVERED
                                   \-> REQUOTE_REJECTED -> DISPUTE_OPENED
DISPUTE_OPENED -> DISPUTE_RESOLVED   (can open from HUB_CHECKIN onward)
```

All transitions go through the existing `delivery_state_machine.transition()`. No one else mutates delivery state directly.

---

## 4. Ledger (M1 provides)

### 4.1 Function

```python
# ledger/service.py
def record_event(transaction_id, type: EventType, payload: dict, actor: dict) -> "DealEvent":
    """
    actor = {"user_id": ..., "role": "farmer|buyer|hub_operator|admin|system"}
    1. Validate payload against section 3.2 (raise on missing fields).
    2. Append a row to ledger_deal_events with the hash chain.
    3. After commit, call escrow.on_event(transaction_id, type) if the escrow module exists
       (guarded: ImportError or flag off must NOT raise).
    4. Return the stored event.
    """
```

**Nobody except `ledger/service.py` inserts into `ledger_deal_events`.**

### 4.2 Table `ledger_deal_events`

| Column | Type | Notes |
|---|---|---|
| id | pk | |
| transaction_id | fk | indexed |
| seq | int | 0-based position within the transaction chain |
| type | text | EventType value |
| payload | json/text | canonical JSON string |
| actor_user_id | fk nullable | |
| actor_role | text | |
| created_at | timestamp UTC | |
| prev_hash | char(64) | `"0"*64` for seq 0 |
| hash | char(64) | |

### 4.3 Exact hash algorithm (so any agent produces the same result)

```python
import hashlib, json

def canonical(obj) -> str:
    return json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)

def compute_hash(prev_hash, transaction_id, seq, type, payload, actor_user_id, actor_role, created_at_iso):
    body = canonical({
        "transaction_id": transaction_id, "seq": seq, "type": type,
        "payload": payload, "actor_user_id": actor_user_id,
        "actor_role": actor_role, "created_at": created_at_iso,
    })
    return hashlib.sha256((prev_hash + body).encode("utf-8")).hexdigest()
```

`created_at_iso` is stored exactly as hashed (second precision, `Z` suffix).

### 4.4 Endpoints (M1)

```
GET  /deals/{transaction_id}/proof
200 -> {
  "transaction_id": 12,
  "valid": true,
  "event_count": 5,
  "first_broken_seq": null,
  "head_hash": "ab12...",
  "events": [ {"seq":0,"type":"LOT_DISPATCHED","actor_role":"farmer","created_at":"...","hash":"..."} ],
  "disclaimer": "Tamper-evident, not tamper-proof."
}
```
Public read (no login), but `events` omit personal data and photo refs. If tampered: `valid:false` and `first_broken_seq` set.

```
POST /deals/{transaction_id}/events          # farmer dispatch, buyer delivered, dispute open
body: { "type": "LOT_DISPATCHED", "payload": {...} }
201 -> stored event summary

POST /hub/transactions/{transaction_id}/checkin   # hub_operator
body (multipart): measured_weight_kg, measured_grade, photo
POST /hub/transactions/{transaction_id}/verify    # hub_operator
-> computes variance, writes verified_weight_kg and verified_grade, records HUB_VERIFIED,
   proposes a re-quote if variance exceeds tolerance.

POST /deals/{transaction_id}/requote/respond      # farmer or buyer
body: { "accept": true }
```

Variance rule: `variance_pct = abs(verified_weight_kg - listed_weight_kg) / listed_weight_kg * 100`; a grade mismatch also counts as out of tolerance. Tolerance default `3.0`, config key `HUB_VARIANCE_TOLERANCE_PCT`.

### 4.5 Columns M1 adds to `transactions` (one migration)

`listed_weight_kg`, `listed_grade`, `verified_weight_kg` (nullable), `verified_grade` (nullable), `verified_at` (nullable), `hub_id` (if not present).

**M4 only reads `verified_weight_kg` and `verified_grade`. M3 only reads them too.**

---

## 5. Escrow hook (M3 provides)

```python
# escrow/service.py
def on_event(transaction_id, event_type: EventType) -> None:
    """Idempotent. Safe to call twice with the same event. Never raises into the caller."""
```

| Event | Escrow effect |
|---|---|
| `HUB_VERIFIED` | release `ESCROW_PARTIAL_PCT` (default 30) of funded amount, state `released_partial` |
| `DELIVERED` | release the remainder, state `released_full`, then `settled` |
| `DISPUTE_OPENED` | state `frozen` |
| `DISPUTE_RESOLVED` | apply outcome: `favor_farmer` release rest; `favor_buyer` refund rest; `mutual` split by `split_farmer_pct`; `no_fault` release rest; then `settled` |
| `REQUOTE_ACCEPTED` | recompute total from new price x verified weight |

States: `unfunded`, `funded`, `released_partial`, `released_full`, `frozen`, `settled`.
Invariant: `funded_amount == released_to_farmer + refunded_to_buyer + still_held`.

### Tables (all prefixed `escrow_`)

- `escrow_accounts`: id, transaction_id (unique), state, funded_amount, released_to_farmer, refunded_to_buyer, partial_pct, is_simulated (always true), created_at
- `escrow_movements`: id, escrow_id, kind (`fund`, `release_partial`, `release_full`, `refund`, `freeze`), amount, related_event_id, created_at

### Endpoints

```
POST /escrow/{transaction_id}/fund        # buyer, Razorpay test mode (or mock when flag off)
GET  /escrow/{transaction_id}             # farmer/buyer/admin: state, movements, is_simulated
POST /dev/escrow/{transaction_id}/trigger # DEV ONLY (404 when ENVIRONMENT=production)
body: { "event_type": "HUB_VERIFIED" }
```

Every response includes `"is_simulated": true`. UI shows "Simulation / test mode".

---

## 6. Advice API (M2 provides)

```
GET /advice/{commodity}/{market}
```

### Response shape (exact)

```json
{
  "commodity": "onion",
  "market": "Lasalgaon",
  "unit": "kg",
  "forecast": 24.5,
  "band_low": 22.1,
  "band_high": 27.0,
  "band_level": 0.8,
  "skill": 0.14,
  "best_model": "ets",
  "recommendation": "HOLD",
  "recommendation_reason": "Expected gain exceeds storage cost; lower band supports holding.",
  "msp_floor": null,
  "data_source": "agmarknet",
  "data_age_days": 6,
  "history_points": 540,
  "generated_at": "2026-11-14T02:00:00Z"
}
```

Rules:
- Prices are **per kg**.
- `recommendation` is one of `SELL_NOW`, `HOLD`, `NO_CLEAR_EDGE`.
- `skill` is `null` if there was not enough history; then `recommendation` is `NO_CLEAR_EDGE`.
- `data_source` is `agmarknet`, `synthetic` or `mixed`. If not fully real, the UI must say so.
- Unknown commodity/market: `404 ADVICE_NOT_FOUND`.

### Day-1 sample file (M2 commits it)

Serve the JSON above from `GET /advice/onion/Lasalgaon` with `ADVICE_USE_SAMPLE=true`, so M3 can build immediately. M2 must keep that flag working until the real model is ready.

---

## 7. Offers and net-in-hand (M3 provides)

```
GET /offers/{offer_id}/net
200 -> {
  "offer_id": 7,
  "offer_price_per_kg": 26.0,
  "transport_per_kg": 1.8,
  "handling_per_kg": 0.4,
  "fees_per_kg": 0.3,
  "shrink_per_kg": 0.5,
  "net_per_kg": 23.0,
  "batna_per_kg": 23.6,
  "batna_source": "nearest_mandi",
  "fairness": "amber",            // green | amber | red
  "is_estimate": true,
  "suggested_counter_per_kg": 25.0
}

POST /offers/{offer_id}/counter     # farmer, once only
body: { "price_per_kg": 25.0 }
POST /offers/{offer_id}/counter/respond   # buyer
body: { "accept": true }
```

- Offer state adds `countered`. A second counter returns `409 COUNTER_ALREADY_USED`.
- Fairness: `green` if net >= batna; `amber` if within 10% below batna; `red` otherwise (thresholds in config).
- Config table `offer_rate_config`: key, value, unit, is_estimate (default true), editable by admin. Keys: `handling_per_kg`, `fee_pct`, `shrink_pct`, `vehicle_rate_per_km`, `fairness_amber_pct`.
- BATNA reads M2's `band_low` if the flag `FEATURE_ADVICE` is on and the call succeeds; otherwise it falls back to the nearest mandi modal price. Never fail the endpoint because advice is down.

---

## 8. Pooling (M4 provides)

### Tables (prefixed `pool_`)

- `pool_demands`: id, buyer_id, commodity, grade, quantity_kg, delivery_district, needed_by, max_price_per_kg, status (`open`, `pooled`, `closed`), is_demo
- `pool_pools`: id, demand_id, hub_id, transporter_id nullable, total_kg, landed_cost, status (`proposed`, `confirmed`, `delivered`)
- `pool_members`: id, pool_id, lot_id, farmer_id, committed_kg, price_per_kg, share_pct, joined (bool), manual_verified_weight_kg nullable
- `pool_grade_prices`: commodity, grade, multiplier (editable; for example A = 1.00, B = 0.92)

### Endpoints

```
POST /pools/demands                      # buyer
POST /pools/demands/{id}/assemble        # runs optimiser, returns proposed pool
POST /pools/{pool_id}/join               # farmer, one tap
GET  /pools/{pool_id}                    # members, shares, payout formula
GET  /pools/{pool_id}/payouts            # per farmer payout + formula string
```

Payout per farmer = `weight_used_kg x price_per_kg x grade_multiplier`, where
`weight_used_kg = transactions.verified_weight_kg` if present, else `pool_members.manual_verified_weight_kg`, else `committed_kg`. The response states which source was used (`"weight_source": "verified|manual|committed"`).

Optimiser: PuLP or OR-Tools, with a **greedy fallback** if solver import fails or times out (2 s).

---

## 9. Frontend ownership

| Workstream | Frontend folder (create if absent) | Routes |
|---|---|---|
| M1 | `src/features/proof/`, `src/features/hub/` | `/hub`, `/proof/:id` |
| M2 | `src/features/advice/` | embedded `AdviceCard` component |
| M3 | `src/features/offers/`, `src/features/escrow/` | embedded in offer and transaction pages |
| M4 | `src/features/pooling/`, `src/features/voice/` | `/pools`, `/demands/new` |

Shared files (router, nav, layout): add one line each, nothing else.

## 10. Backend ownership

| Workstream | Router files | Model/migration prefix |
|---|---|---|
| M1 | `routers/ledger.py`, `routers/hub.py` | `ledger_` |
| M2 | `routers/advice.py`, `services/forecast/` | `advice_` |
| M3 | `routers/offers.py`, `routers/escrow.py` | `offer_`, `escrow_` |
| M4 | `routers/pooling.py` | `pool_` |

Migration file names: `YYYYMMDD_m1_<topic>.py`, `..._m2_...`, `..._m3_...`, `..._m4_...`. Each migration only touches its own prefixed tables, plus M1's one allowed change to `transactions` (section 4.5).

## 11. i18n

Add keys in a commented block per feature in en, hi, mr (same keys in all three):

```
// ===== M1 proof/hub =====    keys start with  proof.  hub.
// ===== M2 advice =====        keys start with  advice.
// ===== M3 offers/escrow =====  keys start with  offer.  escrow.
// ===== M4 pooling/voice =====  keys start with  pool.  voice.
```

## 12. Feature flags

| Flag | Owner | Default (prod) | Meaning |
|---|---|---|---|
| `FEATURE_PROOF_LEDGER` | M1 | off | hub flow and ledger endpoints |
| `FEATURE_ADVICE` | M2 | off | `/advice` real model |
| `ADVICE_USE_SAMPLE` | M2 | off | return the hardcoded sample |
| `FEATURE_OFFER_NET` | M3 | off | net-in-hand and counter-offer |
| `FEATURE_ESCROW_SIM` | M3 | off | simulated escrow |
| `FEATURE_POOLING` | M4 | off | pooling pages and API |
| `FEATURE_VOICE` | M4 | off | Bhashini voice components |
| `FEATURE_DEV_TOOLS` | all | off | dev-only trigger endpoints |

For the demo, set all to on in the demo environment only.

## 13. Stub rules (working before others finish)

| If you need... | ...until it exists, do this |
|---|---|
| M1 events (M3) | Use `POST /dev/escrow/{id}/trigger` |
| M1 verified weight (M4) | Use `pool_members.manual_verified_weight_kg` |
| M2 advice (M3) | Call `/advice/onion/Lasalgaon` with `ADVICE_USE_SAMPLE=true`; fallback to mandi price on any error |
| M3 escrow hook (M1) | `record_event` guards the import; no-op if absent |

When a dependency lands, delete the stub path in the same PR that switches to the real one.

## 14. Definition of done (every workstream)

1. Acceptance criteria in `improvement.md` met.
2. Tests added and the whole suite passes.
3. Flag documented in section 12 and working both on and off.
4. i18n keys present in en, hi and mr.
5. No secrets, no SQLite files, no real personal data committed.
6. Status row in `improvement.md` section 10 updated.
