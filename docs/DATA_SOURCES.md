# Data Sources

Every external dataset the app uses, with the endpoints that were **verified live on 2026-09-08**, the license, the citation text the UI must show, and the caching policy. The spec forbids simulated data: if a source below is unreachable the app shows an explicit unavailable state (DESIGN.md §8). This file seeds the `data_sources` and `licenses` tables.

## Summary

| id | Dataset | Access pattern | License | Redistribution |
|---|---|---|---|---|
| `argo_ifremer_erddap` | Argo GDAC via Ifremer ERDDAP | scheduled index + on-demand levels | CC-BY-4.0 | yes, with attribution |
| `argovis` | Argovis API (optional) | on demand | data CC-BY-4.0; cite Argovis | yes |
| `ndsf_dives_wfs` | NDSF Vehicle Dive Metadata (MGDS WFS) | weekly ingest | CC BY-NC-SA 3.0 US (MGDS terms) | non-commercial, share-alike |
| `ndsf_alvin_ops_xlsx` | NDSF Alvin dive log spreadsheet | weekly ingest | NDSF data policy (metadata public) | non-commercial, credit NDSF/WHOI |
| `ndsf_jason_ops_xlsx` | NDSF DSL operations summaries (Jason) | weekly ingest | NDSF data policy | non-commercial, credit NDSF/WHOI |
| `r2r` | Rolling Deck to Repository cruise API | weekly ingest | CC0 per R2R policy (confirm per cruise) | yes; cite cruise DOI |
| `bcodmo_932975` | BCO-DMO megafauna counts + cropped images | monthly ingest | CC-BY-4.0 | yes, with attribution |
| `ifcb_whoi` | WHOI IFCB dashboard | daily time series; on-demand bins/images | **not stated** | metadata cached; images proxied only |
| `nasa_gibs` | NASA GIBS WMTS imagery | browser-direct tiles; daily time domains | US Government work (no copyright); acknowledgement requested | yes |
| `coastwatch_erddap` | NOAA CoastWatch ERDDAP grids (MUR SST, VIIRS chl-a) | on-demand point series | NOAA/JPL open data; acknowledgement requested | yes |

---

## 1. Argo floats

### 1a. Ifremer ERDDAP `ArgoFloats` (primary)
* Base: `https://erddap.ifremer.fr/erddap/tabledap/ArgoFloats.{json|geoJson|csv|nc}`
* Variables (verified): `platform_number, cycle_number, direction, data_mode, time, latitude, longitude, pres, temp, psal, doxy, nitrate, chla, turbidity`, `*_qc`, `*_adjusted`, `*_adjusted_error`, `project_name, pi_name, float_serial_no, firmware_version`.
* Example: `ArgoFloats.geoJson?platform_number,cycle_number,time,latitude,longitude,data_mode&time>=2026-08-29T00:00:00Z&time<=2026-09-08T00:00:00Z&latitude>=-20&latitude<=30&longitude>=-140&longitude<=-80&distinct()`
* Levels for one profile: constrain `platform_number=` and `cycle_number=`; request `pres,temp,psal,doxy,chla,temp_qc,psal_qc,pres_qc,temp_adjusted,psal_adjusted,pres_adjusted`.
* Notes: no API key; ERDDAP constraints are `&var>=value`; use `orderBy`/`distinct`; expect slow responses for wide time windows, so the worker maintains a rolling 30-day index and the API caps ad-hoc windows.

### 1b. Argovis (optional)
* Base: `https://argovis-api.colorado.edu` — `/argo?startDate&endDate&polygon|box|center+radius&platform&data&compression=minimal`, `/argo/meta`, `/argo/vocabulary`, `/grids/{gridName}`, `/timeseries/{name}`, `/summary`. Verified: anonymous `GET /argo?startDate=…&endDate=…&compression=minimal` returns `[id, lon, lat, timestamp, [data types], [mission]]` rows. API keys (free) raise rate limits; header name and limits to confirm at https://argovis.colorado.edu.
* Citation if used: Tucker, T., Giglio, D., Scanderbeg, M., & Shen, S. S. P. (2020). Argovis: A Web Application for Fast Delivery, Visualization, and Analysis of Argo Data. *J. Atmos. Oceanic Technol.* 37(3), 401–416.

### License and citation (both)
* License: **CC-BY-4.0** (SEANOE record).
* Citation: *Argo (2026). Argo float data and metadata from Global Data Assembly Centre (Argo GDAC). SEANOE. https://doi.org/10.17882/42182*
* Acknowledgement (show verbatim): *"These data were collected and made freely available by the International Argo Program and the national programs that contribute to it (https://argo.ucsd.edu, https://www.ocean-ops.org). The Argo Program is part of the Global Ocean Observing System."*

---

## 2. Alvin and Jason dive data (NDSF)

### 2a. NDSF Vehicle Dive Metadata — MGDS WFS (primary, all vehicles)
This is the service behind https://ndsf.whoi.edu/vehicle-dive-metadata/ (embedded map at `https://www.marine-geo.org/ndsf/dives/mapview.html`).
* Endpoint: `https://www.marine-geo.org/services/mapserver/ndsf_metadata?SERVICE=WFS&VERSION=1.1.0&REQUEST=GetFeature&TYPENAME=Dives&outputformat=geojson` (`outputformat` also `csv`, `gml`). `GetCapabilities` and `DescribeFeatureType` work.
* Filters (query params, verified from the map's `form.js`): `vehicle` (ALVIN, JASON, JASON2, SENTRY, ANGUS, ARGO, ARGO2, CMARC, DSL120, DSL120A, MANGUS, MEDEA, MEDEA2, OCEANIC EXPLORER, TOWCAM, ABE), `cruise_id`, `minlon, maxlon, minlat, maxlat`, `starttime, endtime`, `chief_sci`, `pilot`, `feature`, booleans `current_ndsf_vehicle`, `data_at_mgds`, `has_framegrabs`.
* Menu helper: `https://www.marine-geo.org/ndsf/services/dive_menu_filter.php?vehicle=ALVIN` → `{cruise_ids[], vehicles[]}`.
* Properties: `dive_id, dive_uid, vehicle, vehicle_type, platform_term, current_ndsf_vehicle, dive_start, dive_end, data_start, data_end, location_lat, location_lon, max_depth (negative m), dive_feature, cruise_feature, all_features, devices, comments, observers[{obs1,obs2,pilot}], pilot, chief_scientists[{name,affiliation}], chief_sci_string, cruise_id, cruise_uid, dsl_ops_id, ship_id, framegrab_url, has_framegrabs, mgds_event_url, data_at_mgds`.
* Verified scale: full unfiltered GeoJSON is ~8.2 MB with **8,358 dives** (ALVIN 5,001; JASON2 1,727; SENTRY 744; ANGUS 304; JASON 246; …), years 1971–2026; 1,871 dives have Sealog framegrab replays (e.g. `https://seaplay.whoi.edu/sealog-alvin/lowering_replay/Alvin-D5044`, `https://seaplay.whoi.edu/sealog-jason/lowering_replay/J2-1311`).
* AT42-21 / Alvin 5044 and RR2102 / J2-1309, J2-1311 are all present.
* Also available: the MGDS event catalog WFS `https://www.marine-geo.org/services/mapserver/eventwfs` (Stations, Launches, Lines) and the entries search `https://www.marine-geo.org/services/search/entries?entry_id=AT42-21` / `?platform_id=Alvin` (XML with dataset DOIs), and `FileServer` (`minlongitude…platform, data_type, format`).
* License: MGDS Terms of Use — **CC BY-NC-SA 3.0 US**; cite both original contributors and MGDS; not for navigation; no commercial databases.
* Citation: *National Deep Submergence Facility (Woods Hole Oceanographic Institution). NDSF Vehicle Dive Metadata, served by the Marine Geoscience Data System (MGDS), https://ndsf.whoi.edu/vehicle-dive-metadata/ (accessed <date>).* Plus for any dataset-level link-out: the MGDS dataset DOI.

### 2b. NDSF operations spreadsheets (enrichment)
Linked from https://ndsf.whoi.edu/data/ (file names change on each update — discover the current link from that page, do not hard-code):
* Alvin dive log: e.g. `https://ndsf.whoi.edu/wp-content/uploads/sites/67/2026/08/Dive-Log-15Aug2026-2.xlsx`, sheet `ALVIN-DiveLog`, 6,068 rows (1964–2026). Columns: `Dive, Date, Cruise, Leg, R2R Cruise ID, Fixing R2R Cruise ID, Ch Sci, Ops Area, Lat, Lon (DDM strings e.g. "09-47.259N"), Pilot, Obs 1, Obs 2, Launch Time, Time Submerged, Recovery Time, Bottom Time, Depth, Purpose, Sponsor, Observed, Sampled, Remarks`. Dive 5044 row verified: 2019-12-25, AT42-21, Sylvan, "9N Biotransect", 2518 m, Microbiology, NSF.
* Jason / DSL ops summaries: e.g. `https://ndsf.whoi.edu/wp-content/uploads/sites/67/2026/08/DSL_Ops_Summaries_June2026.xlsx`, sheet `All Lowerings` (2,714 rows, all tethered vehicles). Columns: `Vehicle, Lowering Id, DSL Ops Id, Vessel, Cruise Id, Chief Scientist(s)/Affiliation(s), Work Area (Navy Ref), Work Area (Text), Lat, Lon, UTM, Year, Old_Lowering Id, Start/Launch, Start Data, End Data, End/On Deck, Line/Area, Data, Lowering, Coverage, MinDepth, MaxDepth, Comments`. J2-1309 and J2-1311 (RR2102, Fornari/WHOI, EPR 9N) verified.
* Join key: `(vehicle, dive_id)` ↔ spreadsheet `Dive` / `Lowering Id`.
* Policy: NDSF Data Policy — dive metadata is public; **video and photo imagery are CC BY-NC-SA 4.0** with credit *"<chief scientist>, <institution>, and other participating institutions. Funder: <funder>. © Woods Hole Oceanographic Institution"* and the non-commercial notice. The app links to Sealog replays and the Alvin Frame-Grabber (`http://4dgeo.whoi.edu/alvin`, HTTP only) and never re-hosts imagery. WHOI archive `https://dlacruisedata.whoi.edu/NDSF/alvin/dive/<dive>/` was under maintenance at verification time.

### 2c. R2R cruise metadata
* `https://service.rvdata.us/api/cruise/?cruise_id=AT42-21` → `cruise_id, cruise_name, cruise_doi, vessel_name, operator_name, depart_date, depart_port_name, arrive_date, arrive_port_name, …`
* `…/api/fileset/?cruise_id=` (instrument filesets with DOIs), `…/api/product/?cruise_id=` (processed products incl. `r2rnav` navigation), `…/api/vessel/?vessel_id=Atlantis`.
* AT42-21: R/V Atlantis, 2019-12-17 → 2020-01-07, Manzanillo → Balboa, Chief Scientist Jason Sylvan (TAMU), DOI **10.7284/908846**.
* License: R2R states datasets carry a **CC0** statement; cite the cruise DOI. Confirm per-cruise on ingest (`license` field if present).

---

## 3. BCO-DMO dataset 932975 — EPR inactive-sulfide megafauna
* Landing: https://www.bco-dmo.org/dataset/932975 · DOI **10.26008/1912/bco-dmo.932975.1** · handle https://hdl.handle.net/1912/69826
* ERDDAP: `https://erddap.bco-dmo.org/erddap/tabledap/bcodmo_dataset_932975_v1.{json|csv|geoJson}`; columns: `File_Name, Site, Habitat, Corrected_Area_m2, verbatimIdentification, Feeding_Mode, individualCount, WoRMS_scientificName, WoRMS_scientificNameID, eventDate, latitude, longitude, depth, Vehicle_Altitude_m, AngleCategory, Adjustment_Value, Useable_Proportion_of_Image`.
* Files: CSV `https://datadocs.bco-dmo.org/file/Dw7n62vu5lXxO0/932975_v1_megafauna-counts.csv`; cropped morphotype images `https://datadocs.bco-dmo.org/dataset/932975/file/QAqEj2MfBmVMN3/cropped_images_CreatureDex-2024-07-05.zip` (12 MB); wide-format `Morphotype_Counts-BCODMO_2024-07-05.xlsx`.
* Content (verified): 143 images, 51 morphotypes. Lucky's Mound 2019-12-25 (49 images, `D5044_OIS_down_…`, Alvin 5044, AT42-21); Oceanic Rise 2021-04-07 (50, `RR2102_J1309_OID_…`, Jason J2-1309); Sentry Spire 2021-04-09 (44, `RR2102_J1311_OID_…`, Jason J2-1311). ~2,500 m, 9.77–9.79°N, 104.29°W.
* Original survey stills are not in the dataset; the dataset page points to NDSF navigation data (https://ndsf.whoi.edu/data/). Pairing therefore uses dive metadata + Sealog replay links (§2).
* License: **CC-BY-4.0**. Citation: *Beaulieu, S., Meneses, M., Best, A., Dykman, L., Mills, S., Wu, J., Mullineaux, L. (2024). Megafauna counts by taxon in images collected during three surveys (December 25, 2019, April 7 and 9, 2021) with deep-submergence vehicles at inactive sulfide mounds on the East Pacific Rise. Biological and Chemical Oceanography Data Management Office (BCO-DMO). Version 1. https://doi.org/10.26008/1912/bco-dmo.932975.1* Funding acknowledgement per the dataset page (NSF; PI Lauren Mullineaux).

---

## 4. IFCB (WHOI dashboard)
* Base: `https://ifcb-data.whoi.edu` (ifcbdb 5.2, MIT-licensed software by the Sosik lab). Verified JSON endpoints:
  * `/api/list_datasets` → 28 dataset names (`mvco`, `NESLTER_transect`, `NAAMES`, `EXPORTS`, `arctic`, …)
  * `/api/most_recent_bin?dataset=mvco` → `{pid, timestamp, temperature, humidity, …}`
  * `/api/bin/{pid}` → `lat, lng, depth, timestamp_iso, instrument, num_triggers, num_images, concentration, ml_analyzed, primary_dataset, previous_bin_id, next_bin_id, tags, comments, has_blobs, has_features, scale, shape`
  * `/api/time-series/{metric}?dataset=mvco&resolution=week` → `{x: [iso…], y: […]}` (metrics seen on the dashboard: temperature, humidity, size, volume analyzed, ROI concentration, trigger count, image count; `start`/`end` params returned 500 in one test — confirm exact parameter names before relying on them)
  * `/api/image/{pid}/{target}` (ROI metadata), `/data/{pid}_{target}.png|.jpg` (ROI image), `/data/{pid}_features.csv`, `/data/{pid}_class_scores.mat`, `/api/closest_bin?dataset=&target_date=`, `/api/nearest_bin`, `/api/plot/{pid}?x=&y=` (large), `/api/export_metadata/{dataset}` (CSV of all bins; very slow for MVCO)
* MVCO dataset DOI: **10.26025/9q7z-a148** (DataCite: Sosik, Futrelle, Peacock, Crockford, Shalapyonok, Olson; WHOI; 2006–).
* License: **not stated** on the dashboard, About page, or DataCite record. Policy until written permission is obtained: cache dataset list, time series and bin metadata with access dates; proxy ROI images with short cache headers and no persistent storage; always show credit "IFCB data from the WHOI IFCB dashboard, Sosik lab, Woods Hole Oceanographic Institution; DOI 10.26025/9q7z-a148" and the funding acknowledgement listed on the dashboard (NSF, NASA, Gordon and Betty Moore Foundation, Simons Foundation, WHOI). A tracked issue covers contacting the lab.

---

## 5. Satellite ocean surface

### 5a. NASA GIBS (map tiles)
* WMTS REST template: `https://gibs.earthdata.nasa.gov/wmts/epsg3857/best/{Layer}/default/{YYYY-MM-DD}/{TileMatrixSet}/{z}/{y}/{x}.{png|jpg}`; capabilities `…/wmts/epsg3857/best/1.0.0/WMTSCapabilities.xml` (5.8 MB); per-layer time domain `…/wmts/epsg3857/best/1.0.0/{Layer}/default/{TileMatrixSet}/all/all.xml` (DescribeDomains).
* Verified layer ids (format, tile matrix set, time domain):

| Layer | Format | TMS | Period / coverage |
|---|---|---|---|
| `GHRSST_L4_MUR_Sea_Surface_Temperature` | png | GoogleMapsCompatible_Level7 | daily, 2002-06-01 → present (gaps) |
| `GHRSST_L4_MUR_Sea_Surface_Temperature_Anomalies` | png | Level7 | daily |
| `MODIS_Aqua_L2_Chlorophyll_A` | png | Level7 | daily, 2002-07-04 → present |
| `VIIRS_SNPP_L2_Chlorophyll_A` | png | Level7 | daily, 2012 → present |
| `VIIRS_NOAA20_Chlorophyll_a` | png | Level7 | daily, 2017-12 → present |
| `OCI_PACE_Chlorophyll_a` | png | Level7 | daily, 2024 → present |
| `SMAP_L3_Sea_Surface_Salinity_CAP_8Day_RunningMean` | png | Level6 | daily, 2015-05 → present |
| `JPL_MEaSUREs_L4_Sea_Surface_Height_Anomalies` | png | Level6 | 5-day, 1992-09-30 → 2017-10-19 (historical only) |
| `MODIS_Aqua_CorrectedReflectance_TrueColor`, `VIIRS_SNPP_CorrectedReflectance_TrueColor` | jpg | Level9 | daily |
| `AVHRR_MetOp-B_L3U_Sea_Surface_Temperature` | png | Level7 | daily |

* Legends/colormaps: `https://gibs.earthdata.nasa.gov/colormaps/v1.3/{Layer}.xml` (confirm per layer during implementation).
* Terms: NASA imagery/data are US Government works, free to use. Requested acknowledgement (show verbatim): *"We acknowledge the use of imagery provided by services from NASA's Global Imagery Browse Services (GIBS), part of NASA's Earth Observing System Data and Information System (EOSDIS)."*

### 5b. NOAA CoastWatch ERDDAP (values / time series)
* `jplMURSST41` (griddap): `analysed_sst` (°C), `analysis_error`, `mask`, `sea_ice_fraction`; 0.01°, daily, 2002-06-01 → present. Point series: `https://coastwatch.pfeg.noaa.gov/erddap/griddap/jplMURSST41.json?analysed_sst[(2026-08-01T09:00:00Z):1:(2026-09-01T09:00:00Z)][(9.79)][(-104.29)]`
  * Acknowledgement (verbatim): *"These data were provided by JPL under support by NASA MEaSUREs program."* Reference: https://podaac.jpl.nasa.gov/dataset/MUR-JPL-L4-GLOB-v4.1
* `nesdisVHNnoaa20chlaDaily`: `chlor_a` (mg m⁻³), 4 km, daily, NOAA/NESDIS/STAR (near-real-time; coverage lags). Gap-filled alternative `nesdisVHNnoaaSNPPnoaa20chlaGapfilledDaily` (9 km). Multi-decade climate-quality alternative `pmlEsaCCI60OceanColor8Day`.
* License text on both: data are free to use and redistribute but "not intended for legal use, since it may contain inaccuracies"; credit NOAA CoastWatch and the producing agency.

---

## Attribution footer (every dashboard)
Rendered from the registry; example for the EPR Megafauna dashboard:

> Megafauna counts: Beaulieu et al. (2024), BCO-DMO, doi:10.26008/1912/bco-dmo.932975.1, CC-BY-4.0 · Dive metadata: NDSF/WHOI via MGDS (CC BY-NC-SA 3.0 US), accessed 2026-09-07 · Cruise AT42-21: R2R doi:10.7284/908846 · Imagery © Woods Hole Oceanographic Institution, CC BY-NC-SA 4.0 (linked, not re-hosted).
