# REST API (OpenAPI 3.1)

Base path `/api/v1`. The FastAPI app generates `openapi.json`; the exported file is checked in at `apps/api/openapi/openapi.json` and CI fails if it drifts. Interactive docs at `/api/docs` (Swagger UI) and `/api/redoc`.

## Conventions

| Topic | Rule |
|---|---|
| Auth | Session cookie (`ocean_session`, httpOnly, SameSite=Lax) + `X-CSRF-Token` header on mutating requests. Public endpoints work without a session. |
| Errors | RFC 9457 `application/problem+json` with `type` URIs such as `/problems/source-unavailable`, `/problems/validation`, `/problems/conflict`. |
| Provenance | Every response that contains upstream data includes `provenance: Provenance[]` (below). Never omitted, never empty when data is present. |
| Geodata | Point layers return RFC 7946 GeoJSON `FeatureCollection`; `bbox=minLon,minLat,maxLon,maxLat`. |
| Time | ISO 8601 UTC; `start`/`end` query params; ranges are half-open `[start, end)`. |
| Pagination | Cursor-based: `?limit=&cursor=` → `next_cursor`. Map layers use bbox + zoom instead. |
| Caching | `ETag` + `Cache-Control` on cacheable GETs; `Last-Modified` reflects `fetched_at`. |
| Rate limits | Per-IP token bucket on public endpoints; per-user on authenticated; `429` with `Retry-After`. |
| Versioning | Path version; additive changes only within v1. |

### Provenance object

```json
{
  "source_id": "ndsf_dives_wfs",
  "source_name": "NDSF Vehicle Dive Metadata (MGDS WFS)",
  "citation": "National Deep Submergence Facility (WHOI) and Marine Geoscience Data System (MGDS), NDSF Vehicle Dive Metadata, https://ndsf.whoi.edu/vehicle-dive-metadata/",
  "doi": null,
  "license": {"id": "CC-BY-NC-SA-3.0-US", "url": "https://creativecommons.org/licenses/by-nc-sa/3.0/us/"},
  "acknowledgement": "Data provided by NDSF/WHOI via the Marine Geoscience Data System.",
  "source_url": "https://www.marine-geo.org/services/mapserver/ndsf_metadata?SERVICE=WFS&...&cruise_id=AT42-21",
  "fetched_at": "2026-09-07T03:00:12Z",
  "stale": false
}
```

### Source-unavailable problem

```json
{
  "type": "/problems/source-unavailable",
  "title": "Upstream source unavailable",
  "status": 502,
  "detail": "erddap.ifremer.fr returned HTTP 503",
  "source_id": "argo_ifremer_erddap",
  "last_success_at": "2026-09-06T21:00:04Z",
  "retry_after": 120
}
```

## Endpoints

### Auth and users
| Method | Path | Notes |
|---|---|---|
| POST | `/auth/register` | email, password, display_name → sends verification email |
| POST | `/auth/login` / `/auth/logout` | rate-limited |
| POST | `/auth/verify-email`, `/auth/password-reset`, `/auth/password-reset/confirm` | |
| GET | `/auth/oauth/{provider}/start`, `/auth/oauth/{provider}/callback` | provider = `orcid` |
| GET / PATCH | `/users/me` | profile |

### Workspaces, dashboards, sharing, annotations
| Method | Path | Notes |
|---|---|---|
| GET / POST | `/workspaces` | list mine / create |
| GET / PATCH / DELETE | `/workspaces/{id}` | |
| GET / POST / DELETE | `/workspaces/{id}/members` | role management; invite by email |
| GET | `/workspaces/{id}/activity` | activity feed, cursor-paginated |
| GET / POST | `/workspaces/{id}/dashboards` | |
| GET / PATCH / DELETE | `/dashboards/{id}` | PATCH body `{version, patch: JSON-Patch[] , message?}` → `409 conflict` with current state on version mismatch |
| GET | `/dashboards/{id}/versions`, `/dashboards/{id}/versions/{n}` | history; `POST /dashboards/{id}/versions/{n}/restore` |
| GET / POST | `/dashboards/{id}/share-links` | mode `live` or `snapshot`, optional `expires_at` |
| DELETE | `/share-links/{id}` | revoke |
| GET | `/s/{slug}` | public; returns dashboard state (+ frozen provenance for snapshots) |
| GET / POST | `/dashboards/{id}/annotations` | filters: bbox, target_type, resolved |
| PATCH / DELETE | `/annotations/{id}` | edit body, resolve/unresolve |
| GET / POST | `/annotations/{id}/comments` | |
| WS | `/ws/dashboards/{id}` | see protocol below |

### Sources and attribution
| Method | Path | Notes |
|---|---|---|
| GET | `/sources` | registry with license, citation, last success/attempt times |
| GET | `/sources/{id}` | plus recent fetch log |
| GET | `/licenses` | |
| POST | `/admin/sources/{id}/refresh` | admin; enqueues a refresh job |
| GET | `/health` | liveness + per-source status summary |

### Argo
| Method | Path | Notes |
|---|---|---|
| GET | `/argo/profiles` | GeoJSON; `bbox`, `start`, `end`, `platform`, `bgc=true`, `data_mode`, `limit` (default: last 10 days) |
| GET | `/argo/profiles/{id}` | metadata + levels (fetched on demand, cached) |
| GET | `/argo/floats/{platform_number}` | float metadata + track (GeoJSON LineString + cycle points) |
| GET | `/argo/profiles.csv` | same filters; CSV with citation header comment |

### Deep submergence
| Method | Path | Notes |
|---|---|---|
| GET | `/dives` | GeoJSON; `vehicle[]`, `cruise_id`, `bbox`, `start`, `end`, `chief_sci`, `min_depth`, `max_depth`, `has_framegrabs`, `feature` |
| GET | `/dives/{dive_uid}` | full record incl. ops enrichment, links (Sealog replay, MGDS event, R2R cruise) |
| GET | `/dives/facets` | distinct vehicles, cruises, chief scientists, year histogram (for filter UI) |
| GET | `/cruises/{cruise_id}` | R2R metadata + dives on the cruise |

### EPR megafauna (BCO-DMO 932975)
| Method | Path | Notes |
|---|---|---|
| GET | `/megafauna/images` | GeoJSON; `site`, `habitat`, `dive_id`, `taxon_id`; properties include total count and paired `dive_uid` |
| GET | `/megafauna/images/{file_name}` | image metadata, counts by taxon, paired dive card |
| GET | `/megafauna/taxa` | 51 morphotypes with WoRMS links and feeding mode |
| GET | `/megafauna/taxa/{id}/image` | cropped image from the dataset's CC-BY supplemental ZIP |
| GET | `/megafauna/summary` | `group_by=site|habitat|taxon|feeding_mode`; counts and densities |

### IFCB
| Method | Path | Notes |
|---|---|---|
| GET | `/ifcb/datasets` | |
| GET | `/ifcb/datasets/{name}/timeseries` | `metric`, `resolution=day|week|month`, `start`, `end` |
| GET | `/ifcb/datasets/{name}/bins` | bins in a time window (positions when available) |
| GET | `/ifcb/bins/{pid}` | bin metadata; `previous`/`next` navigation |
| GET | `/ifcb/bins/{pid}/images/{target}.png` | proxied ROI image (no persistent storage) |
| GET | `/ifcb/bins/{pid}/features.csv` | proxied |

### Satellite
| Method | Path | Notes |
|---|---|---|
| GET | `/satellite/layers` | registry incl. WMTS template, TMS, format, time domain, legend |
| GET | `/satellite/layers/{id}/times` | available dates in `[start,end)` (gaps removed) |
| GET | `/satellite/point` | `layer_id`, `lat`, `lon`, `start`, `end` → daily values from ERDDAP griddap (cached) |

Tiles are requested by the browser directly from GIBS (CORS-enabled); the API never proxies tiles.

## WebSocket protocol (`/ws/dashboards/{id}`)

Client → server: `hello {client_id, name?}` · `presence {cursor?, viewport?, selection?}` · `patch {base_version, ops}` · `ping`.

Server → client: `welcome {state, version, peers[]}` · `presence {client_id, ...}` · `patch {version, ops, author}` · `conflict {state, version}` · `annotation {event, annotation}` · `peer_left {client_id}`.

Patches are persisted through the same path as `PATCH /dashboards/{id}` and broadcast to all replicas via `NOTIFY dashboard_<id>`.

## Example: dive with provenance

```json
{
  "dive_uid": "16936",
  "dive_id": "5044",
  "vehicle": "ALVIN",
  "cruise_id": "AT42-21",
  "ship": "R/V Atlantis",
  "dive_start": "2019-12-25T14:12:00Z",
  "dive_end": "2019-12-25T23:02:00Z",
  "position": {"type": "Point", "coordinates": [-104.283417, 9.78765]},
  "max_depth_m": 2518,
  "dive_feature": "9N Biotransect",
  "chief_scientists": [{"name": "Sylvan"}],
  "observers": {"obs1": "Amanda Achberger", "obs2": "Lauren Dykman", "pilot": "B. Strickrott"},
  "ops": {"purpose": "Microbiology", "sponsor": "NSF", "bottom_time": "PT6H19M"},
  "links": {
    "framegrab_replay": "https://seaplay.whoi.edu/sealog-alvin/lowering_replay/Alvin-D5044",
    "mgds_event": "https://www.marine-geo.org/tools/relateddata/event/...",
    "r2r_cruise": "https://doi.org/10.7284/908846"
  },
  "provenance": [
    {"source_id": "ndsf_dives_wfs", "fetched_at": "2026-09-07T03:00:12Z", "...": "..."},
    {"source_id": "ndsf_alvin_ops_xlsx", "fetched_at": "2026-09-07T03:01:40Z", "...": "..."},
    {"source_id": "r2r", "fetched_at": "2026-09-07T03:02:05Z", "...": "..."}
  ]
}
```
