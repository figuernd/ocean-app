# Roadmap — Epics and Phases

Epics are GitHub issues labeled `epic`; each has sub-issues. Phases are labels `phase:1` … `phase:4`. Issue links are filled in below as they are created.

| Phase | Goal | Epics |
|---|---|---|
| 1 — Foundation | Running skeleton with provenance rules and one real dataset on a map | E1 Platform foundation · E2 Provenance, attribution & licensing · E4 Map & dashboard framework · E6 Deep Submergence dashboard |
| 2 — Datasets | All five required datasets as dashboards | E5 Argo · E7 EPR Megafauna · E8 IFCB · E9 Satellite |
| 3 — People | Logins, sharing, collaboration | E3 Accounts & auth · E10 Shareable links · E11 Collaboration |
| 4 — Polish | Documentation, operations, quality | E12 Documentation & help · E13 Operations & quality |

## Epics

| # | Epic | Summary | Issue |
|---|---|---|---|
| E1 | Platform foundation | Monorepo, FastAPI + Postgres/PostGIS + React/Vite skeleton, docker compose, CI, OpenAPI export | — |
| E2 | Provenance, attribution & licensing | Source registry, fetch log, provenance envelope, access dates in UI, no-simulated-data guard, Data Sources page | — |
| E3 | Accounts & authentication | Registration, sessions, email verification, password reset, ORCID OAuth, roles | — |
| E4 | Map & dashboard framework | MapLibre base map, layer registry, shared time control, dashboard state model, saved dashboards | — |
| E5 | Argo Floats dashboard | ERDDAP index ingest, profile map, T/S plots, float tracks, region export | — |
| E6 | Deep Submergence (Alvin & Jason) dashboard | NDSF WFS ingest, ops spreadsheets, R2R cruises, dive cards with Sealog/MGDS links | — |
| E7 | EPR Megafauna dashboard (BCO-DMO 932975) | ERDDAP ingest, taxa + cropped images, dive pairing, count charts | — |
| E8 | IFCB Plankton dashboard | Datasets, time series, bins, ROI image proxy, bin map | — |
| E9 | Satellite Ocean Surface dashboard | GIBS WMTS layers with time domains, legends, ERDDAP point series | — |
| E10 | Shareable links | URL state, short links (live/snapshot), public viewer, expiry/revocation | — |
| E11 | Collaboration | Workspaces & members, realtime presence/state sync, annotations & comments, history, activity feed | — |
| E12 | Documentation & help | In-app help, user guide, Data Sources & Citations page, API docs | — |
| E13 | Operations & quality | Deployment, scheduled jobs, rate limiting, observability, e2e tests, accessibility | — |

## Dependency sketch

```mermaid
flowchart LR
  E1 --> E2 --> E4
  E1 --> E3
  E4 --> E5 & E6 & E7 & E8 & E9
  E6 --> E7
  E3 --> E10 --> E11
  E4 --> E10
  E2 --> E12
  E5 & E6 & E7 & E8 & E9 --> E12
  E1 --> E13
```
