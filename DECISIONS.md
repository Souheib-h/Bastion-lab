# Architecture Decision Records

This document captures the key architectural decisions made throughout the `Bastion-host` project, not _what_ was built, but _why_ it was built that way.

---

## ADR-001: Bastion on its own network, not dual-homed

**Status:** Decided

### Context

The bastion needs to reach both k3s-net (10.10.0.0/24) and monitoring-net (10.20.0.0/24). Two designs were possible: give the bastion an interface on each network directly, or place it on its own segment routed through the existing router.

### Decision

A third isolated network, `bastion-net` (10.30.0.0/24), routed to both existing networks through OPNsense/FortiGate.

### Why

- A dual-homed bastion sits directly on the networks it is meant to gate; a compromise gives an attacker L2 presence on both. A routed bastion never touches either network at L2.
- Matches the segmentation pattern already used between k3s-net and monitoring-net (see K3s-lab-monitoring ADR-001).
- Centralizes routing and firewsteraring decisions in one place (the router), not spread across every host that needs cross-network access.

### Alternatives rejected

- **Dual-homed bastion** — simpler to route, but weaker isolation.
- **VPN instead of a bastion** — heavier to set up and maintain for a single-operator lab; ProxyJump over SSH covers the same need with less overhead.

---

## ADR-002: Alpine Linux for the bastion VM

**Status:** Decided

### Context

Every other lab VM runs Ubuntu except the Ansible control node, which runs Alpine (see K3s-lab-monitoring ADR-011).

### Decision

Alpine Linux for the bastion.

### Why

- Minimal attack surface: a bastion should run as little software as possible. Alpine's base install is a fraction of Ubuntu's.
- Already proven in this lab (Ansible control node), so the OpenRC quirks are already known rather than a new unknown.

### Alternatives rejected

- **Ubuntu** — consistent with most other VMs, but a heavier base image for a host whose only job is to relay SSH.

---