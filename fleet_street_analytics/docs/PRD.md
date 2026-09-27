# Fleet Street Analytics — Product Requirements Document (v0.1, draft)

**Owner:** Fleet Street Analytics (FSA) · **Market:** Ireland (initially) · **Status:** Preliminary design

---

## 1. Summary

A multi-party political analytics platform. It combines generalised public data (census, crime, vehicle, geographic, past results) with canvassing data collected by politicians (TDs), and predicts how small areas will vote. It shows the predictions on a map, for example: *"Main Street (north side): 10 households, 60% predicted to vote Party A."* The main use is making door-to-door canvassing more productive.

## 2. Goals / Non-goals

**Goals**
- Predict party vote share at street / street-segment level.
- Show predictions on an interactive map so canvassing teams can plan.
- Serve several competing parties from one platform, with strict separation between them.
- GDPR and Irish Data Protection Act 2018 compliance for the Master DB and the platform. The Master DB holds generalised data only. The generalisation rules for TD → Party uploads are configurable and not yet defined.
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
- **Contents:** address- and person-level canvass data entered by the TD. It has core fields (address/Eircode, number of occupants, gender(s), age, political leaning, issues raised, contact date, notes), and TDs can add their own **custom fields**. The TD tier puts no generalisation or data-minimisation constraints on what is stored.
- **Access:** that TD only, plus delegated staff if the TD grants it. The platform must enforce this, including against FSA staff (see 6.3).
- **Outbound:** an automatic pipeline sends data to the Party DB, removing and generalising the fields that the rules say to (see 5). Those rules are **to be defined later**.

## 5. Generalisation rules (upward flows)

**Public → Master (fixed):** remove direct identifiers, aggregate to a street segment or Small Area, and coarsen quasi-identifiers (age → bands, dates → month). No personal data is stored in the Master DB.

**TD → Party (configurable, not yet defined):** the pipeline will delete some fields and generalise others before data reaches the Party DB. Which fields, and how, is **left open for now**. The pipeline must be built as a rule-driven framework so these rules can be set later without code changes.

**Minimum group size:** a **configurable setting with no default**. It is unset until the product owner chooses a value. When it is set, it applies to Party uploads and to shared prediction outputs. Groups below the minimum are merged with neighbours or suppressed.

Every pipeline run is logged: what was sent, when, and which rule set was used. Uploads always go through the pipeline and are never manual exports.

## 6. Functional requirements

### 6.1 Ingestion (Master)
- F1. Pluggable connectors per source: scheduled, versioned, with source licence/ToS metadata recorded.
- F2. Raw scraped data stays in a short-lived quarantine store. It is generalised, then deleted within N days (default 7).
- F3. Geocode everything to a common spatial key: street segment ID and CSO Small Area ID.

### 6.2 TD data entry
- F4. Web/mobile form and CSV import. Core fields are validated, and TDs can define and fill custom fields.
- F5. Lightweight per-record tools: a consent flag with the date it was given, and one-click export and erase of a record.
- F6. Erasure in a TD DB triggers a regeneration of the affected Party DB aggregates.

### 6.3 Access control & tenancy
- F7. Isolation per party and per TD, enforced at the database layer (separate databases or row-level security plus per-tenant encryption keys), not only in application code.
- F8. The analytics engine is the only component that reads both Master and Party data. It runs as an isolated job per party, and its outputs are written only to that party's space.
- F9. Every read or write of TD and Party data goes to an immutable audit log.

### 6.4 Analytics
- F10. Model: area-level vote-share prediction. Features come from the Master DB. Party DB aggregates act as priors or calibration (e.g. hierarchical / small-area estimation, tuned against past tallies).
- F11. Output per street segment: household count, predicted share per party, confidence interval, data freshness.
- F12. If a minimum group size is configured, outputs respect it. No displayed unit may then have fewer households than the minimum.
- F13. Backtest each model version against the most recent election tallies and report accuracy per constituency.

### 6.5 Map & canvassing UI
- F14. Interactive map with a choropleth by predicted share for a chosen party, and drill-down from constituency → Small Area → street segment.
- F15. Tooltip, e.g. "Street X: 12 households · Party A 58% (±9%)".
- F16. Filters for persuadable/swing segments, not-yet-canvassed segments, and turnout likelihood.
- F17. Export a walk list per street segment. A TD's own layer may also show their own raw canvass records.

## 7. Non-functional requirements
- **Compliance:** a DPIA (Art. 35) before launch; a Record of Processing Activities; a DPO appointed; a data processing agreement between FSA and each party (FSA acts as **processor** for Party/TD data and **controller** for Master data); a published privacy notice.
- **Security:** encryption at rest and in transit, per-tenant keys, MFA for all users, least privilege, annual pen-test.
- **Hosting:** EU region only.
- **Retention:** Master raw data per F2. TD and Party retention periods are configurable and to be decided.
- **Performance:** map tiles load in under 2 s for a constituency. A prediction job for one constituency finishes in under 15 min.

## 8. Success metrics
- Prediction error (MAE of party share) vs actual tallies at box/Small Area level.
- Canvasser contact rate uplift (supporters and persuadables reached per hour) vs a baseline.
- Zero cross-tenant data access incidents. Zero violations of the configured minimum group size, once one is set (verified by automated checks).

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
6. **Minimum group size:** not yet chosen (see 5). Street-level predictions for very small groups are inferences about identifiable households, so pick the value before launch.
7. **Party → Master flow:** the original brief mentions generalised uploads into Master and then rules out party data. This PRD takes the **stricter reading (no Party/TD data reaches Master)**. Confirm.
8. **Delegated TD access:** can staff and canvassers write to or read a TD DB? The default is TD-granted, per-user, and audited.
9. **Model transparency:** what model explanations should parties see?
10. **TD → Party rules:** which fields are deleted or generalised on upload (see 5).
11. **TD-tier lawful basis:** canvass data is given willingly, so the likely basis is consent (explicit consent for political opinions, Art. 9(2)(a)) and/or the DPA 2018 electoral provisions. GDPR still applies to the TD DB, with the TD as controller. The practical need is to record consent and be able to export or erase a record on request (F5). Counsel to confirm.
