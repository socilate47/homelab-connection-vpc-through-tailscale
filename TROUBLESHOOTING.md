# Troubleshooting log (from the real build)

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Instance types greyed out ("not supported in us-east-1e") | Subnet was created in `us-east-1e`, a zone without those (Arm) types | Recreate the subnet in `us-east-1a`; the AZ is immutable. It was **not** a free-plan limit |
| 2 | SSH hangs (timeout, not refused) with a correct `/32` rule | ISP carrier-grade NAT: AWS saw several source IPs (`.99/.102/.105/.108` of one /24) while `ifconfig.me` showed one | Allow the ISP's `/24` temporarily; then SSH over Tailscale and delete the rule. Timeout = packets dropped; refused = nothing listening |
| 3 | Console error "CIDR block, a security group ID or a prefix list has to be specified" | Source box empty — typed text not committed | Choose *Custom*, type the CIDR, press Enter until it shows as a tag |
| 4 | Browser EC2 Instance Connect failed with the AWS service range rule | Not resolved (the range used matched AWS's published EC2 Instance Connect range for the region) | Used the ISP-range rule instead |
| 5 | `curl` from the instance to tailscale.com timed out | Outbound path blocked (security group outbound rule) | Restore outbound *All traffic → 0.0.0.0/0* |
| 6 | Pinging the router's private IP works with no ICMP rule | Tailscale traffic enters via `tailscale0`, bypassing the security group | Expected; this does not prove forwarding |
| 7 | Plan blamed on free plan / credits | Wrong diagnosis (see #1) | Read the tooltip on the greyed-out type |
| 8 | OPNsense "advertised routes" had `10.50.0.0/16` | Mixed up advertise and accept | Delete it from advertised routes; tick **Accept Subnet Routes** instead |
| 9 | Lab → AWS stops when *Accept Subnet Routes* is unticked on OPNsense, works again when ticked | Without it OPNsense has no `10.50.0.0/16` route, so the packet goes out the WAN and is lost | Keep it on. Lab → AWS needs **both** Accept Subnet Routes **and** the Source NAT rule. AWS → lab does not need it |

## WORKAROUND FOUND: lab VM → AWS gets no replies
**Fix that worked:** OPNsense → Firewall → NAT → **Source NAT** (menu name in 26.x; "Outbound" in older versions) → add rule:
interface `tailscale`, IPv4, protocol any, source `any`, destination `any`, translate source = interface address (leave empty), then Apply.
Result: `ping 10.50.1.113` from the lab VM → 5/5 replies, ttl 63. The form rejected `192.168.0.0/16` / `10.50.0.0/16` typed in the address box;
use `any` or aliases (Firewall → Aliases, type *Network(s)*).
**Why it works (from measurements):** Tailscale delivered traffic when at least one end was a plain Tailscale address
(`100.x.x.1 → 192.168.70.153` ok; `10.50.1.113 → 100.x.x.2` ok) but lost it when *both* ends were subnet addresses
(`10.50.1.113 → 192.168.70.153`). NAT makes the lab source a Tailscale address, so replies take the working pattern.
**Root cause not proven; most likely (docs + community, unconfirmed):** Tailscale's site-to-site mode (kb/1214) is specified for **Linux** routers only
("Both subnet routers must use a Linux-based operating system"), and needs `--snat-subnet-routes=false`, which FreeBSD/OPNsense lacks (GitHub issue #7073).
An OPNsense forum thread reports the same pattern and the same NAT workaround. The "source-address check on OPNsense" idea is my inference only.
Alternative if NAT is unwanted: a small Linux Tailscale router VM behind OPNsense. Revert the access policy to the default allow-all (explicit CIDRs did not help).
Ruled out: AWS security group, network ACL, Tailscale access policy (default allow-all, also tried naming the CIDRs), OPNsense firewall rules/log, VLAN/bridge, VM firewall, RouteAll (true).
Trade-off: AWS sees all lab-to-tailnet traffic as one source (OPNsense's Tailscale address).
Untested: lab → other VPC machines (needs VPC route `192.168.x.0/24 → router`, SG rules, and masquerade or src-NAT on the AWS router for VPC-to-lab traffic).
Evidence collected:
- Monitor VM (`192.168.70.153`, VLAN 70, gateway OPNsense) pings the router private IP: 0 replies (20/20 lost).
- OPNsense firewall log: `pass out` on `tailscale0` for those pings; **no block** lines.
- AWS router `tcpdump -ni tailscale0 icmp`: every request arrives and a reply is sent immediately.
- VM `tcpdump -ni ens18`: requests leave, **no reply ever arrives**.
- OPNsense routing table has `10.50.0.0/16 → tailscale0` (after Accept Subnet Routes).
- Works: AWS router → VM; laptop → VM; laptop → router private IP.
- Side observation: OPNsense logs its own Tailscale UDP 41641 to the router's private IP leaving via `tailscale0`
  (route-loop risk from accepting the subnet that contains the peer's endpoint).

Steps tried (in order; #4 worked):
1. `tcpdump -ni tailscale0 icmp` and `tcpdump -ni vlan07 icmp` on OPNsense while the VM pings — do replies arrive, and are they forwarded?
2. Add a logged pass rule for `10.50.0.0/16` → the VLAN alias on the Tailscale interface (or a floating rule).
3. Check Tailscale access rules (ACLs) for source `10.50.0.0/16` → `192.168.70.0/24`.
4. Outbound/source NAT on the Tailscale interface — **worked** (see top of this section).

## Other OPNsense notes from this build
- Web GUI on the Tailscale address only worked after `configctl webgui restart` following a reboot (boot-order race). Fix: executable script
  `/usr/local/etc/rc.syshook.d/start/99-webgui-after-tailscale` that waits for `tailscale0` to get a `100.` address, then restarts the GUI.
  The root shell is csh: heredocs with a quoted delimiter hang; run `sh` first (or the pasted lines show a `?` prompt).
- Packet captures taken on `tailscale0` see packets *before* the firewall, so they show whether Tailscale delivered a packet at all.
- "Advertised routes" = networks behind this node; "Accept Subnet Routes" = learn other nodes' routes. Never advertise the AWS CIDR from OPNsense.
- An AWS security group does not see Tailscale traffic to the router itself (it enters via `tailscale0`, not `ens5`).
