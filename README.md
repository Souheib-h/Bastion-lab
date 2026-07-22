# Bastion-lab

> A hardened SSH jump host on its own isolated network, routed through OPNsense/FortiGate, providing controlled access to the K3s-lab-monitoring and K3s-lab networks.

**Related:** [K3s-lab-monitoring](https://github.com/Souheib-h/K3s-lab-monitoring) · [K3s-lab](https://github.com/Souheib-h/K3s-lab)

---

## What this is

A single-purpose bastion VM sitting on its own network segment. It is not part of k3s-net or monitoring-net; it reaches both only through the router. Every SSH session to a lab host goes through it first.

- SSH key only, no password auth, no direct root login
- fail2ban on the SSH port
- ProxyJump-based access from the operator's machine

---

## Architecture

```
  bastion-net (10.30.0.0/24)                    OPNsense / FortiGate
  Bastion-srv    10.30.0.10  ─────────────────▶  (router, no direct L2
                                                   access to either net)
                                                          │
                                        ┌─────────────────┴─────────────────┐
                                        ▼                                   ▼
                              k3s-net (10.10.0.0/24)          monitoring-net (10.20.0.0/24)
                              K3s-srv-1/2/3, agents            Zabbix, Wazuh, Prometheus,
                              K3s-db, Load-srvs                Grafana, Ansible-ctl
```

The bastion has no interface on k3s-net or monitoring-net. Every session is a hop through the router, which is the only device with visibility into all three segments.

---

## Status

| Item | Status |
| ---- | ------ |
| Network + VM | Not started |
| SSH hardening | Not started |
| fail2ban | Not started |
| ProxyJump access | Not started |
| Logging to Loki | Planned (depends on Loki setup) |

---

## Reference

| Doc | Description |
| --- | ----------- |
| [Provisioning](docs/provisioning.md) | VM creation, bastion-net setup, routing through OPNsense/FortiGate |
| [SSH setup](docs/ssh-setup.md) | Server-side hardening and client-side ProxyJump access |
| [Logging](docs/logging.md) | Connection logging, planned integration with Loki |
| [Architecture Decisions](DECISIONS.md) | Why a separate network, why Alpine, why ProxyJump over a VPN |
| [Troubleshooting](docs/troubleshooting.md) | Issues hit while building this |

---

## Environment

- **Host**: ThinkPad E14 Gen 5, KVM/libvirt on Arch Linux
- **VM OS**: Alpine Linux
