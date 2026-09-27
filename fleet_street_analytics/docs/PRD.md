# Fleet Street Analytics — Product Requirements Document (v0.1, draft)

**Owner:** Fleet Street Analytics (FSA) · **Market:** Ireland (initially) · **Status:** Preliminary design

---

## 1. Summary

A multi-party political analytics platform. It combines generalised public data (census, crime, vehicle, geographic, past results) with canvassing data collected by politicians (TDs), and predicts how small areas will vote. It shows the predictions on a map, for example: *"Main Street (north side): 10 households, 60% predicted to vote Party A."* The main use is making door-to-door canvassing more productive.

## 2. Goals / Non-goals

**Goals**
- Predict party vote share at street / street-segment level (groups of ~10+ households).
- Show predictions on an interactive map so canvassing teams can plan.
- Serve several competing parties from one platform, with strict separation between them.
- GDPR and Irish Data Protection Act 2018 compliance by design. Generalisation happens at every upward data flow.
- FSA builds and owns a master dataset that grows over time, made only of public and generalised data.

**Non-goals (v1)**
- Predicting how an individual or a single household will vote on any shared or aggregated layer.
- Ad targeting, messaging automation or voter contact tools (possible later phases).
- Markets outside Ireland.

## 3. Users & roles

| Role | Description | Access |
|---|---|---|
| **FSA Admin / Analyst** | Fleet Street Analytics staff | Master DB (full). Runs analytics jobs. No direct read access to any Party or TD DB. |
| **Party HQ user** | Staff at a party's national HQ | Their own Party DB and the analysis outputs for their party. No access to the Master DB, other parties, or raw TD data. |
| **TD** (and delegated staff, if approved) | An elected representative or candidate | Their own TD DB (full), plus their party's map and predictions. |
| **Canvasser** (Phase 2) | Volunteer in the field | Read-only map and walk lists for their assigned area. Can submit doorstep results to the TD DB. |

## 4. Data architecture — three tiers

```
 Public sources ──scrape/ingest──▶ ┌────────────────────────┐
                                   │  MASTER DB (FSA-owned) │  generalised, area-level only
                                   └───────────┬────────────┘
                                               │ read (analytics engine only)
                                               ▼
                                   ┌────────────────────────┐
                                   │   ANALYTICS ENGINE     │──▶ predictions → that party only
                                   └───────────▲────────────┘
                                               │ read (one party per job)
                                   ┌───────────┴────────────┐
                                   │  PARTY HQ DB (1/party) │  generalised TD data
                                   └───────────▲────────────┘
                                               │ generalised upload (automatic)
                                   ┌───────────┴────────────┐
                                   │   TD DB (1 per TD)     │  address/person-level detail
                                   └────────────────────────┘
```

### 4.1 Master DB (FSA)
- **Contents:** public and scraped data, generalised to area level. Examples: CSO census Small Area statistics, Garda recorded crime by station or area, area-level vehicle registration statistics, Eircode routing-key geography, past election results and tallies, property price register aggregates, deprivation indices, OSM street geometry.
- **Rule:** stores no personal data about identifiable individuals or households. Any record that could identify one is aggregated before it is stored.
- **Inbound:** public sources only. **No data from Party or TD DBs flows into the Master DB.**
- **Ownership:** FSA is controller and owner. This is FSA's long-term asset.

### 4.2 Party HQ DB (one per party)
- **Contents:** generalised uploads from that party's TD DBs, plus party-level inputs such as target constituencies and campaign priorities.
- **Access:** only that party. It cannot read the Master DB or any other party's DB.
- **Outbound:** read by the analytics engine for that party's jobs only. Never copied into the Master DB.

### 4.3 TD DB (one per TD)
- **Contents:** address- and person-level canvass data entered by the TD. It follows a **defined schema**: address/Eircode, number of occupants, gender(s), age band, political leaning (scale plus party), issues raised, contact date, consent/opt-out flags, and a limited notes field.
- **Access:** that TD only, plus delegated staff if the TD grants it. The platform must enforce this, including against FSA staff (see 6.3).
- **Outbound:** an automatic generalisation pipeline pushes aggregates to the Party DB (see 5).

## 5. Generalisation rules (upward flows)

| Rule | TD → Party | Public → Master |
|---|---|---|
| Remove direct identifiers (name, phone, email, exact house number) | ✅ | ✅ |
| Aggregate to a street segment or Small Area | ✅ | ✅ |
| **Minimum group size k** (default **10 households**, configurable, never below 5). Groups below k are merged with neighbours or suppressed | ✅ | ✅ |
| Count political leaning as a distribution, never per household | ✅ | n/a |
| Coarsen quasi-identifiers (age → bands, dates → month) | ✅ | ✅ |
| Keep a record of each run (what was sent, when, k used) | ✅ | ✅ |

Generalisation is a **deterministic, tested pipeline** that runs before any cross-tier write. It is never a manual export.

## 6. Functional requirements

### 6.1 Ingestion (Master)
- F1. Pluggable connectors per source: scheduled, versioned, with source licence/ToS metadata recorded.
- F2. Raw scraped data stays in a short-lived quarantine store. It is generalised, then deleted within N days (default 7).
- F3. Geocode everything to a common spatial key: street segment ID and CSO Small Area ID.

### 6.2 TD data entry
- F4. Web/mobile form and CSV import against the TD schema. Invalid or out-of-schema fields are rejected.
- F5. Per-record data-subject controls: record opt-out/objection, erase the record, export the record (SAR).
- F6. Erasure in a TD DB triggers a regeneration of the affected Party DB aggregates.

### 6.3 Access control & tenancy
- F7. Isolation per party and per TD, enforced at the database layer (separate databases or row-level security plus per-tenant encryption keys), not only in application code.
- F8. The analytics engine is the only component that reads both Master and Party data. It runs as an isolated job per party, and its outputs are written only to that party's space.
- F9. Every read or write of TD and Party data goes to an immutable audit log.

### 6.4 Analytics
- F10. Model: area-level vote-share prediction. Features come from the Master DB. Party DB aggregates act as priors or calibration (e.g. hierarchical / small-area estimation, tuned against past tallies).
- F11. Output per street segment: household count, predicted share per party, confidence interval, data freshness.
- F12. Outputs follow the same k-threshold as uploads. No displayed unit may have fewer than k households.
- F13. Backtest each model version against the most recent election tallies and report accuracy per constituency.

### 6.5 Map & canvassing UI
- F14. Interactive map with a choropleth by predicted share for a chosen party, and drill-down from constituency → Small Area → street segment.
- F15. Tooltip, e.g. "Street X: 12 households · Party A 58% (±9%)".
- F16. Filters for persuadable/swing segments, not-yet-canvassed segments, and turnout likelihood.
- F17. Export a walk list per street segment with no per-household predictions. A TD's own layer may also show their own raw canvass records.

## 7. Non-functional requirements
- **Compliance:** a DPIA (Art. 35) before launch; a Record of Processing Activities; a DPO appointed; a data processing agreement between FSA and each party (FSA acts as **processor** for Party/TD data and **controller** for Master data); a published privacy notice.
- **Security:** encryption at rest and in transit, per-tenant keys, MFA for all users, least privilege, annual pen-test.
- **Hosting:** EU region only.
- **Retention:** TD data is reviewed and purged after each electoral cycle unless there is a lawful reason to keep it. Party aggregates are regenerated, not accumulated forever.
- **Performance:** map tiles load in under 2 s for a constituency. A prediction job for one constituency finishes in under 15 min.

## 8. Success metrics
- Prediction error (MAE of party share) vs actual tallies at box/Small Area level.
- Canvasser contact rate uplift (supporters and persuadables reached per hour) vs a baseline.
- Zero cross-tenant data access incidents. Zero k-threshold violations in outputs (verified by automated checks).

## 9. Phasing
1. **MVP:** Master ingestion (CSO SAPS, election tallies, geography), one party, TD entry, the generalisation pipeline, a basic model, a map.
2. **Multi-party:** tenant isolation hardening, audit, DPIA sign-off, onboarding of a second party.
3. **Field:** canvasser mobile app, walk lists, offline mode.
4. **Later:** turnout modelling, campaign resource allocation, trend tracking between elections.

## 10. Risks & open questions
1. **Special-category data.** Political opinions fall under GDPR Art. 9. Parties and candidates rely on the Irish DPA 2018 electoral-activities provisions. **FSA itself likely cannot**, so this is why the Master DB must hold no political-opinion personal data. Legal counsel to confirm.
2. **Vehicle registration data** is not freely public in Ireland. Only licensed or area-level aggregate statistics may be used. Scraping individual vehicle records is out of scope.
3. **Eircode:** full address-level Eircode data (ECAD) needs a licence. Routing keys are public.
4. **Electoral register:** its use is restricted to electoral purposes. Confirm whether it may be joined with other data and by whom (probably TD/party tier only, never Master).
5. **Scraping:** each source's ToS and licence must allow the use. Record this per connector (F1).
6. **Small-area inference risk:** even predictions at k=10 are inferences about identifiable households. Confirm the k value and the legal basis for showing predictions to canvassers.
7. **Party → Master flow:** the original brief mentions generalised uploads into Master and then rules out party data. This PRD takes the **stricter reading (no Party/TD data reaches Master)**. Confirm.
8. **Delegated TD access:** can staff and canvassers write to or read a TD DB? The default is TD-granted, per-user, and audited.
9. **Model transparency:** what model explanations should parties see?
