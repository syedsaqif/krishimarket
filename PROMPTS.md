# Antigravity Prompts (copy and paste)

> Put `improvement.md` in the repo root and `CONTRACTS.md` + this file in `docs/`.
> Order: **one person runs Prompt 0**, merges it to `main`, then all 4 members branch from `main` and run their own prompt.

---

## Prompt 0: Scaffold (ONE person, once, before anyone else starts)

```
Read improvement.md and docs/CONTRACTS.md fully. Do NOT implement any feature yet.
Your job is to create the shared scaffold so four people can work in parallel.

First inspect the repo (framework, ORM, router registration, auth dependency, migration
tool, test setup, frontend router and i18n files) and write a short plan.

Then create:
1. shared/events.py with the EventType enum exactly as in CONTRACTS.md section 3.1.
2. ledger/service.py containing ONLY the record_event() signature and docstring from
   section 4.1, raising NotImplementedError. Include the canonical() and compute_hash()
   functions from section 4.3 (these are final, not stubs) with unit tests.
3. escrow/service.py containing on_event() as a safe no-op that logs and returns.
4. Empty router files with one placeholder route each, registered in main with their
   feature flags (all default off): routers/ledger.py, routers/hub.py, routers/advice.py,
   routers/offers.py, routers/escrow.py, routers/pooling.py.
5. A feature-flag helper reading the flags in section 12, plus .env.example entries.
6. GET /advice/onion/Lasalgaon returning the exact sample JSON from section 6 when
   ADVICE_USE_SAMPLE=true.
7. Empty frontend folders from section 9 with a placeholder index file each, and the
   i18n comment blocks from section 11 in en, hi and mr.
8. Add *.sqlite, *.db to .gitignore if missing (do not delete tracked files yet).

Do not change existing behaviour. Run the full existing test suite and show it passes.
Commit in small commits on a branch called scaffold.
```

---

## Prompt M1: Trust Core

```
Read improvement.md and docs/CONTRACTS.md fully. You are implementing Workstream M1
(Trust Core) ONLY. Do not touch other workstreams' files (ownership: CONTRACTS.md
sections 9 and 10). Shared files: smallest possible edit.

Branch: m1-trust-core. Flag: FEATURE_PROOF_LEDGER.

Step 1: inspect the repo and write a plan listing files to create or change. Wait for
no approval, but keep the plan in your first message.

Step 2: implement in this order, with tests for each step and a commit per step:
 a. hub_operator role (admin-created), role-based permission matrix (CONTRACTS section 2)
    on delivery events. Wrong role returns 403 FORBIDDEN_EVENT. (KI-03)
 b. Migration adding the transactions columns in section 4.5.
 c. ledger_deal_events table and the real record_event() per sections 4.1 to 4.3, calling
    escrow.on_event in a guarded way. Payload validation per section 3.2.
 d. Hub endpoints: checkin and verify per section 4.4, using
    delivery_state_machine.transition(). Closes KI-02.
 e. Variance rule and re-quote flow (REQUOTE_PROPOSED, accept/reject, rejection opens a
    dispute with evidence attached). Tolerance from HUB_VARIANCE_TOLERANCE_PCT.
 f. GET /deals/{id}/proof recompute-and-verify endpoint, plus frontend proof page with
    QR code on the transaction page, green when valid and red when broken, and the
    "Tamper-evident, not tamper-proof" notice.
 g. Hub operator frontend page (/hub): list assigned transactions, check-in form with
    photo, weight, grade, and the verify action.
 h. Security fixes: KI-05 authenticated ID document download route (remove the
    unauthenticated mount), KI-15 fail-closed production defaults (no demo admin, no
    plaintext OTP; unset ENVIRONMENT behaves as production), KI-14 missing icon/og files,
    KI-16/24 remove committed SQLite files from git tracking and pin requirements.

Constraints: follow the hash algorithm in section 4.3 exactly. Only ledger/service.py
inserts into ledger_deal_events. Add i18n keys (en, hi, mr) under the M1 block.

Required tests: wrong-role 403s; full dispatch to delivered path; variance above and
below tolerance; tampering with any ledger row makes /proof return valid:false with the
right first_broken_seq; unauthenticated ID download rejected.

Finish by updating the M1 row in improvement.md section 10 and summarising what is
done, what is not, and any contract change you needed.
```

---

## Prompt M2: Data & AI

```
Read improvement.md and docs/CONTRACTS.md fully. You are implementing Workstream M2
(Data & AI, calibrated advice) ONLY. Do not touch other workstreams' files. Shared
files: smallest possible edit.

Branch: m2-advice. Flags: FEATURE_ADVICE, ADVICE_USE_SAMPLE.

Step 1: inspect the repo (price_records model, seeding code, current forecast and
recommender code) and write a plan in your first message.

Step 2: implement in this order, tests and a commit per step:
 a. Keep GET /advice/{commodity}/{market} returning the sample JSON when
    ADVICE_USE_SAMPLE=true (already scaffolded). Never break this until the end.
 b. Data honesty (KI-01): add a source column (agmarknet / synthetic) to price records
    via an advice_-prefixed migration, label existing synthetic rows, disable
    auto-seed in production, and write an ingestion command that backfills real data
    from data.gov.in using an API key from env (DATA_GOV_IN_API_KEY). If no key is
    available, do not fabricate data; make the command fail with a clear message.
 c. Aggregation (KI-06): one series per commodity + market per day using the median
    modal price. Convert quintal to kg (store per kg).
 d. Models in services/forecast/: naive, seasonal-naive, ETS/SARIMA (statsmodels),
    gradient boosting with calendar and lag features. Optional Prophet or Chronos only
    if it installs cleanly; otherwise skip.
 e. Rolling-origin backtest as a management command, storing MAE per model and
    skill = 1 - MAE / MAE_naive in advice_backtests. Select the best model per
    commodity. Require a minimum history length (config); below it return skill null.
 f. Split-conformal prediction band (default 80%) from backtest residuals.
 g. Cost-aware recommendation: HOLD only if expected gain beats storage + shrinkage +
    interest AND the lower band supports it; SELL_NOW otherwise; NO_CLEAR_EDGE when
    skill is null or not above zero. Rates in an editable config, labelled estimates.
    Add the MSP floor where known and never advise selling below it.
 h. Real GET /advice/{commodity}/{market} matching CONTRACTS section 6 exactly, with
    data_source, data_age_days and a stale-data indicator.
 i. Frontend AdviceCard in src/features/advice: forecast line, shaded band, data badge
    (real/synthetic/mixed), skill text, stale note, MSP marker. i18n under M2 block.

Constraints: never display any accuracy number that the backtest did not compute.
Unit-test the backtest and skill calculation on a small fixed series, and test that a
weak series returns NO_CLEAR_EDGE.

Finish by updating the M2 row in improvement.md section 10 and listing what is done,
what is not, and which data you could and could not backfill.
```

---

## Prompt M3: Money & Offers

```
Read improvement.md and docs/CONTRACTS.md fully. You are implementing Workstream M3
(Money & Offers: net-in-hand, counter-offer, simulated escrow) ONLY. Do not touch
other workstreams' files. Shared files: smallest possible edit.

Branch: m3-money-offers. Flags: FEATURE_OFFER_NET, FEATURE_ESCROW_SIM, FEATURE_DEV_TOOLS.

Step 1: inspect the repo (offers, transactions, disputes, recommender, transporters)
and write a plan in your first message.

Step 2: Part A, offers (do first), tests and a commit per step:
 a. KI-07: consistent quintal/kg handling; store kg, convert for display only.
 b. offer_rate_config table (admin-editable, is_estimate default true) with keys in
    CONTRACTS section 7.
 c. GET /offers/{id}/net exactly as specified. Transport from the recommender's
    distance x vehicle rate; populate the unused estimated_delivery_cost column.
 d. BATNA: nearest-mandi modal price net of carrying cost, other pending offers, and
    /advice band_low when FEATURE_ADVICE is on. If the advice call fails, silently fall
    back; never fail the endpoint.
 e. Fairness colour per section 7 thresholds.
 f. Counter-offer: countered state, exactly one round (second attempt returns 409
    COUNTER_ALREADY_USED), buyer accept/decline.
 g. Frontend in src/features/offers: offer card showing net-in-hand, best alternative,
    fairness colour, counter button. Label all rates "estimate". i18n under M3 block.

Step 3: Part B, escrow (simulated):
 a. escrow_accounts and escrow_movements tables per CONTRACTS section 5.
 b. Implement escrow.on_event() (idempotent, never raises into the caller) with the
    event table in section 5, replacing the scaffold no-op. Partial release percent from
    ESCROW_PARTIAL_PCT (default 30).
 c. POST /escrow/{id}/fund using Razorpay TEST mode when keys exist, otherwise a mock
    that behaves the same. Replace the "buyer marks as paid" action.
 d. Dispute outcomes (KI-09): favor_farmer, favor_buyer, mutual (split_farmer_pct),
    no_fault.
 e. Dev-only POST /dev/escrow/{id}/trigger, returning 404 in production, so you can
    test without M1.
 f. Frontend in src/features/escrow: status timeline and movements. Every escrow screen
    shows "Simulation / test mode".

Constraints: invariant funded = released + refunded + held must hold after every
operation; test it for every event and dispute outcome, and test idempotency (same event
twice changes nothing). Never claim real fund holding anywhere in code or UI.

Finish by updating the M3 row in improvement.md section 10 with what is done, what is
not, and whether it was tested against real M1 events or only the dev trigger.
```

---

## Prompt M4: Pooling & Experience

```
Read improvement.md and docs/CONTRACTS.md fully. You are implementing Workstream M4
(Demand-Pull Pooling plus polish) ONLY. Do not touch other workstreams' files. Shared
files: smallest possible edit.

Branch: m4-pooling. Flags: FEATURE_POOLING, FEATURE_VOICE.

Step 1: inspect the repo (lots, hubs, transporters, notifications, recommender's
consolidated_truck rule, Bhashini components, Dexie queue) and write a plan in your first
message.

Step 2: Part A, pooling, tests and a commit per step:
 a. Migration for pool_demands, pool_pools, pool_members, pool_grade_prices per
    CONTRACTS section 8. Seed an editable grade multiplier table.
 b. Optimiser service: choose lots at or near one hub meeting grade and quantity,
    minimising landed cost (price x kg + transport) within truck capacity from the
    transporters table. Use PuLP or OR-Tools with a 2-second limit and a greedy
    fallback. Return a clear infeasible reason when it cannot meet the demand.
 c. Endpoints per section 8: post demand, assemble, join, get pool, payouts. Payout =
    weight_used_kg x price_per_kg x grade_multiplier, with weight source order
    verified, manual, committed, and weight_source returned in the response.
 d. Notifications: farmers get a one-tap join with stated price and deadline.
 e. Seed script creating realistic demo lots and farmers, every row is_demo = true and
    labelled "Demo data" in the UI.
 f. Frontend in src/features/pooling: buyer "Post demand" page, farmer join card,
    pool detail with members, shares, and the payout formula displayed in plain
    language, plus the editable grade-price table (admin). i18n under M4 block.

Step 3: Part B, polish (only after Part A passes tests):
 a. Wire the unused Bhashini ReadAloud and TranslateButton components (KI-29) behind
    FEATURE_VOICE.
 b. Voice-to-draft-lot with schema validation and a read-back confirmation before
    saving.
 c. Connect the existing Dexie queue (KI-28) so a lot can be drafted offline and
    synced later.

Constraints: all demo data clearly marked; the optimiser must have tests for a feasible
case, an infeasible case, and the greedy fallback; never block on M1, use
manual_verified_weight_kg until verified_weight_kg exists.

Finish by updating the M4 row in improvement.md section 10 with what is done and what is
not.
```

---

## Prompt I: Integration (run on `main` after all merges)

```
Read improvement.md and docs/CONTRACTS.md. All four workstreams are merged. Your job is
integration only; do not add features.

1. Find every remaining stub from CONTRACTS section 13 and replace it with the real
   implementation in the same change: dev escrow trigger paths in normal flows, the
   manual_verified_weight_kg fallback ordering, ADVICE_USE_SAMPLE usage.
2. Verify the end-to-end demo path from improvement.md section 8 works through the real
   endpoints: advice card, demand and pool, offer net-in-hand and counter, escrow fund,
   hub check-in and verify with a variance re-quote, delivered, payouts, proof page
   valid, then a tampered row turns it invalid.
3. Write an integration test that exercises that path against a fresh test database.
4. Check the final checklist in improvement.md section 8. For each unchecked item, fix
   it or report exactly why not.
5. Confirm all feature flags default off in production config and on in the demo
   environment file. Report any synthetic, demo or estimate value that is not labelled
   in the UI.

Report findings as a list of fixed and unfixed items.
```

---

## Prompt R: Review (any member, before opening a PR)

```
Review my branch against docs/CONTRACTS.md and the acceptance criteria for my workstream
in improvement.md. Check specifically: (1) every API shape matches the contract exactly,
(2) I only edited files my workstream owns, plus minimal shared-file edits, (3) nothing
writes to ledger_deal_events except ledger/service.py, (4) flags work both on and off,
(5) i18n keys exist in en, hi and mr, (6) no secrets, SQLite files or unlabelled
synthetic/estimate values, (7) tests cover the failure cases, not only the happy path.
List problems by severity. Do not fix them until I confirm.
```
