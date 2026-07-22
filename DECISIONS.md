# Architecture Decision Records

This document captures the key architectural decisions made throughout the `K3s-lab-monitoring` project — not *what* was built, but *why* it was built that way. Each ADR answers: what was the context, what was decided, what alternatives were considered, and what are the consequences.

---

## ADR-001 — Two isolated networks instead of one flat network

**Status:** Decided

### Context

The lab runs a K3s cluster (3 control planes, 3 workers, 1 LB, 1 DB) alongside a dedicated monitoring stack (Wazuh, Prometheus, Grafana, Zabbix). Both sets of VMs could have shared a single libvirt network.

### Decision

Two separate networks:
- `k3s-net` (10.10.0.0/24) — cluster traffic
- `monitoring-net` (10.20.0.0/24) — monitoring stack

Routed through OPNsense.

### Why

- **Security boundary** — monitoring VMs can observe the cluster without being part of it. If a monitoring tool is compromised, it doesn't have L2 access to the cluster nodes.
- **Realistic architecture** — production monitoring infrastructure is always network-segregated from what it monitors.
- **Traffic isolation** — Prometheus scrape traffic, Wazuh agent communication, and Zabbix polling don't pollute cluster inter-node traffic.

### Alternatives rejected

- **Single flat network** — simpler, but no security boundary and not representative of real environments.
- **VLANs on a single bridge** — overkill for a KVM lab, harder to manage without physical switching.

---

## ADR-002 — OPNsense as the inter-network router

**Status:** Decided

### Context

With two isolated networks, something needs to route between them. Options ranged from a simple kernel IP forwarding setup to a full firewall VM.

### Decision

OPNsense 26.1 deployed as a dedicated VM with:
- WAN interface on `k3s-net` (10.10.0.254)
- LAN interface on `monitoring-net` (10.20.0.254)

### Why

- **Visibility** — OPNsense provides packet capture, firewall logs, and traffic graphs. Proved invaluable during debugging (confirmed inter-network routing with live packet captures showing TTL decrements and MAC translation at the L3 boundary).
- **Realistic SOC context** — Wazuh and Grafana should ideally receive OPNsense syslog as a data source. Having a real firewall in the path makes this possible.
- **Learning value** — OPNsense config (NAT outbound disable, block private networks off on WAN) is directly transferable to real deployments.

### Alternatives rejected

- **iptables forwarding on the host** — works, but no visibility, no logs, not a realistic setup.
- **libvirt routed mode** — attempted, failed due to masquerade chain (`LIBVIRT_PRT`) intercepting traffic before custom routing rules. Required adding `RETURN` rules to exempt inter-network traffic — ultimately replaced by OPNsense for cleaner separation.

### Notable fix

libvirt's nftables/iptables masquerade chain (`LIBVIRT_PRT`) rewrites source IPs before custom routing can work. Fix: switch libvirt firewall backend to `iptables`, add `RETURN` rules to exempt inter-network traffic, disable outbound NAT on OPNsense.

---

## ADR-003 — Wazuh all-in-one for SOC

**Status:** Decided

### Context

Wazuh has three components: indexer (OpenSearch-based), manager, and dashboard. They can be deployed separately (distributed) or co-located (all-in-one).

### Decision

All-in-one deployment on a single `Wazuh-srv` VM (8GB RAM, 4 vCPU).

### Why

- **Scale** — the lab monitors fewer than 20 endpoints, well within the all-in-one supported range (up to 100 agents).
- **Simplicity** — distributed deployment adds cert complexity, inter-component networking, and failure domains that aren't justified at this scale.
- **Resource efficiency** — a single 8GB VM handles all three components comfortably.

### Alternatives rejected

- **Distributed deployment** — appropriate for production at scale, overkill here.
- **Wazuh Cloud** — defeats the purpose of a self-hosted SOC lab.

---

## ADR-004 — Prometheus + node_exporter for NOC metrics

**Status:** Decided

### Context

The NOC stack needs to collect and visualize host-level metrics (CPU, RAM, disk, network) from K3s nodes and monitoring VMs.

### Decision

- Prometheus scrapes `node_exporter` instances on each monitored VM
- Grafana uses Prometheus as its NOC datasource
- Dashboard: **Node Exporter Full** (Grafana ID `1860`)

### Why

- **Standard** — node_exporter + Prometheus is the de facto standard for Linux host metrics. 1000+ community dashboards available.
- **Native Grafana support** — no plugin required, no compatibility issues.
- **Pull model** — Prometheus scrapes targets on its schedule; no agent-to-server push config needed beyond exposing port 9100.
- **Separation of concerns** — Prometheus handles metrics; Wazuh handles security events. Each tool does one thing well.

### Alternatives rejected

- **Zabbix agent for metrics** — Zabbix collects the same data, but requires the Grafana-Zabbix plugin to visualize it in Grafana (see ADR-005). Prometheus is simpler and more Grafana-native for this use case.
- **Wazuh for metrics** — Wazuh is a SIEM, not a metrics platform. It doesn't expose time-series data in a Prometheus-compatible format.

---

## ADR-005 — Zabbix retained in the stack for SOC/infrastructure monitoring

**Status:** Decided (revised — see history below)

### Context

Zabbix was originally planned as part of the monitoring stack. During Phase 3, the Grafana-Zabbix plugin (`alexanderzobnin-zabbix-app`) failed to connect despite correct credentials, reachable API, and valid network path.

### Decision history

1. **Initially:** Zabbix included in the stack for infrastructure monitoring alongside Prometheus.
2. **Phase 3 (mid-debug):** Zabbix temporarily removed from the active stack after exhausting standard troubleshooting (two plugin versions, two Grafana versions, API token vs user/password). Documented in `docs/troubleshooting/zabbix-apache-authorization-header.md`.
3. **Phase 3 (resolution):** Root cause identified and fixed. Zabbix reinstated.

### Root cause of the Phase 3 issue

Apache was stripping the `Authorization` header before it reached PHP-FPM, causing every authenticated Zabbix API call to return `"Not authorized"`. Fix: `sudo a2enconf php8.5-fpm && sudo systemctl reload apache2`.

See full post-mortem: [`docs/troubleshooting/zabbix-apache-authorization-header.md`](docs/troubleshooting/zabbix-apache-authorization-header.md)

### Final decision

Zabbix is retained for:
- Infrastructure monitoring of K3s nodes and monitoring VMs (host groups, triggers, alerting)
- Complementing Wazuh for infrastructure-level visibility (process monitoring, service checks, SNMP if needed later)
- Learning value — Zabbix is widely used in enterprise environments

### Role split

| Tool | Role |
|---|---|
| **Prometheus + node_exporter** | Time-series metrics, NOC dashboards |
| **Zabbix** | Infrastructure monitoring, triggers, alerting |
| **Wazuh** | SOC — FIM, brute force detection, CVE scan, SIEM |
| **Grafana** | Unified visualization (Prometheus + Zabbix datasources) |

---

## ADR-006 — Ansible for agent/exporter deployment (Phase 5)

**Status:** Decided

### Context

Both Prometheus (node_exporter) and Wazuh (agents) need to be deployed on all 10 monitored VMs. Manual installation on each VM is feasible but not repeatable or scalable.

### Decision

All agent/exporter deployment is deferred to Phase 5 and handled exclusively via Ansible:
- `node_exporter` on all K3s nodes + monitoring VMs
- Wazuh agents on all K3s nodes + monitoring VMs
- Zabbix agents on all K3s nodes + monitoring VMs

Phase 2 installs only the central servers (Prometheus, Wazuh manager, Zabbix server, Grafana). No agents installed manually.

### Why

- **Repeatability** — if a VM is rebuilt, one playbook run restores the full agent stack.
- **Consistency** — same config on all nodes, no manual drift.
- **Portfolio value** — Ansible automation across a 10-VM lab is a meaningful demonstration of IaC skills.

### Alternatives rejected

- **Manual install on each VM** — done for the central servers (educational value), not appropriate for repeated per-node tasks.
- **K3s DaemonSet for node_exporter** — valid approach, but adds K8s complexity to what should be a simple host-level metric collection. Deferred as a potential enhancement post-Phase 6.
