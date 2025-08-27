# Elastic Ingest Series: Ingesting SNMP Data 101

Welcome to the next session in the Elastic Ingest Series! This webinar introduces ingesting SNMP data from network devices using Logstash's SNMP input plugin. We'll explore SNMP fundamentals (including OIDs and MIBs), secure connectivity across versions 1, 2c, and 3, testing with `snmpwalk`, incorporating generic MIBs for output field mapping, and formatting data for the Elastic Common Schema (ECS) to enable visualization in Kibana.

Building on previous sessions (e.g., filter plugins from [2025-03-27](../2025-03-27/README.md) and enrichment processors from [2025-04-23](../2025-04-23/README.md)), this session lays the foundation for advanced SNMP topics like traps in our upcoming "Ingesting SNMP Traps 201" webinar.

## Agenda (60 minutes)

- **Introduction and SNMP Basics (15 minutes)**: Overview of SNMP, its role in network monitoring, key concepts (OIDs as unique identifiers, MIBs as translation dictionaries for output), and the Logstash SNMP input plugin. We'll discuss how to find numeric OIDs and why they're required in configurations.
- **Connectivity and Authentication (10 minutes)**: Configuring SNMP v1, v2c, and v3; testing connectivity with `snmpwalk`; troubleshooting with `stdout` and `rubydebug` to inspect `@metadata`.
- **Incorporating MIBs and Formatting Data (10 minutes)**: Using bundled generic MIBs (e.g., SNMPv2-MIB, IF-MIB), finding numeric OIDs, advanced options like `oid_mapping_format`, and Logstash filters for renaming and structuring output (custom MIBs teased for 201 session).
- **ECS Mapping and Visualization Tips (5 minutes)**: Aligning SNMP fields to ECS fields (with `ecs_compatibility` enabled) for querying and Kibana dashboards.
- **Live Demo (20 minutes)**: Polling a network device (using generic switches or a simulator), applying filters, and ingesting into Elasticsearch, with troubleshooting steps.
- **Q&A (10-15 minutes)**: Open floor for attendee questions.

## Prerequisites

To follow along, you'll need:

- An Elastic Stack deployment (Elasticsearch, Kibana, Logstash) version 8.x or later. (Free tier on [Elastic Cloud](https://www.elastic.co/cloud) is sufficient.)
- Basic familiarity with Logstash configurations (inputs, filters, outputs) from prior sessions (e.g., [2025-03-27](../2025-03-27/README.md)).
- Access to an SNMP-enabled device (e.g., router, switch). Alternatively, use a simulator like [snmpsim](https://github.com/etingof/snmpsim).
- Logstash SNMP integration plugin: Install with `bin/logstash-plugin install logstash-integration-snmp` if not bundled.
- Tools: `snmpwalk` and `snmptranslate` (install via `apt install snmp` on Linux or equivalent) for testing and finding OIDs.
- A site like [Observium MIB Browser](https://mibs.observium.org/mib/) to browse MIBs and find corresponding numeric OIDs.
- Review the [SNMP input plugin docs](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-snmp.html) and [ECS Reference](https://www.elastic.co/guide/en/ecs/current/index.html).
- Enable ECS compatibility: Set `pipeline.ecs_compatibility: v8` in `logstash.yml` for global support, or `ecs_compatibility => "v8"` per input.

## Setup

1. Clone this repo: `git clone https://github.com/untergeek/ingest_with_aaron.git`.
2. Navigate to `2025-08-27/` for session files.
3. Configure `logstash.yml` with `pipeline.ecs_compatibility: v8` (recommended) or update configs with `ecs_compatibility => "v8"`.
4. Start Logstash with a sample config (see [live_demo.md](./live_demo.md)).
5. Open Kibana to verify ingested data.

## Detailed Topics

Explore these companion files for in-depth guidance:

- [snmp_basics.md](./snmp_basics.md): SNMP protocol, OIDs, MIBs (as dictionaries for output mapping), and Logstash integration.
- [connectivity_auth.md](./connectivity_auth.md): Configuring v1, v2c, v3 (community strings, USM), with `snmpwalk` examples, troubleshooting with `stdout`/`rubydebug`, and `@metadata` usage.
- [mibs_formatting.md](./mibs_formatting.md): Using bundled generic MIBs (e.g., IF-MIB, SNMPv2-MIB), finding numeric OIDs, advanced options (e.g., `oid_mapping_format`, `target`), and filters like `mutate` or `grok` (custom MIBs covered in 201).
- [ecs_mapping.md](./ecs_mapping.md): Mapping SNMP data to ECS fields (e.g., `network.protocol: "snmp"`, `system.network` metrics) for Kibana.
- [live_demo.md](./live_demo.md): Step-by-step demo configs for polling and ingestion.
- [error_handling.md](./error_handling.md): Troubleshooting common issues (e.g., timeouts, auth failures, invalid OIDs).
- [summary.md](./summary.md): Key takeaways, homework (poll your own device), and teaser for "Ingesting SNMP Traps 201".

## Demo Overview

The live demo will showcase:

- Configuring the SNMP input plugin to poll a switch using v2c (community string: `public`), with `target => "snmp"` and ECS compatibility.
- Using bundled MIBs for field name mapping (e.g., setting `oid_mapping_format => "ruby_snmp"` for readable names like `SNMPv2-MIB::sysDescr.0`).
- Polling scalar OIDs (e.g., `1.3.6.1.2.1.1.1.0`) and using tables for interface stats.
- Applying filters (e.g., `mutate` to rename fields, `grok` for parsing) to structure data.
- Mapping to ECS fields (e.g., `host.ip`, `system.network.in.bytes`) for ingestion.
- Verifying data in Kibana with a simple lens visualization, and troubleshooting with `stdout`/`rubydebug` to inspect `@metadata`.

See [live_demo.md](./live_demo.md) for the full config.

## Resources

- Elastic Docs: [SNMP Input Plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-snmp.html), [ECS Reference](https://www.elastic.co/guide/en/ecs/current/index.html).
- Previous Sessions: [Main repo](https://github.com/untergeek/ingest_with_aaron) for archives.
- Community: [Elastic Discuss forums](https://discuss.elastic.co/) for SNMP-related questions.
- OID Lookup: [Observium MIB Browser](https://mibs.observium.org/mib/) for finding numeric OIDs from MIB names.

## Next Steps

Try polling your own device with the demo config! Stay tuned for "Ingesting SNMP Traps 201" (tentative: September 2025), where we'll explore reactive event handling and custom MIBs.

Thank you for joining!
