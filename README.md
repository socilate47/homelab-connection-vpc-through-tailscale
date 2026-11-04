# Homelab connection: AWS VPC through Tailscale

A hands-on **learning guide**: build an AWS VPC from scratch, then connect it privately to a Proxmox + OPNsense homelab
using **Tailscale subnet routing** (no Site-to-Site VPN, no open ports at home). Every step explains *why*, shows the
calculations behind the address plan, and includes the real problems that came up and how they were diagnosed.

(Repository: `homelab-connection-vpc-through-tailscale`.)

```mermaid
flowchart LR
  subgraph LAB["Homelab (Proxmox)"]
    VM["Lab VMs<br/>VLANs 192.168.x.0/24"]
    OPN["OPNsense<br/>Tailscale node"]
    VM --> OPN
  end
  subgraph AWS["AWS VPC 10.50.0.0/16"]
    RTR["EC2 subnet router<br/>Tailscale node"]
    SUB["Subnet 10.50.1.0/24"]
    RTR --> SUB
  end
  OPN <-->|"encrypted Tailscale tunnel"| RTR
```
```
 Lab VM ─▶ OPNsense ══ encrypted tunnel ══ EC2 router ─▶ VPC 10.50.0.0/16
 192.168.x.0/24   (Tailscale)              (Tailscale)
```

## Learning path
| # | Chapter | You will learn |
|---|---|---|
| 1 | [IP addressing and CIDR](docs/01-ip-addressing-and-cidr.md) | binary, masks, `/16` vs `/24`, overlap checks, address planning, with exercises |
| 2 | [Build the VPC](docs/02-build-the-vpc.md) | VPC, subnet, gateway, route table, security group, EC2 — and why each exists |
| 3 | [Tailscale and the AWS router](docs/03-tailscale-subnet-router.md) | advertise vs accept, IP forwarding, approving routes |
| 4 | [The lab side (OPNsense)](docs/04-lab-side-opnsense.md) | accepting routes, advertising VLANs, the source-NAT fix |
| 5 | [Packet flows and testing](docs/05-packet-flows-and-testing.md) | proving both directions, `tcpdump` method, diagrams of what failed and why |
| 6 | [Cost and clean-up](docs/06-cost-and-cleanup.md) | what costs money, stop vs terminate, checklist |
| — | [Glossary](docs/07-glossary.md) | |

Reference docs: [`system.md`](system.md) · [`architecture.md`](architecture.md) · [`gude.md`](gude.md) (quick step list) · [`requirements.md`](requirements.md)

## Prerequisites
- An AWS account (set a **budget alert** first), a Tailscale account, a Proxmox host with OPNsense (Tailscale plugin).
- Comfort with a terminal and SSH. No prior VPC knowledge needed.

## Status (honest)
| Path | State |
|---|---|
| Laptop ↔ AWS router, laptop ↔ lab VMs | Verified |
| AWS router → lab VM | Verified |
| Lab VM → AWS router | Verified **with an OPNsense source-NAT workaround**; the root cause is not proven ([chapter 5](docs/05-packet-flows-and-testing.md#56-why-does-it-fail-honest-status)) |

Scope: this guide covers the tunnel and routing between the lab and the AWS router.

The instance used to build this was terminated afterwards to stop costs; the guide rebuilds it.

> All addresses are examples or private ranges. No keys, tokens or account identifiers belong in this repository.
