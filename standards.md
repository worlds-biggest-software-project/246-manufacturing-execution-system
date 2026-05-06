# Standards & API Reference

> Project: Manufacturing Execution System (MES) · Generated: 2026-05-03

---

## Industry Standards & Specifications

### ISA / IEC Standards

**ANSI/ISA-95 / IEC 62264 — Enterprise-Control System Integration**
- URL: https://www.isa.org/standards-and-publications/isa-standards/isa-95-standard
- Wikipedia overview: https://en.wikipedia.org/wiki/ANSI/ISA-95
- The foundational MES standard. Defines a five-level functional hierarchy (Levels 0–4), data models for production orders, schedules, performance results, and resource information, and the interface between ERP (Level 4) and manufacturing operations management (Level 3). ISA-95 Part 1 (Models & Terminology) was updated as ANSI/ISA-95.00.01-2025. An XML implementation (B2MML — Business to Manufacturing Markup Language) provides ready-to-use schemas for ERP–MES message exchange. All major MES vendors implement ISA-95 data models; any AI-native MES should align to these models for interoperability.

**ANSI/ISA-88 / IEC 61512 — Batch Control**
- URL: https://www.isa.org/standards-and-publications/isa-standards/isa-88-standards
- Wikipedia: https://en.wikipedia.org/wiki/ISA-88
- Defines equipment models (physical, procedural) and recipe structures for batch manufacturing. Specifies procedural hierarchy: procedure → unit procedure → operation → phase. Used in pharmaceutical, food & beverage, and chemical process MES deployments. A conformant MES should map recipe management and batch execution to the ISA-88 equipment and procedural models.

**ISA/IEC 62443 — Security for Industrial Automation and Control Systems (IACS)**
- URL: https://www.isa.org/standards-and-publications/isa-standards/isa-iec-62443-series-of-standards
- Overview: https://www.dragos.com/blog/isa-iec-62443-concepts
- OT cybersecurity framework organised into four levels (SL 1–4) and seven foundational requirements including identification and authentication, use control, system integrity, data confidentiality, restricted data flow, and timely event response. Increasingly a vendor short-list requirement for MES procurement. An AI-native MES should implement SL 2 controls as a baseline, including role-based authentication, encrypted communications, and network segmentation between IT and OT layers.

### ISO Standards

**ISO 22400-1:2014 / ISO 22400-2:2014 — KPIs for Manufacturing Operations Management**
- Part 1 (Overview): https://www.iso.org/standard/56847.html
- Part 2 (Definitions): https://www.iso.org/standard/54497.html
- Defines 34 standardised KPIs for manufacturing operations management, including two definitions of OEE (Overall Equipment Effectiveness), throughput, yield, scrap rate, mean time between failures, and 30 additional indicators. Part 2 specifies the calculation method for each KPI. Any MES claiming ISO-aligned OEE reporting should implement these definitions to allow cross-plant and cross-vendor benchmarking.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Provides the base quality management framework requirements (process documentation, corrective action, management review) that most MES quality modules are designed to satisfy. IATF 16949 and 21 CFR Part 11 both build on ISO 9001 foundations.

**ISO 50001:2018 — Energy Management Systems**
- URL: https://www.iso.org/standard/69426.html
- Specifies requirements for energy performance measurement and improvement. Increasingly integrated into MES energy KPI modules as manufacturers pursue sustainability reporting obligations.

### Automotive Standards

**IATF 16949:2016 — Automotive Quality Management System**
- URL: https://www.iatfglobaloversight.org/iatf-169492016/about/
- Overview: https://quality-one.com/iatf-16949/
- Mandatory quality management standard for automotive supply-chain manufacturers. Drives MES requirements for: full component traceability (raw material to vehicle), PPAP (Production Part Approval Process) documentation and submission, SPC monitoring of key product characteristics, and long-duration record retention (15+ years for ECU/firmware records). An automotive-focused MES module should generate PPAP-compliant PSW (Part Submission Warrant) packages and maintain forward and reverse traceability chains.

### Pharmaceutical & Life Sciences Standards

**US FDA 21 CFR Part 11 — Electronic Records and Electronic Signatures**
- URL: https://www.fda.gov/regulatory-information/search-fda-guidance-documents/part-11-electronic-records-electronic-signatures-scope-and-application
- Guide: https://tulip.co/blog/manufacturers-guide-to-21-cfr-part-11-compliance/
- Governs the use of electronic records and signatures in FDA-regulated manufacturing (pharma, biotech, medical devices). Compliance requirements for an MES include: validated system with documented IQ/OQ/PQ, secure and time-stamped audit trail of all record changes, e-signature linked to a single authenticated user with reason and timestamp, and record retention and retrieval capability. Any MES targeting pharmaceutical customers must implement a validated 21 CFR Part 11 compliance pack.

**EU Annex 11 — Computerised Systems (EU GMP)**
- URL: https://health.ec.europa.eu/system/files/2016-11/annex11_01-2011_en_0.pdf
- European equivalent of 21 CFR Part 11 for GMP-regulated manufacturers. Specifies lifecycle validation, data integrity, backup, and electronic signature requirements for computerised manufacturing systems. MES deployments in EU pharmaceutical plants must satisfy Annex 11 in addition to (or instead of) 21 CFR Part 11.

### Machine Connectivity & Communication Standards

**OPC-UA (IEC 62541) — Unified Architecture**
- OPC Foundation: https://opcfoundation.org/developer-tools/samples-and-tools-unified-architecture
- Online reference: https://reference.opcfoundation.org/
- GitHub (OPC Foundation): https://github.com/opcfoundation
- The primary machine-to-machine communication standard for shop-floor data exchange. Provides a platform-independent, secure, publish-subscribe and request-response architecture for real-time data from PLCs, CNCs, robots, and sensors. SDKs available in .NET, Java, C, C++, Python, and JavaScript. OPC-UA companion specifications (e.g., ISA-95 MES-MOM, CNC/Umati, Machinery) extend the base standard to domain-specific information models. An MES should implement OPC-UA as the primary machine integration protocol.

**MQTT / Sparkplug B**
- Sparkplug Specification (Eclipse Foundation): https://sparkplug.eclipse.org/specification/
- HiveMQ introduction: https://www.hivemq.com/blog/mqtt-sparkplug-essentials-part-1-introduction/
- MQTT is the lightweight publish-subscribe messaging protocol used in IIoT for low-bandwidth machine data streaming. Sparkplug B (Eclipse Foundation specification, currently v3.0) extends MQTT with a standardised topic namespace, payload structure (Google Protocol Buffers), and state management (Birth/Death certificates for device awareness). Sparkplug B reduces custom adapter logic at the edge and is increasingly preferred for cloud-native MES integrations where OPC-UA server infrastructure is not available.

**MTConnect**
- URL: https://www.mtconnect.org/
- Open standard for CNC machine tool data exchange, developed by AMT (Association for Manufacturing Technology). Defines a REST/HTTP-based protocol and XML data model for streaming cutting tool, spindle, and program data from CNC controllers. Widely supported by FANUC, Mazak, Haas, and Okuma. A discrete-manufacturing MES should support MTConnect as an alternative connectivity path for CNC machine fleets.

---

## Data Model & API Specifications

**B2MML (Business to Manufacturing Markup Language)**
- URL: https://www.mesa.org/b2mml/
- XML implementation of the ISA-95 data models, published by MESA International. Provides ready-to-use schemas for work orders, production schedules, performance results, product definitions, and resource management. Enables vendor-neutral ERP–MES message exchange without proprietary middleware. An AI-native MES should expose ISA-95-aligned data objects and support B2MML import/export for interoperability with existing ERP integrations.

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- De-facto standard for describing REST APIs. All MES REST APIs should be documented as OpenAPI 3.1 specifications to enable developer tooling (auto-generated SDKs, interactive documentation, API testing). Published API catalogues (SAP Business Accelerator Hub, Tulip Developer Portal, Plex Developer Portal) all use OpenAPI-compatible formats.

**JSON Schema**
- URL: https://json-schema.org/
- Standard for describing the structure and validation rules of JSON data payloads. Relevant for MES API payload validation, event schemas (work-order completion events, quality result payloads), and configuration schema documentation.

---

## Security & Authentication Standards

**OAuth 2.0 / OpenID Connect**
- OAuth 2.0: https://oauth.net/2/
- OpenID Connect: https://openid.net/connect/
- Standard authorisation and identity federation protocols. All reviewed MES platforms (SAP, Tulip, Plex) use OAuth 2.0 for API authentication. An AI-native MES API should implement OAuth 2.0 client-credentials flow for machine-to-machine access and authorisation-code flow with PKCE for user-interactive sessions.

**NIST SP 800-82 — Guide to OT Security**
- URL: https://csrc.nist.gov/publications/detail/sp/800-82/rev-3/final
- US federal guidance on securing industrial control and OT environments. Complements ISA/IEC 62443 with practical implementation guidance on network segmentation, remote access, patch management, and incident response for OT systems including MES.

---

## Similar Products — Developer Documentation & APIs

### Tulip
- **Description:** No-code manufacturing operations platform; composable MES with IIoT connectivity, operator guidance, and AI-powered workflow building.
- **API Documentation:** https://docs.tulip.com/integrating/overview/
- **Core API Reference:** https://docs.tulip.com/integrating/tulip-api/
- **Developer Portal:** https://support.tulip.co/api
- **Standards:** REST/JSON; namespaced, versioned API (apps, tables, connectors, stations, automations)
- **Authentication:** OAuth 2.0 / API token with configurable scopes
- **Connectors:** HTTP, SQL, OPC-UA, MQTT configurable in-platform

### SAP Digital Manufacturing Cloud
- **Description:** ERP-integrated cloud MES providing closed-loop production scheduling, quality management, and OEE analytics natively connected to SAP S/4HANA.
- **API Documentation (OData V4):** https://api.sap.com/package/SAPDigitalManufacturingCloud/overview
- **REST API Hub:** https://api.sap.com/package/SAPDigitalManufacturingCloud/rest
- **Developer Guide:** https://help.sap.com/docs/sap-digital-manufacturing/business-process-extensions-developer-s-guide/api-services-restful-and-odata
- **Help Portal:** https://help.sap.com/docs/sap-digital-manufacturing
- **Standards:** REST/JSON; OData V4; SAP BTP event mesh
- **Authentication:** OAuth 2.0 (client credentials); API endpoint: api.eu20.dmc.cloud.sap

### Plex Smart Manufacturing (Rockwell)
- **Description:** Cloud-native smart manufacturing platform with MES, quality, inventory, and scheduling; strong automotive pedigree.
- **Developer Portal:** https://developers.plex.com/docs
- **ERP API Overview:** https://plex.rockwellautomation.com/en-us/resources/enterprise-resource-planning-erp-apis.html
- **Community Node.js Client (unofficial):** https://github.com/machinemetrics/node-plex-systems-api
- **Standards:** REST/JSON; role-based API security
- **Authentication:** API key / OAuth 2.0

### Siemens Opcenter
- **Description:** Tier-1 enterprise MES covering discrete, process, pharmaceutical, and electronics manufacturing with deep compliance and PLM integration.
- **Product Page:** https://plm.sw.siemens.com/en-US/opcenter/execution/
- **API / Integration:** https://plm.sw.siemens.com/en-US/opcenter/ (REST APIs, B2MML, OPC-UA documented in product documentation)
- **Standards:** REST, SOAP, B2MML/ISA-95, OPC-UA
- **Authentication:** LDAP / Active Directory; role-based access

### AVEVA MES
- **Description:** Process and hybrid manufacturing MES with strong genealogy, quality, and SCADA/historian integration.
- **Product Page:** https://www.aveva.com/en/products/manufacturing-execution-system/
- **AVEVA Connect Cloud:** https://www.aveva.com/en/platform/aveva-connect/
- **Standards:** OPC-DA, OPC-UA, REST API for ERP integration
- **Authentication:** Active Directory / AVEVA Secure Connect

### MachineMetrics
- **Description:** Universal machine data platform with AI-native OEE root-cause analysis (Max AI) and MES evolution for discrete manufacturing.
- **Main Site:** https://www.machinemetrics.com/
- **Machine Connectivity Protocols:** MTConnect, OPC-UA, FANUC FOCAS, Haas NGC, Mazak
- **Standards:** REST API; MTConnect; OPC-UA
- **Authentication:** API key; OAuth 2.0 for enterprise SSO

### OPC Foundation SDKs (open-source reference implementation)
- **Description:** Open-source OPC-UA stacks for .NET Standard, enabling machine connectivity development without proprietary middleware.
- **GitHub:** https://github.com/OPCFoundation/UA-.NETStandard
- **Documentation:** https://opcfoundation.github.io/UA-.NETStandard/
- **Standards:** IEC 62541 (OPC-UA)
- **Languages:** .NET Standard; community ports in Python, Java, C, JavaScript (Node)

---

## Notes

- **Unified Namespace (UNS)**: An architectural pattern gaining traction in Industry 4.0 implementations where OPC-UA and MQTT/Sparkplug B are combined into a single message broker (typically HiveMQ, EMQX, or AWS IoT Core) that acts as a real-time data backbone for the plant. An AI-native MES should be designed to consume data from a UNS rather than requiring point-to-point machine integrations.
- **MCP (Model Context Protocol)**: The Anthropic Model Context Protocol is relevant if the MES exposes AI agent tools for querying production data, generating shift reports, or triggering corrective actions. An MES MCP server would expose tools such as `get_oee_summary`, `query_work_order_status`, `log_downtime_event`, and `get_quality_trend`.
- **MESA International**: The Manufacturing Enterprise Solutions Association publishes integration guidelines, the B2MML schema library, and best-practice frameworks that supplement ISA-95. URL: https://www.mesa.org/
- **B2MML schemas**: Freely downloadable from MESA International and widely used as the XML message format for ISA-95-aligned ERP–MES integration. No licensing fee for implementation.
