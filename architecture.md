# Architecture

> Diagrams and a step-by-step explanation of every choice are in [docs/](docs/) — start at the [README](README.md).

```
 Lab VLAN VM ──▶ OPNsense (Tailscale node) ══ encrypted tunnel ══ AWS router (Tailscale node) ──▶ VPC 10.50.0.0/16
 192.168.n.0/24    advertises 192.168.n.0/24                       advertises 10.50.0.0/16
                   accepts     10.50.0.0/16                        accepts     192.168.n.0/24
```

## Data flow
1. A lab VM sends traffic for `10.50.x.x` to its gateway (OPNsense on its VLAN).
2. OPNsense has a route `10.50.0.0/16 → tailscale0` (learned because *Accept Subnet Routes* is on).
3. Tailscale carries it (direct if possible, otherwise via a DERP relay) to the AWS router.
4. The AWS router forwards it into the VPC (IP forwarding on, source/destination check off).
5. Replies take the reverse path; the router learned `192.168.n.0/24 → tailscale0` (`--accept-routes`).

## Key decisions
| Decision | Why |
|---|---|
| Tailscale subnet routing, not Site-to-Site VPN | About $3–7/month versus ~$36; no home ports open; survives changing home IPs |
| Public subnet + internet gateway, no NAT Gateway | NAT Gateway is the usual accidental cost; the router only needs outbound internet |
| Advertise only on the side that owns the network | AWS router advertises `10.50.0.0/16`; OPNsense advertises lab VLANs; each *accepts* the other's |
| Never advertise `10.50.0.0/16` from OPNsense | It would claim the path to AWS and cause loops/black-holes |
| Keep the VPC CIDR clear of lab ranges | Tailscale routes by prefix; overlaps break routing |
| Default NACL left open | Security group is the filter; stateless NACL rules are easy to get wrong |
| Close SSH to the world after bootstrap | Reach the router over Tailscale instead |
| Source NAT on OPNsense's Tailscale interface | Lab→AWS replies from `10.50.x.x` to `192.168.x.x` were lost; making the lab source a Tailscale address avoids that. Cost: AWS sees one source for all lab traffic |
| Address plan | VPC `10.50.0.0/16`; lab `192.168.<vlan>.0/24`; avoid `172.17/16` (Docker), `172.31/16` (AWS default VPC) |

## Gotchas that shaped the design
- Subnet **Availability Zone is fixed at creation**; some zones lack instance types (us-east-1e had no Arm types).
- A public subnet is defined by its **route table** (`0.0.0.0/0 → igw`), not by its auto-assign setting.
- The Linux Tailscale client ignores subnet routes unless started with `--accept-routes`.
- Home ISP NAT pool: a `/32` security-group rule for your own IP will not match reliably.
