
# Cisco → (NetFlow/sFlow) → pmacct → ClickHouse → Grafana

**What you get**
- Per **ASN**, **prefix**, and **single IP** bandwidth analytics.
- Uses **pmacct** collectors (NetFlow/IPFIX via `nfacctd`, or sFlow via `sfacctd`).
- **BGP peering** from pmacct to your router (or RR) for accurate ASN/prefix mapping.
- Stores flow aggregates in **ClickHouse**. Visualized in **Grafana** (dashboard included).

## Quick start

1) Edit `pmacct/bgp_neighbors.conf` with your ASNs and router IP(s).
2) If your exporter is **IPFIX/NetFlow**, keep `nfacctd` service;  
   If **sFlow**, use `sfacctd` service (or use both).
3) Adjust sampling rate & ports in `pmacct/*.conf` to match your router.
4) Start the stack:
```bash
docker compose up -d
```
5) **Grafana**: http://localhost:3000 (admin / admin on first run) → Data source and dashboard are auto-provisioned.
6) On your Cisco NCS, point exporter(s) to this host's IP:
   - NetFlow/IPFIX: UDP **2055**
   - sFlow: UDP **6343**
   - BGP to pmacct: TCP **179** (pmacct listens in the container via host network).

## Notes
- The included dashboard shows **Top ASNs**, **Top Prefixes**, **Top IPs**, and total **Mbps/PPS**.
- ClickHouse is chosen to comfortably handle high-cardinality group-bys (ASN/prefix/IP).
- If you prefer, you can disable the `sfacctd` service entirely.
- All files live under this folder; persistence is via Docker volumes.
