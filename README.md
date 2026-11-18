# Splunk UniFi App (`unifi`)

A Splunk app for UniFi gateway (UDR7) logs, built on Splunk Enterprise 10.6. It contains the inputs, the index, CIM field mappings, macros and Dashboard Studio dashboards.

## Layout
`unifi/` is the app folder. Copy it to `$SPLUNK_HOME/etc/apps/unifi`. Everything is in `local/`.

| File | Purpose |
|---|---|
| `app.conf` | App label "UniFi", visible |
| `indexes.conf` | Index `net`: 30-day retention, 150 GB cap |
| `inputs.conf` | UDP 5514 → sourcetype `syslog`; UDP 10514 → sourcetype `netconsole` (both to index `net`) |
| `props.conf` | Per-source `TZ = America/Los_Angeles` (the gateway sends local time with no zone), plus CIM Network Traffic fields from the iptables log lines |
| `eventtypes.conf` / `tags.conf` | Eventtype `unifi_traffic` (excludes LAN-only br→br noise), tagged `network` and `communicate` |
| `transforms.conf` | Lookup definitions `unifi_hosts` (auto-built) and `unifi_hosts_manual` (your overrides) |
| `savedsearches.conf` | "UniFi - Build host lookup": every 15 min, builds ip → hostname/mac from DHCP acks and UniFi client events (alias > hostname > DHCP name), keeps 90 days |
| `lookups/*.csv` | **Header-only templates.** The real IP/host data lives only on the server; never commit populated copies |
| `macros.conf` | `unifi_port_service` (port → service/category), `unifi_cef` (parses the CEF header and fields whose values contain spaces) |
| `data/ui/views/` | Dashboards: Overview, Inbound / Threats, Outbound, Ports, Security Events |
| `data/ui/nav/default.xml` | App nav |

## CIM fields (from `[PREFIX-A|D-ID] DESCR=... IN= OUT= SRC= DST= PROTO= SPT= DPT=`)
`action` (allowed/blocked), `direction` (inbound/outbound/internal), `rule`, `rule_id`, `src`/`src_ip`/`src_port`, `dest`/`dest_ip`/`dest_port`, `transport`, `protocol`, `src_interface`/`dest_interface`, `dvc`, `vendor_product`, `fw_chain`, `wan`. Plus `src_host` / `dest_host` from the host lookups (manual entries win).

Every dashboard has an **Internal host** dropdown (token `hostf`), and host tables filter the page when you click a row.

## Deploy
Copy the changed files to the server, install them into `/opt/splunk/etc/apps/unifi/` owned by `splunk`, then run `debug/refresh`. Nav changes need a restart.
