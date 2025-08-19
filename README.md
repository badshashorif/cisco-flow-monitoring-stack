# Cisco Flow Monitoring Stack (pmacct → ClickHouse → Grafana)

**Per-ASN / per-Prefix / per-IP traffic analytics** from Cisco (IOS-XR/NCS, IOS-XE) via **NetFlow/IPFIX** or **sFlow**.
Enriched with **BGP** (from your router/RR) for accurate ASN & prefix mapping.
Data stored in **ClickHouse**, visualized in **Grafana** (auto-provisioned datasource & starter dashboard).

> ✅ Supports NetFlow/IPFIX **and** sFlow
> ✅ BGP enrichment in pmacct (neighbor to router/RR)
> ✅ ClickHouse for high-cardinality group-bys (ASN/prefix/IP)
> ✅ Ready to `docker compose up -d`

---

## Architecture

```
[Cisco NCS/IOS-XR/IOS-XE]
   ├─ NetFlow/IPFIX (UDP/2055) ┐
   └─ sFlow (UDP/6343)         ├──> [pmacct (nfacctd/sfacctd) + BGP peer]
                                └──> [ClickHouse] -> [Grafana]
```

---

## What’s Included

```
cisco-flow-monitoring-stack/
├─ docker-compose.yml
├─ clickhouse/
│  └─ init/01_schema.sql                 # tables: flows_by_asn / flows_by_prefix / flows_by_ip
├─ grafana/
│  └─ provisioning/
│     ├─ datasources/clickhouse.yaml     # auto ClickHouse datasource
│     └─ dashboards/flows.json           # starter dashboard (ASNs/Prefixes/IPs)
└─ pmacct/
   ├─ nfacctd.conf                       # NetFlow/IPFIX collector → ClickHouse writer
   ├─ sfacctd.conf                       # sFlow collector → ClickHouse writer (optional)
   └─ bgp_neighbors.conf                 # pmacct’s BGP peers (router/RR)
```

---

## Requirements

* Docker & Docker Compose
* Router(s) capable of exporting **NetFlow/IPFIX** and/or **sFlow**
* One reachable BGP session from pmacct to your router or route-reflector

---

## Quick Start

1. **Set BGP peers** for pmacct:

```
pmacct/bgp_neighbors.conf
-------------------------
neighbor 10.10.10.1 as 65001
# add more if you want (RR/edges)
```

2. **Pick your collector(s)**

* NetFlow/IPFIX → `pmacct/nfacctd.conf` (UDP/2055)
* sFlow → `pmacct/sfacctd.conf` (UDP/6343)
  (Both can run; comment out one service in compose if you won’t use it.)

3. **Match sampling**
   Set `sampling_rate` in the chosen pmacct conf to match your router’s exporter (e.g., 1000).

4. **Run**

```bash
docker compose up -d
```

5. **Point the router exporter(s)** to the Docker host IP:

* NetFlow/IPFIX → UDP **2055**
* sFlow → UDP **6343**
* BGP to pmacct → TCP **179** (pmacct uses **host network** in compose)

6. **Grafana**
   Open `http://<host-ip>:3000` (first login: admin/admin). The ClickHouse datasource and a starter dashboard are provisioned automatically.

---

## Cisco Config Examples

> Adjust interface names, loopbacks, VRFs, ASNs, and IPs to your environment.

### IOS-XR / NCS — IPFIX (or v9) + Sampler + Interface Attach

```xr
flow exporter-map EXPORT-PMACCT
 destination 192.0.2.50
 source Loopback0
 transport udp 2055
 version ipfix
 template data timeout 60
 options interface-table timeout 600

flow monitor-map MON-IPv4
 record ipv4
 exporter EXPORT-PMACCT
 cache timeout active 60

sampler-map SAM-1
 random 1 out-of 1000

interface TenGigE0/0/0/0
 ipv4 flow monitor MON-IPv4 sampler SAM-1 ingress
 ipv4 flow monitor MON-IPv4 sampler SAM-1 egress
!
```

### IOS-XR — sFlow

```xr
monitoring sflow
  collector ipv4 192.0.2.50 6343
  agent ipv4 Loopback0
  sampling-rate 2000
  polling-interval 30
  interface TenGigE0/0/0/0
    enable
  !
!
```

### IOS-XR — BGP peering to pmacct (for enrichment)

```xr
route-policy ALL
  pass
end-policy

router bgp 65001
 bgp router-id 10.10.10.1
 neighbor 192.0.2.50
  remote-as 65099
  update-source Loopback0
  address-family ipv4 unicast
   send-community-ebgp
   route-policy ALL in
   route-policy ALL out
  !
!
```

---

## pmacct → ClickHouse Mapping

* **Aggregates written:**

  * `flows_by_asn`: `src_as, dst_as, packets, bytes`
  * `flows_by_prefix`: `src_net, dst_net, packets, bytes` (as strings)
  * `flows_by_ip`: `src_host, dst_host, packets, bytes` (as strings)

* **Commit interval**: 5s (edit `clickhouse_commit_interval` in pmacct conf)

* **Retention**: TTLs defined in `clickhouse/init/01_schema.sql` (tune 14–90d)

---

## Useful Grafana / ClickHouse Queries

**Prefix filter**

```sql
SELECT toStartOfMinute(ts) AS t, sum(bytes)*8/1e6 AS Mbps
FROM pmacct.flows_by_prefix
WHERE ts > now() - INTERVAL 30 MINUTE
  AND (src_prefix = '103.178.66.0/24' OR dst_prefix = '103.178.66.0/24')
GROUP BY t ORDER BY t;
```

**ASN filter**

```sql
SELECT toStartOfMinute(ts) AS t, sum(bytes)*8/1e6 AS Mbps
FROM pmacct.flows_by_asn
WHERE ts > now() - INTERVAL 30 MINUTE
  AND (src_as = 15169 OR dst_as = 15169)
GROUP BY t ORDER BY t;
```

**Single IP**

```sql
SELECT toStartOfMinute(ts) AS t, sum(bytes)*8/1e6 AS Mbps
FROM pmacct.flows_by_ip
WHERE ts > now() - INTERVAL 30 MINUTE
  AND (src_ip = '103.217.110.2' OR dst_ip = '103.217.110.2')
GROUP BY t ORDER BY t;
```

**Top-N ASNs (last 5m)**

```sql
SELECT src_as, dst_as, round(sum(bytes)*8/1e6,2) AS Mbps, sum(packets) AS pps
FROM pmacct.flows_by_asn
WHERE ts > now() - INTERVAL 5 MINUTE
GROUP BY src_as, dst_as
ORDER BY Mbps DESC
LIMIT 20;
```

---

## Security & Ops Tips

* Open only required ports on the host: **2055/udp**, **6343/udp**, (optional BGP) **179/tcp**, **3000/tcp** (Grafana), **8123/tcp** (CH http).
* Change Grafana admin password immediately.
* Keep sampling sane (1:1000 or 1:2000 to start).
* Use RR as BGP peer for richer global prefixes if feasible.

---

## Troubleshooting (Best-Practice Checklist)

### A) Containers & Logs

```bash
docker compose ps
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
# If any Restarting/Exited:
docker logs -f nfacctd
docker logs -f sfacctd
docker logs -f ch-flow
docker logs -f grafana-flow
```

> **Note:** `nfacctd`/`sfacctd` in compose run with `-D` (daemonize).
> For **live debug** temporarily run foreground inside the container:

```bash
# NetFlow/IPFIX collector debug
docker exec -it nfacctd sh -lc "nfacctd -f /etc/pmacct/nfacctd.conf -d"

# sFlow collector debug
docker exec -it sfacctd sh -lc "sfacctd -f /etc/pmacct/sfacctd.conf -d"
```

### B) Are flow packets arriving? (Host)

```bash
sudo tcpdump -ni any udp port 2055 -c 20     # NetFlow/IPFIX
sudo tcpdump -ni any udp port 6343 -c 20     # sFlow
```

No packets? Check router exporter (VRF, ACL, NAT), host FW rules, IP reachability.

### C) Is pmacct listening on the right ports?

(We use `network_mode: host`, so listeners are on the host namespace.)

```bash
sudo ss -lunp | egrep '2055|6343'        # collectors
sudo ss -ltnp | egrep ':179|:8123|:3000' # BGP/CH/Grafana
```

### D) BGP enrichment up?

* On host: is **179/tcp** reachable from the router? Are sessions established?
* Quick sniff:

```bash
sudo tcpdump -ni any tcp port 179 -c 10
```

* Inside `nfacctd`/`sfacctd`, check the BGP daemon is enabled in conf and the peer is in `bgp_neighbors.conf`.
  If needed, test with a direct TCP connect from router to host: `telnet <host-ip> 179`.

### E) ClickHouse receiving data?

**List tables**

```bash
docker exec -it ch-flow clickhouse-client --query "SHOW TABLES FROM pmacct"
```

**Recent rows**

```bash
docker exec -it ch-flow clickhouse-client --query "
SELECT count() FROM pmacct.flows_by_ip WHERE ts > now() - INTERVAL 5 MINUTE;"
```

If **0**, likely no flows or plugin misconfig. Re-check pmacct conf: `plugins`, `aggregate[...]`, `clickhouse_table[...]`, `clickhouse_schema[...]`, and `clickhouse_server[*]`.

**HTTP probe**

```bash
curl -s "http://localhost:8123/?query=SELECT%201"
```

### F) Grafana provisioning OK?

```bash
docker exec -it grafana-flow ls /etc/grafana/provisioning/datasources
docker exec -it grafana-flow ls /etc/grafana/provisioning/dashboards
docker logs grafana-flow | egrep -i "provision|datasource|dashboard"
```

If dashboards don’t show, import `grafana/provisioning/dashboards/flows.json` manually via UI.

### G) Sampling ratio mismatch → wrong Mbps/PPS

* Router sampling (e.g., 1:1000) **must** match pmacct `sampling_rate` in `nfacctd.conf` / `sfacctd.conf`.
* Wrong sampling inflates/deflates all rates.

### H) Interface attach / active timeout

* For NetFlow/IPFIX, ensure **monitor map** is attached ingress/egress where needed and **active timeout \~60s** (faster time-series).

### I) Prefix/ASN not resolving correctly

* Verify BGP session is established and the router is actually sending the RIB/adj-RIB-in you expect.
* Consider peering pmacct to an RR carrying full local routes for better coverage.

### J) Performance tuning

* Increase `clickhouse_commit_interval` (e.g., 5–10s) to reduce write overhead.
* Ensure SSD storage for ClickHouse.
* Keep Grafana refresh 5–10s instead of 1–2s under heavy load.
* Adjust TTLs in the CH schema per your retention needs (14–90 days+).

### K) Full reset (last resort)

```bash
docker compose down -v
docker compose pull
docker compose up -d
```

> ⚠️ This removes volumes & history. Backup first if needed.

---

## Upgrades

```bash
docker compose pull
docker compose up -d
```

---

## Contributing

PRs & issues welcome! Helpful additions:

* Router-specific exporter snippets (XR/XE, Juniper, MikroTik, Huawei)
* Extra Grafana panels / alerts
* Ansible roles for provisioning

---

## License

MIT (or your preferred OSS license)


