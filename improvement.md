# KrishiMarket: Improvement Plan (Pool, Prove, Pay)

> **Audience:** AI coding agents (Antigravity) and the 4 team members.
> **Goal:** Move KrishiMarket from "show the farmer a price" to "help the farmer get that price, with proof."
> **How to use:** Each team member tells the agent: *"Read improvement.md. Implement Workstream M<N> only. Follow the Global Rules and Contracts."* Do not implement other workstreams.

---

## 0. Instructions for the agent (read first)

1. **Inspect before editing.** Read the repo structure, existing models, routers, `delivery_state_machine`, the recommender, hub check-in component and tests. Match existing conventions (naming, ORM, auth, i18n, test style). Do not assume file paths; find them.
2. **Stay in your lane.** Only create or edit files owned by your workstream. For shared files (`main.py`, nav, i18n files) make the smallest possible change.
3. **Plan first.** Before coding, write a short plan listing files you will create or change. Then implement in small, reviewable commits.
4. **Tests are required.** Every new endpoint or rule gets backend tests. Existing tests (41 backend, 6 + 7 recommender) must keep passing.
5. **Feature flags.** Put each feature behind a flag (env or config) so a half-finished feature never breaks the demo.
6. **Honesty rules (non-negotiable):**
   - Escrow is **simulated**. Label it "Simulation / test mode" in UI and code comments. Never imply real fund holding.
   - Never display forecast accuracy numbers that were not measured by the backtest.
   - Synthetic or demo data must be labelled as such everywhere it is shown.
   - Fee, shrink and transport rates are **estimates**; label them and keep them editable.
   - The hash chain is tamper-**evident**, not tamper-proof. Say so on the proof page.
7. **Do not** commit secrets, SQLite files, or real personal data. Do not add new heavy dependencies without noting why in the PR.
8. **Done means:** acceptance criteria met, tests pass, flag documented, i18n keys added (en/hi/mr), short notes added to this file's "Status" section.

---

## 1. Shared contracts (agreed on day 1)

These are the only coupling between workstreams. Build against them; use stubs until the owner ships the real thing.

### 1.1 Event types (shared enum)

```
LOT_DISPATCHED, HUB_CHECKIN, HUB_VERIFIED,
REQUOTE_PROPOSED, REQUOTE_ACCEPTED,
DELIVERED, DISPUTE_OPENED, DISPUTE_RESOLVED
```

Put the enum in one shared module (owned by M1). Others import it, never redefine it.

### 1.2 Ledger function (M1 provides)

```python
record_event(transaction_id, type, payload, actor) -> DealEvent
```

All other code calls this. **Nobody else writes to `deal_events` directly.**

### 1.3 Event hook (M3 provides)

```python
escrow.on_event(transaction_id, event_type)
```

M1 calls this after each recorded event. Until M1 is done, M3 triggers it from a dev-only endpoint (disabled in production).

### 1.4 Advice API (M2 provides)

```
GET /advice/{commodity}/{market}
-> {
  forecast, band_low, band_high, skill,
  recommendation,        // "SELL_NOW" | "HOLD" | "NO_CLEAR_EDGE"
  data_source,           // "agmarknet" | "synthetic"
  data_age_days
}
```

M2 publishes a **hardcoded sample response on day 1** so M3 can build against it.

### 1.5 Verified fields (M1 writes, M4 reads)

Stored on the transaction: `verified_weight_kg`, `verified_grade`.

### 1.6 Units

Store everything in **kg**. Convert (quintal, etc.) only for display.

---

## 2. Global rules to avoid merge conflicts

- Separate backend routers, models and frontend folders per workstream (e.g. `routers/ledger.py`, `routers/advice.py`, `routers/offers.py`, `routers/escrow.py`, `routers/pooling.py`).
- One migration file per person; prefix new table names (`ledger_`, `advice_`, `offer_`/`escrow_`, `pool_`).
- Translation keys: add in a **separate, clearly commented section per feature** in en/hi/mr.
- One long-lived branch per person; merge to `main` at checkpoints only.
- Shared-file edits: tiny PRs, merged fast.

---

## 3. Workstream M1: Trust Core (U1 + security fixes)

**Goal:** A hub operator verifies produce (photo, weight, grade); every deal step is recorded in a verifiable hash chain.

**Depends on:** nothing. This is the base.

### Tasks

1. **Role and endpoints**
   - Add `hub_operator` role (created by admin only).
   - Add endpoints that fire `HUB_CHECKIN` then `HUB_VERIFIED` through `delivery_state_machine.transition()`.
   - Write photo, weight, grade into the existing hub check-in columns. Set `verified_weight_kg` and `verified_grade` on the transaction.
2. **Role-restricted events (KI-03)**
   - Farmer dispatches, hub operator checks in/verifies, buyer confirms delivery. Reject others with 403.
3. **Variance rule and re-quote**
   - If hub weight or grade differs from the listing beyond a tolerance (default **3%**, configurable), propose a re-quote (`REQUOTE_PROPOSED`).
   - Both sides can accept (`REQUOTE_ACCEPTED`) or reject. Rejection opens a dispute with the evidence attached.
4. **Proof ledger**
   - Table `ledger_deal_events`: `id, transaction_id, type, payload, actor, created_at, prev_hash, hash`.
   - `hash = SHA-256(prev_hash + canonical_json(payload + type + actor + created_at))`. Use canonical JSON (sorted keys, fixed separators).
   - Implement `record_event(...)` (see contract 1.2) and call `escrow.on_event` if available (guarded, no crash if absent).
   - `GET /deals/{id}/proof` recomputes the whole chain and returns `valid: true/false` plus the first broken index.
   - Frontend proof page with a QR code on the transaction page. Green when valid, red when broken. Include a visible note: "Tamper-evident, not tamper-proof."
5. **Security and hygiene fixes**
   - **KI-05:** serve verification ID files only through an authenticated, authorized download route; remove the unauthenticated static mount.
   - **KI-15:** fail closed. Production defaults must have no demo admin and no plaintext OTP. If `ENVIRONMENT` is unset, behave as production.
   - **KI-14:** add missing icon/og files or remove references so the service worker precache does not 404.
   - **KI-16/24:** remove committed SQLite files from git (add to `.gitignore`); pin `requirements`.

### Acceptance criteria

- Full path works end to end: dispatch -> hub check-in -> verify -> delivered.
- Wrong-role event attempts return 403 (tested).
- Weight variance above tolerance triggers a re-quote; rejection creates a dispute.
- Manually editing any ledger row makes `/deals/{id}/proof` return invalid (tested).
- Unauthenticated request for an ID document is rejected.

---

## 4. Workstream M2: Data & AI (U4 Calibrated Advice)

**Goal:** Honest forecasting that shows uncertainty, compares itself to "tomorrow = today," and says "no clear edge" when it has none.

**Depends on:** nothing. Standalone service plus one chart component.

### Tasks

1. **Data foundation (do first)**
   - Get a proper data.gov.in API key (the fallback key returns only a handful of records). Backfill real price history.
   - Add a `source` column (`agmarknet` / `synthetic`) to price records. Show a visible **data badge** and a **stale-data note** (days since last real record).
   - Disable auto-seed of synthetic data in production (KI-01).
   - Add an **MSP floor marker** where MSP is known. Never advise selling below MSP.
2. **Aggregation (KI-06)**
   - One series per commodity + market per day (median modal price) before modelling.
3. **Models**
   - Baselines: naive, seasonal-naive.
   - Candidates: ETS/SARIMA (statsmodels) and gradient boosting with calendar and lag features. Optional: Prophet or Chronos-2 as a zero-shot benchmark.
4. **Backtest**
   - Rolling-origin backtest per commodity (nightly job or management command).
   - Store MAE and `skill = 1 - MAE / MAE_naive`. Pick the best model per commodity.
   - Require a minimum amount of history; below it, return `NO_CLEAR_EDGE` with a clear reason.
5. **Prediction band**
   - Split-conformal intervals from backtest residuals (default 80%).
6. **Recommendation rule (replaces the fixed 4% threshold)**
   - `HOLD` only if expected gain beats storage cost + shrinkage + interest **and** the lower band supports it.
   - Otherwise `SELL_NOW`. If skill is not above zero: `NO_CLEAR_EDGE` ("decide on cash need").
   - Storage, shrink, interest rates live in an editable config, labelled as estimates.
7. **API and UI**
   - Implement `GET /advice/{commodity}/{market}` exactly as in contract 1.4.
   - Chart component with forecast line, shaded band, and a badge such as "Real data, 6 days old, beats naive by 14%" (numbers must come from the backtest).

### Acceptance criteria

- Endpoint matches the contract; a sample response was published on day 1.
- Synthetic data is never presented as real; badge and `data_source` are correct.
- A commodity with weak skill returns `NO_CLEAR_EDGE` (tested).
- Backtest has unit tests on a small fixed series.
- No accuracy claim anywhere that is not computed.

---

## 5. Workstream M3: Money & Offers (U5 + U2)

**Goal:** Show the farmer what they actually bank per offer and let money move only on verified events.

**Depends on:** M2's `band_low` (optional; fall back to nearest-mandi price) and M1's events (use a dev-only trigger until ready).

### Part A: U5 Net-in-Hand + Fairness Meter + Counter-offer

1. **Units fix (KI-07):** convert and display quintal vs kg consistently; store kg.
2. **Rates config table** (editable, labelled "estimates"): handling, fees, shrink %, vehicle rate per km.
3. **Net per kg** = `offer - transport_per_kg - handling - fees - shrink`.
   - Transport from the recommender's distance x vehicle rate. Populate the unused `estimated_delivery_cost` column.
4. **BATNA** = best of: nearest-mandi modal price net of carrying cost; other pending offers; hold forecast's lower band (from `/advice`, optional).
5. **Fairness cue:** green/amber/red based on where the offer sits against the BATNA range.
6. **Counter-offer:** add a `countered` offer state, **one round only**, pre-filled from the fairness range. Buyer can accept or decline.
7. UI: offer card shows net-in-hand, best alternative, fairness colour, counter button.

### Part B: U2 Milestone Escrow (SIMULATED)

1. States: `unfunded`, `funded`, `released_partial`, `released_full`, `frozen`, `settled`.
2. Default split **30/70** (configurable): partial release at `HUB_VERIFIED`, rest at `DELIVERED`.
3. Implement `escrow.on_event(transaction_id, event_type)` (contract 1.3) and a dev-only trigger endpoint (disabled in production).
4. Use Razorpay **test mode** for the fund step. Keep holds and releases in your own ledger tables (`escrow_*`).
5. Replace the manual "buyer marks as paid" button.
6. Dispute effects (KI-09): `favor_farmer` releases the rest; `favor_buyer` refunds the rest; `mutual` splits by an admin-set ratio; `no_fault` releases. Dispute open -> `frozen`.
7. UI must say "Simulation / test mode" wherever funds are shown.

### Acceptance criteria

- Two offers with different headline prices can rank differently by net-in-hand (tested).
- Counter-offer allows exactly one round.
- Escrow state transitions are tested for each event and each dispute outcome; money totals always reconcile (funded = released + refunded + held).
- Works with stubbed events and with real M1 events.

---

## 6. Workstream M4: Pooling & Experience (U3 + polish)

**Goal:** Let a buyer post a bulk demand and assemble nearby small lots into one deal with one truck and a transparent payout.

**Depends on:** M1's `verified_weight_kg` for final payouts (use a manual field until then).

### Part A: U3 Demand-Pull Pooling

1. Tables: `pool_demands`, `pools`, `pool_members` (lot, committed kg, share).
2. **Optimiser:** choose lots at or near one hub that meet grade and total quantity, minimising landed cost (`price x kg + transport`) within truck capacity from the transporters table. Use OR-Tools or PuLP, **with a greedy fallback**. Show the result is feasible or explain why not.
3. Buyer page: "Post demand" (commodity, grade, kg, district, deadline).
4. Farmer: notification with **one-tap join** at a stated price and deadline.
5. **Editable grade-price differential table** (visible, not hidden logic).
6. **Payout** per farmer = `verified_weight_kg x grade-adjusted price`. Show the formula on screen. Before M1 is ready, use a manual field.
7. Seed realistic **demo lots** marked clearly as demo data.

### Part B: Polish (after Part A is stable)

- Wire the unused Bhashini ReadAloud and TranslateButton components (KI-29).
- Voice-to-draft-lot with schema validation and a **read-back confirmation** before saving.
- Offline draft queue: connect the existing Dexie queue (KI-28) so a farmer can draft a lot offline and sync later.

### Acceptance criteria

- Five demo lots merge into one pool and one truck; each farmer sees their share and the formula.
- Optimiser has tests, including infeasible demand and the greedy fallback.
- Demo data is labelled in UI.
- Offline draft syncs after reconnect.

---

## 7. Build order and priorities

| Phase | What | Notes |
|---|---|---|
| 0 | Foundation fixes (M1 security items, M2 real data key and backfill) | Do first. Honest, secure, reproducible base. |
| 1 | U1 (M1), U4 (M2), U5 (M3 Part A) | Minimum winning slice. Each demos on its own. |
| 2 | U3 (M4), U2 (M3 Part B) | The "Pool, Prove, Pay" wow. |
| 3 | Voice, offline, trust score, pitch deck, demo rehearsal | Polish. |

**If time is short:** ship U1 + U5 + U4 first.

**Stretch ideas (only after the above):** trust score (rating + verified status + dispute outcomes), verified trade passport export, ONDC-style catalogue export mock, forward price lock.

---

## 8. Integration checklist (final day)

- [ ] All stubs replaced with real implementations (advice API, event hook, verified fields).
- [ ] All feature flags documented and set for the demo.
- [ ] `main` passes all backend tests and the frontend builds.
- [ ] No demo admin, plaintext OTP, or committed SQLite in the deployed build.
- [ ] Every synthetic/demo/estimate value is labelled in the UI.
- [ ] 3-minute demo rehearsed end to end.

### 3-minute demo script

1. **0:00** Farmer speaks in Marathi; a lot draft appears and a hub is auto-assigned. Advice card shows a band and a real-data badge.
2. **0:45** Buyer posts a 2,000 kg demand. A pool forms from five lots with one truck. Each farmer sees net-in-hand, fairness colour and best alternative.
3. **1:30** Buyer funds escrow (test mode). Hub operator weighs two lots; one is lighter, so a re-quote is proposed and accepted.
4. **2:15** Delivery confirmed, funds release. Each payout shows its formula.
5. **2:40** Open the proof page (green), edit a database row, refresh (red). Close: *"We do not just show prices, we prove the deal."*

---

## 9. Known limitations (state these honestly in the pitch)

- The hash chain is tamper-evident, not tamper-proof. Roadmap: publish a daily root hash externally.
- Escrow is simulated; real fund holding needs a regulated payment partner under RBI rules.
- Forecast accuracy is unknown until backtested on real data.
- Fee, shrink and transport rates are estimates.
- Competitor comparisons come from public descriptions, not code audits.

---

## 10. Status (agents: update this section when you finish)

| Workstream | Owner | Branch | Flag | Status | Notes |
|---|---|---|---|---|---|
| M1 Trust Core | | | | Not started | |
| M2 Data & AI | | | | Not started | |
| M3 Money & Offers | | | | Not started | |
| M4 Pooling & Experience | | | | Not started | |
