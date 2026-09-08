# Ocean App

An open, collaborative web application for exploring ocean observations on interactive maps and dashboards: Argo float profiles, Alvin and Jason deep-submergence dives, the BCO-DMO East Pacific Rise megafauna image surveys, IFCB plankton imagery, and satellite views of the ocean surface.

Built with React, a FastAPI OpenAPI REST API, and Postgres/PostGIS. Every displayed value is traceable to its source, cited, licensed, and stamped with its access date. No data is ever simulated.

## Documentation

| Doc | Contents |
|---|---|
| [docs/DESIGN.md](docs/DESIGN.md) | Requirements mapping, architecture, dashboards, collaboration, sharing, provenance rules |
| [docs/DATA_SOURCES.md](docs/DATA_SOURCES.md) | Verified endpoints, licenses, citations and caching policy for every source |
| [docs/DATA_MODEL.md](docs/DATA_MODEL.md) | Postgres/PostGIS schema |
| [docs/API.md](docs/API.md) | REST/WebSocket API surface and conventions |
| [docs/ROADMAP.md](docs/ROADMAP.md) | Epics, phases and dependencies (mirrors GitHub issues) |

## Status

Design phase. Implementation is tracked in GitHub issues (epics labeled `epic`).
