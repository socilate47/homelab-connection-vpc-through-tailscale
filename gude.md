# Guide: build the AWS ↔ homelab link (Tailscale subnet routing)

> Quick step list. For the full, explained learning guide (with diagrams and the maths) start at the [README](README.md).

Build order matters. After each step, verify before moving on. `<…>` values are yours.

## 0. Before you start
- Set a **budget alert** (Billing → Budgets → Monthly cost budget, e.g. $5).
- Decide ranges that do not overlap: VPC `10.50.0.0/16`; lab `192.168.<vlan>.0/24`.
- Know which lab VLANs AWS should reach (start narrow).
- Region: `us-east-1`. Check the region selector (top right) before every step.

## 1. VPC
VPC console → *Create VPC* → **VPC only** (not "VPC and more": you want to build each piece).
Name `<name>`, IPv4 `10.50.0.0/16`, no IPv6, tenancy Default. Then *Edit VPC settings* → enable DNS hostnames.
**Check:** state Available; a main route table, default NACL and default security group exist.

## 2. Subnet
*Create subnet* → your VPC, name, **Availability Zone chosen explicitly (e.g. us-east-1a, never "no preference")**,
CIDR `10.50.1.0/24`. Then *Edit subnet settings* → enable **auto-assign public IPv4**.
**Check:** Available, 251 free addresses. The AZ cannot be changed later.

## 3. Internet gateway
*Create internet gateway* → then *Actions → Attach to VPC*. **Check:** state Attached.

## 4. Route table
*Create route table* in the VPC. Add route `0.0.0.0/0 → internet gateway`. *Subnet associations* → tick your subnet.
**Check:** routes `10.50.0.0/16 → local` and `0.0.0.0/0 → igw-…`, both Active; the subnet is explicitly associated.
(A new subnet uses the *main* route table, which has no internet route, until you associate it.)

## 5. Security group
Create in the same VPC. Inbound: SSH 22 from **your ISP range, not just one IP** — temporary.
Leave the default outbound rule (all traffic). Tailscale needs **no inbound rule**.

## 6. EC2 subnet router
Ubuntu Server 24.04 LTS, small type supported in your AZ (Arm `t4g.*` needs the *Arm* AMI), your subnet,
auto-assign public IP, your security group, an ED25519 key pair (`chmod 600` the `.pem`).
Then **Actions → Networking → Change source/destination check → Stop**. (Without this, forwarding silently fails.)
**Check:** Running, 2/2 status checks.

## 7. Install and join Tailscale (on the instance)
```sh
ssh -i <key>.pem ubuntu@<public-ip>

# forward packets
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# install (download, read, then run — do not blind pipe to sh)
curl -fsSL https://tailscale.com/install.sh -o tailscale-install.sh
less tailscale-install.sh
sudo sh tailscale-install.sh

sudo tailscale up --advertise-routes=10.50.0.0/16 --accept-routes --hostname=aws-router
```
Open the printed login link in a browser and sign in to your tailnet. Then in the Tailscale admin console:
*Machines → aws-router → ⋯ → Edit route settings* → approve `10.50.0.0/16`; and *Disable key expiry*.
**Check (from a machine that accepts routes):** `ping <router-private-ip>` replies.

## 8. Lab side (OPNsense)
- VPN → Tailscale → Settings: tick **Accept Subnet Routes**, Apply.
- VPN → Tailscale → Advertised Routes: list **only lab VLANs**. Do **not** add `10.50.0.0/16` here.
- System → Routes → Status: confirm `10.50.0.0/16 → tailscale0`.
- Firewall → NAT → **Source NAT**: add a rule on interface `tailscale` (source `any`, destination `any`, translate to interface address).
  Without it, lab → AWS replies were lost. Use aliases instead of `any` to narrow it later.
- Firewall: allow traffic from `10.50.0.0/16` to the chosen VLANs (use an alias for the VLAN list), narrow, logged.

## 9. Verify, both directions
| Test | Command | Where |
|---|---|---|
| Tunnel up | `tailscale ping aws-router` | any node |
| AWS → lab VLAN host | `ping <lab-vm>` | AWS router |
| Lab → AWS | `ping -c 10 -W 5 <router-private-ip>` | lab VM |
| Where packets die | `sudo tcpdump -ni tailscale0 icmp` | router and OPNsense |
Note: pinging the router's own private IP does not prove forwarding; use a second VPC machine for that.

## 10. Lock down
Reach the router over Tailscale (`ssh ubuntu@<tailscale-ip>`), then **delete every SSH rule** from the security group.
Tighten Tailscale access rules (ACLs) and OPNsense rules to the VLANs and ports actually needed.
