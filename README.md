# Manufacturing Execution System (MES)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source MES for production scheduling, real-time monitoring, quality, and OEE — accessible to manufacturers shut out of Tier-1 platforms.

The MES market is dominated by commercial Tier-1 platforms (Siemens Opcenter, SAP Digital Manufacturing, AVEVA, Rockwell/Plex) with $500K–$3M per-plant TCO and 12–18 month deployments. This project builds an open, standards-aligned MES that orchestrates work orders, OEE, genealogy, and quality on the shop floor, with AI capabilities embedded from day one rather than bolted on. It targets mid-market manufacturers, regulated-industry sites, and IT/OT teams that need real production orchestration without enterprise-suite lock-in.

---

## Why Manufacturing Execution System (MES)?

- No credible open-source MES covers the full functional scope of genealogy, quality, scheduling, and OEE — the market is entirely commercial
- Tier-1 enterprise platforms (Siemens, SAP) carry $500K–$3M per-plant TCO with implementation adding 50–80% to license costs, excluding most mid-market manufacturers
- IoT overlays like TeepTrak and MachineMetrics deploy fast but only monitor — they cannot orchestrate work orders, enforce process steps, or manage genealogy
- Existing MES platforms capture downtime reason codes but rely on operators to diagnose; AI-driven root-cause analysis on sensor streams is still nascent across all incumbents
- A clear gap exists between enterprise-grade compliance tools (Siemens, SAP) and no-code tools (Tulip) for mid-market pharma and medical device manufacturers needing 21 CFR Part 11 without Tier-1 cost

---

## Key Features

### Production Execution

- Work-order creation, dispatch, and real-time status tracking
- Electronic work instructions and operator guidance with error-proofing gates
- Role-based operator panels with step-guided workflows
- Immutable audit trail of every shop-floor transaction

### OEE and Performance

- Real-time OEE calculation across availability, performance, and quality
- Operator-coded downtime classification with root-cause logging
- Multi-site centralised monitoring with plant-to-plant benchmarking
- Threshold-based alerting via webhook, email, and SMS

### Quality and Traceability

- Inspection plans with pass/fail capture and non-conformance logging
- SPC charting with automated limit-violation alerting
- Lot and serial genealogy from raw material to finished goods
- CAPA workflows for non-conformance resolution

### Scheduling and Materials

- Finite-capacity scheduling with real-time re-sequencing on disruptions
- Material and inventory tracking with automatic consumption updates
- BOM and routing management
- Kanban and pull-based production control patterns

### Connectivity and Integration

- OPC-UA and MQTT machine connectivity via no-code configuration
- REST API for ERP integration (SAP, Oracle, Epicor)
- ISA-95 B2MML message exchange for enterprise interoperability
- Webhook-based event triggers for external systems

---

## AI-Native Advantage

Unlike incumbents that capture data and leave diagnosis to operators, this project embeds AI in the execution loop. Real-time OEE root-cause analysis applies machine learning to sensor streams, production logs, and quality data to pinpoint availability, performance, and quality losses within minutes. Dynamic AI-driven scheduling resequences jobs in response to breakdowns, material shortages, and rush orders rather than fixing the plan at shift start. Edge-deployed computer vision detects defects at production speed and feeds data back to the process control loop, while predictive first-pass yield modelling flags upstream parameter drift before quality escapes occur. A natural language interface lets shift supervisors query performance, downtime, and quality trends in plain language, replacing manual end-of-shift report compilation.

---

## Tech Stack & Deployment

The platform aligns with ISA-95 / IEC 62264 (enterprise-to-shop-floor data models), ISA-88 / IEC 61512 (batch and recipe control), OPC-UA (IEC 62541) for machine connectivity, and IEC 62443 for industrial cybersecurity. Regulated deployments target 21 CFR Part 11 (pharma) and IATF 16949 (automotive). Expected deployment modes include self-hosted on-prem (for air-gapped and regulated sites), cloud SaaS, and hybrid edge-plus-cloud for sites running computer vision inference at the machine. ERP integration uses REST APIs and ISA-95 B2MML messaging; machine integration uses OPC-UA and MQTT with no-code configuration.

---

## Market Context

The global MES market is estimated at USD 18.6–20.6 billion in 2026, with projections of USD 25.78 billion by 2030, driven by Industry 4.0 investment and regulatory compliance. Tier-1 platforms charge $500K–$3M per plant in TCO; IoT overlays start around $200/machine/year; mid-market cloud-native tools sit between. Primary buyers are VPs of Manufacturing, Plant Managers, IT/OT integration architects, Quality Directors, and COOs at automotive, pharma, food, and electronics manufacturers — where 1–2% OEE improvement at large plants delivers millions in annual value.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
