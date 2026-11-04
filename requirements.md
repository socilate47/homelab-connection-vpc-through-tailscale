# Requirements

## Functional
- Lab machines can reach AWS VPC machines by private IP, and the reverse.
- The link is encrypted and uses Tailscale subnet routing.
- Only chosen lab VLANs are reachable from AWS (least exposure).
- No service in AWS is open to the internet beyond what bootstrap needs, and bootstrap access is removed afterwards.
- Everything is documented well enough to rebuild from scratch.

## Constraints
- **Cost:** avoid NAT Gateway; small instance; stop it when idle; set a budget alert before launching anything.
- **Do not** use AWS Site-to-Site VPN / Virtual Private Gateway unless the owner decides to change design.
- Address ranges must not overlap: VPC `10.50.0.0/16`, lab `192.168.x.0/24`, Tailscale `100.64.0.0/10`.
- Home public IP is not stable (carrier-grade NAT) — do not depend on a single-IP allow-list.
- Region `us-east-1`; instance type must be supported in the subnet's AZ.

## Dependencies
- AWS account with permission to create VPC, EC2, security groups.
- A Tailscale account/tailnet already containing the lab (OPNsense node).
- OPNsense with the Tailscale plugin; Proxmox host running the lab VMs.
- Local tools: `ssh`, `ping`, `nc`, `tcpdump` (on the router and OPNsense), AWS console access.
- A key pair for the instance (never commit it to the repository).
