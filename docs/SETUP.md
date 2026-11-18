# Setup Guide: Splunk on Proxmox for UniFi Gateway Logs

This guide builds an all-in-one Splunk Enterprise 10.x server in a Proxmox VM and sends a UniFi gateway's logs to it (built and tested on a UDR7 running UniFi Network 11.0.x). It ends with the `unifi` app from this repo installed: CIM fields plus five Dashboard Studio dashboards.

Placeholders used throughout:

| Placeholder | Meaning |
|---|---|
| `<SPLUNK_IP>` | IP of the Splunk VM (give it a fixed DHCP reservation on the gateway) |
| `<GATEWAY_IP>` | LAN IP of the UniFi gateway, which is the source address of its syslog |
| `<LAN_CIDR>` | Your LAN subnet(s) allowed to reach Splunk Web, SSH and forwarders |
| `<VMID>` | Proxmox VM ID |
| `<ADMIN_USER>` | Your OS login and Splunk admin username |
| `<WG_PORT>` | Your WireGuard VPN server port, if you run one on the gateway |

## 1. Architecture

```
UniFi gateway ──UDP 5514 (syslog: CEF + iptables lines)──►  Splunk VM (all-in-one)
              ──UDP 10514 (Netconsole kernel logs)───────►   index = net
                                                              app   = unifi
```

- No syslog tier: the gateway sends straight to UDP inputs on the indexer.
- **Index `net`:** 30-day retention, 150 GB cap.
- **Summary data:** the built-in `summary` index, for later summary-based dashboards.
- **License:** Developer license (the Free license removes logins and alerting).

## 2. Proxmox VM

Sizing used: 4 vCPU (type **host**, so the KV store gets AVX), 6 GB RAM with ballooning off, 200 GB thin disk.

Check the host's free thin-pool space before you pick a disk size. Thin pools can be overcommitted, and a full pool takes every guest down with it. Keep the Splunk index caps below the disk size.

```bash
pveam update   # not required for the ISO, shown for completeness
wget -q --show-progress -P /var/lib/vz/template/iso https://releases.ubuntu.com/24.04/ubuntu-24.04.5-live-server-amd64.iso
cd /var/lib/vz/template/iso && curl -s https://releases.ubuntu.com/24.04/SHA256SUMS | grep 'ubuntu-24.04.5-live-server-amd64.iso' | sha256sum -c -
```

```bash
qm create <VMID> --name Splunk --ostype l26 --machine q35 --bios seabios --cpu host --sockets 1 --cores 4 --memory 6144 --balloon 0 --scsihw virtio-scsi-single --scsi0 local-lvm:200,discard=on,iothread=1,ssd=1 --ide2 local:iso/ubuntu-24.04.5-live-server-amd64.iso,media=cdrom --boot 'order=scsi0;ide2' --net0 virtio,bridge=vmbr0 --agent enabled=1 --onboot 1
qm start <VMID>
```

`discard=on` hands space back to the thin pool when Splunk freezes buckets.

## 3. Ubuntu 24.04 install choices

- **Kernel and install type:** "Try or Install Ubuntu Server" (GA kernel), full Ubuntu Server (not minimized), no third-party drivers.
- **Network:** DHCP. Then make the address a fixed reservation on the gateway.
- **Storage:** "Use an entire disk" + LVM, no LUKS. On the summary screen, **edit `ubuntu-lv` and grow it to the maximum**. The installer only gives `/` about half the volume group by default.
- **Hostname:** `splunk`. Install OpenSSH. No snaps.
- **After first boot:**

```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install qemu-guest-agent
```

On the Proxmox host, detach the ISO and reboot:

```bash
qm set <VMID> --ide2 none,media=cdrom && qm reboot <VMID>
```

The guest agent starts on boot (`systemctl is-active qemu-guest-agent`). The VM stays on **UTC**.

## 4. OS prep for Splunk

### Disable Transparent Huge Pages

```bash
sudo tee /etc/systemd/system/disable-thp.service >/dev/null <<'EOF'
[Unit]
Description=Disable Transparent Huge Pages (Splunk)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=Splunkd.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled; echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now disable-thp.service
cat /sys/kernel/mm/transparent_hugepage/enabled /sys/kernel/mm/transparent_hugepage/defrag   # both [never]
```

### Host firewall (ufw)

Add the rules **before** enabling ufw, so your SSH session isn't cut off.

```bash
sudo ufw default deny incoming && sudo ufw default allow outgoing
sudo ufw allow from <LAN_CIDR> to any port 22 proto tcp comment 'SSH'
sudo ufw allow from <LAN_CIDR> to any port 8000 proto tcp comment 'Splunk Web'
sudo ufw allow from <LAN_CIDR> to any port 8089 proto tcp comment 'Splunk mgmt'
sudo ufw allow from <LAN_CIDR> to any port 9997 proto tcp comment 'UF forwarding'
sudo ufw allow from <GATEWAY_IP> to any port 5514 proto udp comment 'UniFi syslog'
sudo ufw allow from <GATEWAY_IP> to any port 10514 proto udp comment 'UniFi netconsole'
sudo ufw --force enable && sudo ufw status numbered
```

Forwarders on other VLANs need their own 9997 rule.

## 5. Install Splunk Enterprise 10.x

Get the `.deb` wget link from splunk.com (it needs a login), download it to `/tmp`, and verify it:

```bash
cd /tmp && f=splunk-<version>-<build>-linux-amd64.deb && [ "$(curl -s https://download.splunk.com/products/splunk/releases/<version>/linux/$f.sha512 | grep -oE '[0-9a-f]{128}')" = "$(sha512sum $f | cut -d' ' -f1)" ] && echo MATCH || echo MISMATCH
sudo dpkg -i /tmp/$f
```

The postinst may print `find: '/opt/splunk/lib/python3.7/site-packages': No such file or directory`. That's harmless.

First start as the `splunk` service account (the .deb creates it). This prompts for the admin user:

```bash
sudo -u splunk /opt/splunk/bin/splunk start --accept-license
sudo -u splunk /opt/splunk/bin/splunk stop && sudo /opt/splunk/bin/splunk enable boot-start -systemd-managed 1 -user splunk -group splunk
sudo systemctl start Splunkd && systemctl status Splunkd --no-pager | head -n 6
```

Optional: **Settings → Server settings → General settings → Session timeout** = `1440` (minutes) keeps you logged in for a day. If you still get logged out, also set `web.conf [settings] ui_inactivity_timeout = 1440`.

## 6. Install the `unifi` app

Copy `unifi/` from this repo to `/opt/splunk/etc/apps/unifi`, owned by `splunk`. To update single files, copy them from your PC and install each one:

```powershell
scp .\unifi\local\data\ui\views\unifi_overview.xml <ADMIN_USER>@<SPLUNK_IP>:/tmp/
```

```bash
sudo install -o splunk -g splunk -m 664 /tmp/unifi_overview.xml /opt/splunk/etc/apps/unifi/local/data/ui/views/unifi_overview.xml
```

Then pick the reload that matches what changed:

| Change | Reload |
|---|---|
| props/eventtypes/tags/macros (search-time) | Nothing needed, or `http://<SPLUNK_IP>:8000/en-US/debug/refresh` |
| New or changed dashboards | `debug/refresh`, then Ctrl+F5 in the browser |
| `inputs.conf`, `indexes.conf`, `TZ` in props, **nav** | `sudo systemctl restart Splunkd` |

**Host names:** after the first install, run `| savedsearch "UniFi - Build host lookup"` once (it then runs every 15 minutes) and check `| inputlookup unifi_hosts`. Add static-IP devices that never use DHCP to `lookups/unifi_hosts_manual.csv` on the server (`ip,hostname`). Those entries override the automatic names.

Verify with `sha256sum` after every copy. Pasting long heredocs over SSH is fragile, especially when a sudo password prompt interrupts the paste.

## 7. UniFi gateway configuration

### Activity Logging (Syslog)
In UniFi Network, open **Settings** and search for "Syslog". Then set:

- **Activity Logging:** SIEM Server → `<SPLUNK_IP>`, port **5514**
- **Contents:** select all categories (Devices: Gateway, Access Points, Switches. System Log: Admin Activity, Clients, Critical, Devices, Security Detections, Triggers, Updates, VPN, Firewall Default Policy)
- **Debug Logs:** off
- **Netconsole:** on → `<SPLUNK_IP>`, port **10514**. It only carries kernel messages, so it can be silent for long periods.
- **Logging levels:** Verbose while building. Consider Auto later to reduce noise.

### Outbound traffic logging
Create a zone policy **Internal → External**:
- **Name and action:** "Log Internal to External", **Allow**, Any → Any.
- **IP version and protocol:** Both / All.
- **Connection State:** Custom = **New** only. You get one event per connection, and return traffic is still handled by the built-ins.
- **Logging:** Syslog Logging on.
- **Order:** reorder it to the **top** of the Internal → External list. Otherwise a custom Allow above it swallows those connections unlogged. Both are Allow policies, so nothing changes security-wise.

### Inbound attempts (scans and probes) logging
Unsolicited inbound traffic lands in **External → Gateway**, where every policy is built-in and locked. On 11.0.x, the "Firewall Default Policy" logging category did **not** emit these drops.

So create a scoped custom policy **External → Gateway**:
- **Name and action:** "Log Block External to Gateway", **Block**, Any → Any.
- **Protocol:** **TCP** only, with **Match Opposite unchecked**. With it checked, the policy would block everything *except* TCP.
- **Connection State:** Custom = **New** only.
- **Logging:** Syslog Logging on.

Why it's scoped this way: custom policies are evaluated **before** the built-ins, and the built-in allows here are UDP or ICMPv6 (WireGuard `<WG_PORT>`, DHCPv6, neighbor discovery). TCP + New therefore blocks exactly what Block All already blocked, now with logging. UniFi will warn "Block All Traffic to Gateway?". That's expected, because LAN management goes through Internal → Gateway, which this policy doesn't touch.

A UDP version needs a Port List object to exclude `<WG_PORT>` and 546.

### Port forwards
Each port forward entry has its own **Syslog Logging** checkbox. Turn it on for every one you want to see.

### IPS and honeypots
In CyberSecure, turn **Intrusion Prevention** on for the networks you want covered. UniFi recommends 3 or fewer networks for performance. Detections and honeypot hits arrive through the **Security Detections** category.

## 8. What the data looks like

| Kind | Format | Example |
|---|---|---|
| Policy traffic (allowed outbound, blocked inbound, port forwards) | Kernel iptables line | `[LAN_WAN-A-10000] DESCR="Log Internal to External" IN=br0 OUT=eth3 SRC=… DST=… PROTO=TCP SPT=… DPT=…` |
| Internal policy blocks, VPN, client, audit and IPS events | CEF | `CEF:0\|Ubiquiti\|UniFi Network\|11.0.x\|203\|Blocked by Firewall\|4\|UNIFIcategory=Security …` |
| Everything else | Plain syslog | `dnsmasq-dhcp`, `coredns` (ad-block JSON), `dpi-flow-stats`, `systemd`, … |

Reading the iptables prefix `[<CHAIN>-<A|D>-<rule id>]`:
- `WAN_LOCAL` = External → Gateway, `LAN_WAN` = Internal → External, `WAN_*` = inbound
- `A` = accept, `D` = drop
- The rule ID changes if you reorder policies.

## 9. Gotchas found during the build

- **8-hour timestamp offset.** The gateway writes its syslog header in local time with **no time zone**, and Splunk on UTC reads it as UTC. Fix it with per-source `TZ` in props, as this app does. CEF events also carry `UNIFIutcTime`, which is handy for checking `_time`.
- **Backlog bursts.** The gateway sometimes sends hours-old events (for example coredns ad-block logs) all at once. They show up as large `_indextime - _time` lag, which is not a parsing problem.
- **LAN noise.** About 60% of the "Internal → External" events never leave the LAN: broadcast, multicast, IGMP, all with `IN=br* OUT=br*`. The `unifi_traffic` eventtype excludes them.
- **Values with spaces.** Automatic KV extraction cuts CEF values off at the first space (policy names, aliases, VPN durations). The `unifi_cef` macro re-extracts them.
- **Host field.** The stock `syslog` sourcetype sets `host` to the gateway's hostname from the header, not its IP.
- **Bare tag searches return 0** unless `net` is among your role's default search indexes. The dashboards use `eventtype=unifi_traffic`, which includes `index=net`.
- **Dashboard Studio maps (10.6):**
  - A static `dataColors` is ignored. Use `rangeValue` with a `context` palette.
  - Use `additionalTooltipFields` to show City, Region and Country on hover.
- **App nav in 10.x** renders as a left sidebar. The dashboards include a thin markdown link strip at the top to get closer to the 9.x top bar. Nav changes need a restart.
- **Line endings.** This repo pins LF (`.gitattributes`), so files copied to Linux are byte-identical and their checksums match.

## 10. Verification searches

```
index=net earliest=-15m | stats count by host, sourcetype
index=net earliest=-15m | eval lag_sec=_indextime-_time | stats count avg(lag_sec) max(lag_sec) by source
eventtype=unifi_traffic | stats count by direction action transport
index=net "IN=" "OUT=" | rex "\[(?<fw_prefix>[^\]]+)\]\s+DESCR=\"(?<descr>[^\"]*)\"" | stats count by fw_prefix descr
index=net "CEF:" | `unifi_cef` | stats count by UNIFIcategory cef_name
```

## 11. Possible next steps

- Summary searches into `summary`, and dashboards built from them.
- A UDP inbound block-logging policy, using a Port List to exclude the VPN and DHCPv6 ports.
- NetFlow/IPFIX for full flow accounting (Splunk Stream or goflow2).
- Outbound logging for other zones.
- A UniFi add-on, if one fits later (Splunkbase 9831 polls the UniFi API and is independent of syslog).
