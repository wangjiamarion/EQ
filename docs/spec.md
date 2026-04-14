# Satellite Early-Warning of California Fuel Shortage from Hormuz Disruption

**Status:** Draft v0.1 — 2026-04-14
**Branch:** `claude/hormuz-disruption-spec-512TK`
**Primary deliverable:** Research paper + reproducible code, with the data layer designed to support a later public dashboard.

---

## 1. Research question

Can open-source satellite observations of tanker traffic and refinery tank-farm fill levels detect a California fuel shortage induced by the 2026 Hormuz Strait disruption *before* it appears in official EIA statistics, and can the lead time be quantified?

The hypothesis is that a shock propagates through a physically ordered chain:

```
Hormuz disruption (t=0, 2026-02-28)
  └─► Tanker calls at San Pedro Bay decline      (AIS, lead ~1–7 d)
        └─► Refinery tank-farm roofs fall        (S-1 SAR, lead ~6–21 d)
              └─► EIA PADD 5 crude stocks fall   (official, Wednesday, 1-week lag)
                    └─► Gasoline retail price spike (endpoint, follow-on paper)
```

The **claim** is (a) the ordering holds, (b) the satellite signals precede the official series by a statistically measurable margin, and (c) the SAR tank-farm signal is independently informative after conditioning on the AIS signal.

## 2. Scope (v1)

**In scope:**
- **Port:** San Pedro Bay complex (Los Angeles + Long Beach).
- **Refineries (5):** Richmond (Chevron), Torrance (Valero/PBF), El Segundo (Chevron), Carson (Marathon), Benicia (Valero, closing). All have large crude-service external-floating-roof tank farms resolvable in Sentinel-1.
- **Baseline window:** 2025-01-01 → 2026-02-27 (≈14 months pre-shock).
- **Treatment window:** 2026-02-28 → present (rolling; ~6 weeks as of 2026-04-14).
- **Satellite streams (core chain only):**
  - AIS tanker arrivals via IMF PortWatch (primary) and MarineTraffic free tier (cross-check).
  - Sentinel-1 C-band SAR IW GRD, VV+VH, descending + ascending passes over each refinery.
- **Ground truth:** EIA Weekly Petroleum Status Report (PADD 5 crude stocks, refinery gross inputs, percent utilization).

**Explicitly out of scope for v1** (revisit in v2):
- Sentinel-2 port container yard occupancy.
- Sentinel-3 SLSTR refinery thermal/flaring.
- Retail price endpoint (CPI, GasBuddy).
- Non-CA West Coast refineries.
- Multi-shock panel (Russia 2022, Red Sea 2024).

## 3. Identification strategy

Treatment is the Hormuz disruption with a known start date (2026-02-28). The counterfactual is the pre-shock baseline at the same refinery under seasonal controls.

Primary specifications:

1. **Event-study on tank-farm fill:** `tank_fill_it = α_i + δ_t + Σ_k β_k · 1[t = T* + k] + ε_it`, where `i` indexes tanks, `t` is the SAR observation date, `T*` is the shock start, and `β_k` recovers the dynamic response. Cluster SE by refinery complex.
2. **Lead-lag regression vs. EIA:** regress weekly EIA PADD 5 crude stocks on lagged satellite aggregates (AIS weekly-tanker-tonnage, SAR weekly-mean-fill), test whether satellite coefficients remain significant with EIA's own AR lags included. Granger-style F-test on the satellite block.
3. **Decomposition test:** after conditioning on AIS tanker tonnage, does the SAR tank-farm signal add independent explanatory power for EIA stocks? This is the paper's core novelty claim.

Seasonality controls: month fixed effects, California refinery maintenance calendar (proxy via 2024 baseline), and the Benicia closure schedule.

## 4. Data products (what gets built)

The pipeline produces these primary tables. All are keyed on ISO date in America/Los_Angeles.

| Product | Grain | Columns | Source |
| --- | --- | --- | --- |
| `ais_calls_daily.parquet` | port × day | crude_tanker_calls, mean_dwt, anchored_hours, unloading_hours | PortWatch + MarineTraffic |
| `tank_inventory_obs.parquet` | tank × orbit-pass | tank_id, acquisition_utc, orbit, roof_height_m, volume_bbl, sigma_height_m, qc_flag | Sentinel-1 SAR |
| `refinery_fill_weekly.parquet` | refinery × ISO-week | mean_fill_frac, n_obs, delta_vs_baseline | aggregated from above |
| `eia_padd5_weekly.parquet` | PADD5 × week | crude_stocks_kbbl, refinery_gross_inputs_kbd, percent_util | EIA v2 API |

Tank footprint masks (`tank_masks.geojson`) are a one-time asset: a hand-digitized/GEE-assisted polygon set with per-tank diameter and known crude-service flag. Building this mask is the single most error-prone step and gates the rest of the SAR work — see §6.

## 5. Technical architecture

```
┌─────────────────────────┐
│  ingest/                │   Python 3.11 + GEE Python API
│   ├─ ais_portwatch.py   │     PortWatch CSV snapshots (daily cron)
│   ├─ s1_tank_extract.py │     GEE task queue, one task per orbit-refinery-date
│   └─ eia_fetch.py       │     EIA v2 REST, weekly
└──────────┬──────────────┘
           ▼
┌─────────────────────────┐
│  process/               │
│   ├─ roof_height.py     │   SAR layover geometry → height estimate
│   ├─ geometry.py        │   height + diameter → barrels
│   └─ aggregate.py       │   per-tank → per-refinery → weekly panel
└──────────┬──────────────┘
           ▼
┌─────────────────────────┐
│  analysis/              │   Jupyter notebooks
│   ├─ 01_baseline.ipynb  │     2025 distributions, QC
│   ├─ 02_event_study.ipynb
│   └─ 03_leadlag.ipynb
└──────────┬──────────────┘
           ▼
┌─────────────────────────┐
│  outputs/               │
│   ├─ figures/  (paper)  │
│   └─ data/    (release) │
└─────────────────────────┘
```

Storage: local Parquet under `data/`, with GEE asset IDs referenced from code. No private cloud required for v1. A dashboard v2 can sit on the same Parquet store or be migrated to DuckDB/Postgres.

## 6. SAR tank-farm measurement — the hard part

The physical setup is standard for external floating-roof (EFR) tanks: the steel roof floats on the liquid surface and descends as the tank drains. In a SAR image, the bright *layover* return from the tank's far wall overlays the roof deck; the radial distance between the outer rim return and the layover bright ring encodes the roof depression below the tank top.

Minimal-pipeline approach for v1:

1. **Mask:** hand-digitize each crude-service tank in GEE from Sentinel-2 median composite. Record outer diameter (m) and reference tank-top elevation.
2. **Per-pass feature extraction:** clip Sentinel-1 IW GRD to tank footprint buffered by 2× diameter; re-project to range-Doppler; measure layover offset in pixels along the range axis.
3. **Height from geometry:** `h_roof_below_top = (offset_px · pixel_spacing · sin(θ_incidence))`, where θ is the per-pixel incidence angle from the S-1 metadata.
4. **Volume:** `bbl = π · (D/2)² · (H_tank − h_roof_below_top) · 6.2898` (m³→bbl), reported as delta from baseline rather than absolute to cancel tank-height bias.
5. **QC:** reject passes with wind-roughened liquid surface (high VH on tank interior), with snow/rain contamination (rare in CA), or with incidence angle outside 30°–45°.

Expected precision (per published Sentinel-1 EFR work): roof-height RMSE ~1.0 m, which for a 100 m diameter tank is ~50 kbbl — coarse in absolute terms but directionally reliable at a refinery (4–12 tanks) aggregated weekly.

Observation cadence: Sentinel-1C alone gives 12-day repeat; with 1C + 1D it drops to 6-day. Orbital geometry yields roughly 4–5 useful passes per refinery per month.

## 7. Pipeline milestones

| M | Scope | Exit criterion |
| --- | --- | --- |
| M1 | Data-source access confirmed end to end | Single notebook pulls one week of each stream for one refinery |
| M2 | Tank mask complete for 5 refineries | `tank_masks.geojson` reviewed; tank count, diameters, service type recorded |
| M3 | SAR height pipeline on Richmond baseline | R² ≥ 0.7 vs. known Chevron filings for one quarter of 2025 |
| M4 | Full 2025 panel built | `refinery_fill_weekly.parquet` covering 2025-01 → 2026-02 with <10% missing weeks |
| M5 | Event-study on 2026 shock | β_k plot with CIs; lead-lag regression table |
| M6 | Paper draft | Method + results sections; figures 1–4 |
| M7 | Dashboard MVP (paper-later) | Same Parquet store, simple web view |

M1–M5 is the critical path for the paper. M6 is writing. M7 is deferred but constrained by keeping the data layer Parquet/columnar rather than tying it to notebook-internal state.

## 8. Repository layout (proposed)

```
/
├── docs/
│   ├── spec.md              (this file)
│   └── data_sources.md      (validation report)
├── ingest/                  (Python)
├── process/                 (Python)
├── analysis/                (Jupyter notebooks)
├── data/                    (Parquet, gitignored)
├── masks/                   (tank_masks.geojson, versioned)
├── outputs/
│   ├── figures/
│   └── tables/
├── environment.yml          (conda)
└── README.md
```

The existing `index.html` (earthquake/CDCR map) is unrelated and will be left untouched on `main`; this branch supersedes it in the project-root sense only via the `docs/` and code directories.

## 9. Risks and honest caveats

- **Sandbox validation gap.** The four authoritative data-source URLs could not be reached from the spec-drafting environment (all returned 403). The `data_sources.md` report marks each as **needs-live-verification** — the user must confirm endpoints from an unrestricted network before M1 closes.
- **Benicia closure confound.** Valero-Benicia's announced closure creates a structural drawdown independent of any Hormuz effect. Options: (a) drop Benicia from the panel, (b) include it with a closure-dummy interaction. Decision deferred to M2.
- **6-day SAR revisit.** ~4–5 observations per refinery per month is enough to detect a multi-week drainage trend, not enough to catch a single-day event. This is stated up front in the paper.
- **Layover-geometry precision.** Sentinel-1's 10 m GRD resolution caps per-tank precision at ~1 m roof height. The paper claims *directional* detection, not barrel-audit accuracy.
- **Ground-truth granularity mismatch.** EIA publishes PADD 5 (CA + OR + WA + AK + HI), not CA-specific. The panel must either scale PADD 5 to CA (CEC monthly data) or accept a slightly noisier aggregate. Both are defensible; pick one in M4.
- **Iranian crude is not a direct LA input.** LA/LB refineries run ANS, domestic, and Latin American grades; Hormuz disruption propagates through global price + redirection of Middle East cargoes, not through direct LA imports. The paper must be careful about the transmission mechanism claim.

## 10. Open questions for the user

1. Should Benicia be kept in the panel with a closure dummy, or dropped now?
2. Is there an existing GEE project / EE asset ID already available from prior InSAR work, or should this spec assume a fresh GEE project?
3. Target venue for the paper? (That affects length, emphasis on econometrics vs. remote-sensing method, and figure style.)
4. Is a MarineTraffic paid tier available, or must AIS cross-check stay on the free tier (rate-limited, spotty historical)?
