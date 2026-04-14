# Data Source Validation Report

**Status:** Draft v0.1 — 2026-04-14
**Companion to:** `docs/spec.md`
**Scope:** the core causal chain — AIS + Sentinel-1 SAR + EIA — for the LA/LB + 5-refinery v1.

## Validation method and caveat

Each source below was assessed on four axes: **access**, **coverage**, **latency**, and **known frictions**.

**Caveat up front:** during spec drafting, outbound WebFetch to each primary URL returned HTTP 403 from this environment (sandbox egress is blocked). The assessments therefore rely on prior documented behavior and must be **live-verified** from an unrestricted network before milestone M1 closes. Each row below carries an explicit verification command.

---

## 1. IMF PortWatch (primary AIS source)

- **URL:** `https://portwatch.imf.org/`
- **What it provides:** Daily vessel-call counts at ~1,200 ports, broken down by vessel type including **crude tanker** and **product tanker**. LA and Long Beach are covered as separate port entities.
- **Access:** Dashboard + downloadable CSV/JSON; an ArcGIS FeatureServer backs the maps and can be hit directly for programmatic pulls.
- **Latency:** ~1–2 day publication lag.
- **Coverage:** Global, 2019-present. Uses UN Global Platform AIS.
- **Known frictions:**
  - Vessel-type classification is based on AIS self-declaration + PortWatch's classifier; mis-classification between crude tanker and product tanker is the dominant noise source. The spec's aggregates should sum both and treat the ratio as a secondary signal.
  - ArcGIS FeatureServer rate limits are generous but undocumented; budget for local caching.
- **Verification commands (user to run):**
  - `curl -sI https://portwatch.imf.org/` → expect 200.
  - Open dashboard, export CSV for "Port Calls — Los Angeles, Long Beach" for 2026-01 to 2026-04; confirm crude-tanker column is present and non-empty.
- **Status:** **needs-live-verification.**

## 2. MarineTraffic (cross-check AIS source)

- **URL:** `https://www.marinetraffic.com/`
- **What it provides:** Vessel positions, per-vessel history, port-call records.
- **Access:** Free tier is rate-limited and does not expose historical bulk downloads. Paid API tiers expose per-port arrivals/departures via REST.
- **Latency:** Near real-time for positions; port calls within hours.
- **Known frictions:**
  - Free tier is inadequate for a 14-month backfill — plan on either (a) using PortWatch alone as primary and MarineTraffic only for spot-checks, or (b) paid tier.
  - ToS restricts redistribution; any derived artifacts in the paper repo must be aggregated counts only, not per-vessel records.
- **Verification commands (user to run):**
  - Confirm free-tier quota against the 14-month baseline requirement; if insufficient, drop MarineTraffic as a hard dependency and keep PortWatch primary.
- **Status:** **needs-live-verification** — may be descoped to a nice-to-have depending on free-tier limits.

## 3. Sentinel-1 C-band SAR via Google Earth Engine

- **Primary handle:** GEE collection `COPERNICUS/S1_GRD`
- **What it provides:** Sentinel-1A (retired 2022), 1B (lost 2022), 1C (launched Dec 2024), 1D (launched Nov 2025) Ground Range Detected imagery, IW mode over land, 10 m pixel spacing.
- **Access:** Free for research via GEE Python API (`ee` package). Requires a registered Earth Engine project; non-commercial use is free.
- **Coverage:** Southern California is covered by both ascending (~14:00 UTC) and descending (~02:00 UTC) passes. With 1C+1D the repeat cycle is 6 days per geometry.
- **Latency:** GEE ingestion is typically 2–5 days after acquisition. Acceptable for weekly-panel work; inadequate for a true real-time dashboard (move to direct Copernicus Data Space for <24h).
- **Known frictions:**
  - **Layover analysis requires GRD + incidence-angle band + orbit metadata.** GEE exposes `'angle'` band in GRD; verify it is present for IW scenes over CA.
  - **Incidence-angle variation** between ascending and descending passes means the geometric height conversion must be applied per-pass, not per-date.
  - **Sentinel-1 1C/1D onboard AIS** is a separate ESA product stream, not part of the SAR GRD collection. It is not required for v1 (PortWatch already gives AIS) but noted for v2.
- **Verification commands (user to run):**
  - In GEE code editor: `ee.ImageCollection('COPERNICUS/S1_GRD').filterBounds(refinery_point).filterDate('2026-01-01','2026-04-14').filter(ee.Filter.eq('instrumentMode','IW')).size()` — expect ≥ 15 for each refinery.
  - Confirm `'angle'` band presence on one recent image.
- **Status:** **needs-live-verification**, high confidence this will pass.

## 4. Copernicus Data Space (direct Sentinel-1 fallback)

- **URL:** `https://dataspace.copernicus.eu/`
- **What it provides:** The official successor to the retired Copernicus Open Access Hub. Full-resolution SAFE products via OData, STAC, and S3 endpoints.
- **Access:** Free, requires Copernicus SSO registration.
- **When to use:** If GEE latency is too high for the dashboard phase, or if SAR single-look-complex (SLC) products are needed for phase-based roof measurement (out of scope for v1 but noted).
- **Verification commands (user to run):**
  - Register at dataspace.copernicus.eu, obtain client credentials, run a minimal `pystac-client` search for an IW GRD scene over 33.75°N, 118.30°W on 2026-04-01.
- **Status:** **needs-live-verification**, not blocking for v1.

## 5. EIA Weekly Petroleum Status Report (ground truth)

- **URL:** `https://www.eia.gov/petroleum/supply/weekly/`
- **API:** EIA v2 REST — `https://api.eia.gov/v2/petroleum/` (free API key).
- **What it provides:**
  - PADD 5 **weekly crude oil stocks** (`WCESTP51` series and equivalents).
  - PADD 5 **refinery gross inputs** (`WGIRIP51`).
  - PADD 5 **percent refinery utilization**.
- **Access:** Free with API key; bulk CSV also downloadable.
- **Latency:** Released every **Wednesday at 10:30 ET** for the week ending the prior Friday — so a ~5-day lag on the data and a 7-day gap to the next report.
- **Granularity caveat:** PADD 5 aggregates CA, OR, WA, AK, HI. CA-specific weekly data is **not** published by EIA. Two options:
  - (a) Use PADD 5 and accept the aggregate (CA is ~60% of PADD 5 crude runs).
  - (b) Supplement with California Energy Commission (CEC) **monthly** refinery data for CA-specific calibration — lower frequency but CA-pure.
- **Verification commands (user to run):**
  - `curl "https://api.eia.gov/v2/petroleum/stoc/wstk/data/?api_key=$EIA_KEY&frequency=weekly&data[0]=value&facets[duoarea][]=R50&start=2025-01-01"` — expect a JSON payload with weekly PADD 5 stock values.
- **Status:** **needs-live-verification**, very high confidence this will pass.

## 6. California Energy Commission (supplementary ground truth)

- **URL:** `https://www.energy.ca.gov/data-reports/energy-almanac/` (Weekly Fuels Watch)
- **What it provides:** CA-specific weekly gasoline/diesel production, inventories, refinery inputs.
- **Access:** Free, CSV/XLS downloads.
- **Latency:** Weekly, ~1 week lag.
- **Role in v1:** Optional tiebreaker when PADD 5 and the satellite signal diverge. Not on the critical path.
- **Status:** **needs-live-verification**, low priority.

---

## Summary matrix

| Source | Role | Access | Latency | Confidence | Blocking? |
| --- | --- | --- | --- | --- | --- |
| IMF PortWatch | Primary AIS | Free dashboard + CSV | 1–2 d | High | **Yes** — v1 depends on it |
| MarineTraffic | AIS cross-check | Free (limited) / paid | hours | Low (free) | No — descope if needed |
| GEE `COPERNICUS/S1_GRD` | SAR core | Free (non-commercial) | 2–5 d | High | **Yes** |
| Copernicus Data Space | SAR fallback | Free + SSO | <1 d | High | No (v2) |
| EIA v2 API | Ground truth | Free + key | 5 d | Very high | **Yes** |
| CEC Weekly Fuels Watch | CA granularity | Free | 7 d | Medium | No |

## Open verification tasks (to close before M1)

1. Confirm PortWatch CSV export for LA + LB with crude-tanker breakdown, Jan 2025 → present.
2. Register GEE project and confirm `COPERNICUS/S1_GRD` has ≥4 images/refinery/month in 2025.
3. Obtain EIA v2 API key and pull PADD 5 weekly crude stocks 2025-01 → present.
4. Decide MarineTraffic free vs. paid (or drop).
5. Decide PADD 5 vs. PADD 5 + CEC for ground truth granularity.
6. Spot-check that the `'angle'` band is present on a recent IW GRD scene over Richmond — this is the single most important precondition for the SAR geometry pipeline.
