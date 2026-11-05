# Data-Quality Report — TGV punctuality by route (`regularite-mensuelle-tgv-aqst`)

**Group:** Sarah BENADDAD, Marion GOMES DE SOUSA, Farida SAID · **Date:** 2026-11-05 · **Notebook:** `project_04_tgv_punctuality.ipynb` · **Commit:** `<git short hash>`
**Clean-data fingerprint:** `380cf7b65fd6`

---

## 1. Source & provenance
| | |
|---|---|
| Business question | Do long TGV routes have **more** delays, or just **longer** ones? |
| Source(s) & endpoint / URL | SNCF open data, Opendatasoft Explore API v2.1 — `ressources.data.sncf.com/api/explore/v2.1/catalog/datasets/regularite-mensuelle-tgv-aqst/records` |
| Collection method & date range | REST API, paginated + cached (`requests-cache`), filter `date >= 2021-01-01` → 2021-01 to 2026-09, collected 2026-11-04 |
| Licence, ToS constraints | ODbL (attribution, derived data stays ODbL). SNCF warns: rows can't be summed (a train can count in several routes) |
| Unit of observation | 1 route (`gare_depart` → `gare_arrivee`) × 1 `service` × 1 month |
| Final size | raw 8,395 × 26 → clean **8,353 × 12** |

## 2. Schema (the contract)
Notebook §3.4 (raw) and §6 (clean). Key checks:
- `nb_train_*` — `>= 0` — negative counts
- `nb_annulation <= nb_train_prevu` — more cancellations than planned trains
- `nb_train_retard_arrivee <= trains run` — more late trains than trains run
- `sup_60 <= sup_30 <= sup_15` — delay tiers not nested
- `sup_15 <= nb_train_retard_arrivee` — breaks the AQST definition (max threshold 15 min)
- unique (`date`, `service`, `gare_depart`, `gare_arrivee`) — duplicated route-months

Raw data: **398 rows in violation** (355 tiers not nested, incl. 43 negative `sup_30`), all in Q1 2025. Clean data: **passes**.

## 3. Missingness
| Column | % missing | Mechanism | Evidence | Decision |
|---|---|---|---|---|
| `nb_train_retard_sup_15/30/60` | 4.3% (363) | MAR (date) | 100% in 2025-01 to 03, 0% elsewhere (§4.3) | NaN, no imputation |
| `retard_moyen_arrivee` | 0.4% (37) | MAR (service / route) | 28/37 International: Barcelona, Madrid, Stuttgart (§4.4) | NaN, no imputation |
| Absent route-months (rows) | 341 / 8,694 | structural + MNAR | not seasonal (never the same month twice); routes not run, relabelled, or suspended (§4.3) | no imputation |

## 4. Duplicates & outliers — decision table
| Column | Issue | n rows | Decision | Why |
|---|---|---|---|---|
| 26 → 12 columns | column selection | 8,395 | keep key, duration, arrival counts, mean delay, tiers | departure, causes, comments don't serve the question |
| all | exact duplicates | 0 | nothing to do | — |
| `service` | same route-month as National AND International (Jul–Sep 2025) | 42 | drop International row | route is National every other month |
| `gare_*` | renamed / misspelled stations | 0 | nothing to do | difflib + manual check of 58 stations |
| duration, counts, mean delay | impossible values | 0 | nothing to do | all rules return 0 |
| `sup_15/30/60` | negative / not nested (Q1 2025) | 363 | NaN for the quarter, keep rows | systematic publication error |
| other columns, Q1 2025 | affected too? | 363 | keep | no jump vs neighbouring months / Q1 2024, Q1 2026 |
| `service` | Barcelona relabelled International → National (Oct 2025) | 69 | keep, ignore service for this route | same route, all months present |
| absent rows | Madrid–Marseille only 2022-04 to 12 | 120 | no imputation | no train ran |
| absent rows | Tourcoing → Marseille 1 month / 69 | 68 | keep, no return assumed | one-way route, return runs on another branch |
| absent rows | one-off gaps (Italie, Zurich, Bordeaux–Tourcoing) | 15 | no imputation, flagged | suspensions, MNAR |
| `retard_moyen_arrivee` | high outliers > 90 min | 45 | keep, weight by late trains | real incidents, often 1–2 trains |
| `retard_moyen_arrivee` | 0 with late trains | 37 | NaN | disguised missing |
| `retard_moyen_arrivee` | 0 with no late train | 75 | keep | logical |
| `retard_moyen_arrivee` | low outliers > 0 (< 13.6 min) | 35 | keep | short routes (median 75 min), 5-min threshold |

## 5. Key findings
1. **Delays are right-skewed** — Fig. 5.1 — median 13% late trains, 36 min mean delay, tail to 300 min → use medians, weight by trains.
2. **AQST view: long routes late as often but longer** — Fig. 5.2 — Spearman 0.10 (frequency) vs 0.86 (magnitude) → misleading, threshold rises with trip length.
3. **Same threshold (> 15 min): long routes have more delays, not longer ones** — Fig. 5.3 — 6.4% / 10.2% / 14.9% of trains > 15 min late; severity split ≈ 50/30/20% for all classes → use one threshold across routes.

## 6. What cannot be trusted ⚠️
- **`nb_train_retard_arrivee`, `retard_moyen_arrivee` across routes of different length** — AQST threshold is 5 / 10 / 15 min by trip length: counts deflated, means inflated on long routes.
- **`nb_train_retard_sup_*` in Q1 2025** — publication error, set to NaN.
- **`retard_moyen_arrivee` on Barcelona, Madrid, Stuttgart** — fake zeros + means on 1–2 trains.
- **`service`** — double publication Jul–Sep 2025, Barcelona relabelled Oct 2025: not a stable feature.
- **Absent months on Italie, Zurich, Bordeaux–Tourcoing** — suspensions (MNAR): delays likely underestimated.
- **H1 2021** — COVID restrictions: atypical volumes.

## 7. Hypotheses for modelling
- **H1** — share > 15 min ↑ with duration — National routes — Spearman + Kruskal–Wallis.
- **H2** — share > 60 min (among > 15) equal across duration classes — all routes, excl. Q1 2025 — chi-square.
- **H3** — International > National for share > 15 min — routes > 3h — Mann–Whitney.
- **H4** — more late trains in July–August — all route-months — Mann–Whitney.
- **H5** — cancellations ↑ ↔ late share ↑ in the same route-month — all route-months — Spearman.

## 8. How to reproduce
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt jupyter
jupyter nbconvert --to notebook --execute project_04_tgv_punctuality.ipynb
```
Expected fingerprint: `380cf7b65fd6` (with the API cache; a fresh collection can differ, data as of 2026-11-04).
