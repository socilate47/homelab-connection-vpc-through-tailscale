# System

## Purpose
Give a Proxmox cybersecurity homelab and a small AWS VPC a private, encrypted link so that
machines on each side can reach the other, as a hybrid-cloud learning environment (VPC networking,
routing, security groups, Tailscale subnet routing).

## Components
| Component | Role |
|---|---|
| AWS VPC `10.50.0.0/16` | Cloud network. One public subnet `10.50.1.0/24` (us-east-1a) |
| Internet gateway + route table | Gives the public subnet internet access |
| Security group `myhomelab` | Instance firewall (SSH only, tightened; outbound open) |
| EC2 instance (Ubuntu 24.04, Arm) | **Tailscale subnet router**: advertises `10.50.0.0/16`, accepts lab routes |
| Tailscale tailnet | Encrypted overlay that joins both sides; no ports opened at home |
| Proxmox host | Runs the lab VMs (including OPNsense) |
| OPNsense (VM) | Lab router/firewall; Tailscale node that advertises the lab VLANs |
| Lab VLANs (`192.168.<n>.0/24`) | ai, blue, cloud, deception, guests, targets, manager (+ a default LAN) |

## Operating context
- Region `us-east-1`. Instance sized small (it only forwards packets); stop it when unused.
- Home internet is behind carrier-grade NAT: the public source address changes per connection.
- Tailscale access rules are the default (allow all) at the time of writing; tightening is planned.
- Related: no NAT Gateway is used (cost), and no Site-to-Site VPN.
