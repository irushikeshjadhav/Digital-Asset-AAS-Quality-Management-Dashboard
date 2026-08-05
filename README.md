<h1 align="center">
Digital Asset AAS - Quality Management Dashboard
</h1>
<br>

<p align="center">
<a href="https://www.idta.org"><img src="https://img.shields.io/badge/IDTA-AAS%20Metamodel-1F60C4.svg?style=flat" alt="IDTA"></a>
<a href="https://www.eclipse.org/basyx/"><img src="https://img.shields.io/badge/Eclipse-BaSyX-2C2255.svg?style=flat&logo=eclipse" alt="Eclipse BaSyX"></a>
<a href="https://en.wikipedia.org/wiki/RAMI_4.0"><img src="https://img.shields.io/badge/Industrie%204.0-RAMI%204.0-F46800.svg?style=flat" alt="RAMI 4.0"></a>
</p>

Individual coursework project (Digitalization of ICPS, SS2026, Hochschule Emden/Leer) modeling a factory quality-monitoring dashboard as an Industrie 4.0-compliant **Asset Administration Shell (AAS)**, deployed live as a Type 2 shell on Eclipse BaSyX.

---

## Overview

The Digital Factory at Hochschule Emden/Leer runs three production modules - an Injection Moulding Machine (IMM), a laser engraver, and a delta robot - each exposing process data through its own OPC UA server in its own format. Quality indicators, especially Overall Equipment Effectiveness (OEE = Availability x Performance x Quality Rate), depend on data scattered across all three. A Quality Management Dashboard consolidates these sources into one consistent view for operators and supervisors.

This project models that dashboard **as an AAS** - not as a passive data container, but as a **tool with capabilities**: the submodels record which quality KPIs the dashboard shows, where its data comes from, and how it presents that information, while the authoritative measurements stay in the underlying machine systems.

The AAS was built first as a **Type 1 AASX package** (AASX Package Explorer) and then deployed as a **Type 2 shell** on Eclipse BaSyX, where it runs as a live service with all submodels readable over a standard REST API.

---

## Submodels

| Submodel | Basis | Purpose |
|---|---|---|
| Digital Nameplate | IDTA 02006 v3.0 | Identity and manufacturer metadata |
| Software Nameplate | IDTA 02007 v1.0 | Software-specific metadata (type, version, installation) |
| TimeSeries | IDTA 02008 v1.1 | Historical KPI time-series records (via LinkedSegments) |
| DisplayedKPIs | Custom | Capability description: which KPIs the dashboard displays |
| DashboardLayout | Custom | UI layout configuration |

**DisplayedKPIs** is the core submodel, organized into three collections: `QualityOverview` (plant-level PlantOEE, PlantQualityRate, OverallYield), `PerMachineQuality` (a full OEE breakdown for the IMM; a single quality indicator each for the laser engraver and delta robot), and `QualityAlerts` (active alerts, severity, latest message).

**TimeSeries** uses the IDTA LinkedSegments pattern - it references external data segments (served by a REST API backend) rather than embedding raw history inline, keeping the AAS itself lightweight.

---

## Concept Descriptions

Four custom Concept Descriptions ground the OEE properties in a formally specified semantic model, using the IEC 61360 data specification template (PreferredName, Definition, DataType, Unit):

- **OEE** - Overall Equipment Effectiveness as the product of the three pillars (`REAL_MEASURE`, %)
- **Availability** - ratio of actual to planned operating time (`REAL_MEASURE`, %)
- **Performance** - ratio of actual to ideal cycle throughput (`REAL_MEASURE`, %)
- **Quality** - ratio of conforming to total produced parts (`REAL_MEASURE`, %)

Each DisplayedKPIs property referencing these pillars is linked through a `semanticId`, so any consumer can resolve the exact meaning of every value without vendor-specific documentation.

---

## Architecture & Deployment

Three cooperating services make up the deployment:

| Service | Role | Port |
|---|---|---|
| BaSyX AAS Environment | Stores AAS shells/submodels, exposes the IDTA Submodel Repository REST API (authoritative interface) | 8081 |
| BaSyX AAS UI | Browser-based repository browser (supplementary view) | 3001 |
| REST API service | Lightweight HTTP service backed by TimescaleDB, serves historical KPI records referenced by TimeSeries | 8000 |

Three protocols are active end-to-end: **OPC UA** at the field level (each production module), **MQTT** for live distribution (ingestion service to Mosquitto broker), and **HTTP/REST** for AAS access (IDTA Submodel Repository API, submodel IDs passed as Base64url-encoded strings).

At startup, the AASX package is uploaded to BaSyX via its REST API, making all five submodels immediately addressable as individual REST resources - e.g.:

```
GET /submodels/{submodel-id-base64}/submodel-elements
```

---

## Identifier Scheme

All AAS and submodel identifiers use a consistent namespace:

- **AAS ID:** `https://hs-emden-leer.de/ids/aas/QualityManagementDashboard_7027952`
- **globalAssetId:** `https://hs-emden-leer.de/ids/asset/QualityManagementDashboard_7027952`
- **Concept Descriptions:** `https://hs-emden-leer.de/ids/cd/OEE/...`

Each submodel carries an instance-specific ID under the same namespace, required because IDTA template submodels ship with generic `admin-shell.io` IDs that would otherwise collide across multiple deployed instances.

---

## RAMI 4.0 Positioning

- **Layer axis:** Information (structured KPIs, time-series, metadata), Functional (aggregation, visualization, alerts), Business (quality monitoring, root-cause analysis, reporting)
- **Life-cycle axis (IEC 62890):** Instance, on the Production/Usage side - represents one concrete deployed dashboard
- **Hierarchy axis (IEC 62264):** Work Centre level - consolidates data from three machines into one operational view for operators, supervisors, and higher-level production management

---

## Tools Used

| Tool | Purpose |
|---|---|
| AASX Package Explorer | Primary AAS authoring, submodel/element creation, AASX export |
| Eclipse BaSyX | Type 2 AAS server runtime |
| IDTA submodel templates | Basis for the three standard submodels (02006, 02007, 02008) |
| REST API service | Historical TimeSeries backend (reads TimescaleDB) |

---

## Demonstration

Type 2 behaviour was validated two ways: directly querying the live BaSyX Submodel Repository REST API, and browsing the deployed shell through the BaSyX AAS UI - confirming real, populated instance data (not an empty template) is served for every submodel.

---

## Repository Structure

```
.
├── aasx/                     # Type 1 AASX package export
├── submodels/                # Individual submodel JSON exports
├── concept-descriptions/     # OEE concept description JSON exports
└── docs/                     # Architecture diagrams and UI screenshots
```

---

## Known Limitations

1. The `aas-gui` container has a known `config.json` environment-variable issue that prevented reliable UI connectivity to the backend during this project. Type 2 behaviour was therefore validated through direct REST API calls and the BaSyX browser interface instead.
2. The current prototype works with representative KPI values rather than a continuous live feed from the running production modules; full real-time integration is future work.

---

## Future Outlook

- **Live data feed** from the three production modules through the ingestion pipeline, replacing representative values with continuous real-time measurements.
- **Type 3 AAS**: giving the shell active behaviour so it can interact with other shells through an I4.0 language.
- **Cross-referencing** the dashboard AAS with individual machine AASs via formal `ExternalReference` links, making the Digital Thread machine-navigable.
- **MES integration** as a consumer of the Submodel Repository API, realizing the full IEC 62264 value chain from field-device data to enterprise-level quality decisions.

---

## License

This project is released under the MIT License - see `LICENSE` for details.
