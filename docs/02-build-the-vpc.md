# 2. Build the AWS VPC, one piece at a time

We build each piece by hand (not the "VPC and more" wizard) so you see what every part does.
Region used: `us-east-1`. **Always check the region selector (top right) before each step.**

```mermaid
flowchart TB
  NET(("Internet")) --- IGW["Internet gateway"]
  IGW --- RT["Route table<br/>10.50.0.0/16 to local<br/>0.0.0.0/0 to the gateway"]
  RT --- SUBN["Public subnet 10.50.1.0/24<br/>one Availability Zone"]
  SUBN --- EC2["EC2 instance (Tailscale router)<br/>protected by a security group"]
```
```
 Internet ── Internet gateway ── Route table ── Subnet 10.50.1.0/24 ── EC2 (+ security group)
```

## Step 0 — Safety first
Create a **budget alert** (Billing → Budgets → Monthly cost budget, e.g. $5) *before* launching anything.
Avoid the classic accidental costs: a **NAT Gateway** (about $32/month) and idle running instances.

## Step 1 — The VPC
*What:* your own private network inside AWS. Nothing gets in or out until you build the paths.
*How:* VPC → Create VPC → **VPC only** → name, IPv4 `10.50.0.0/16`, no IPv6, tenancy **Default** (dedicated tenancy costs much more) → then
*Actions → Edit VPC settings → enable DNS hostnames*.
*Why `/16`:* see [chapter 1.8](01-ip-addressing-and-cidr.md#18-why-the-vpc-is-a-16-and-the-subnet-a-24).
*Check:* state **Available**; AWS auto-created a main route table, a default network ACL and a default security group.

## Step 2 — The subnet
*What:* a slice of the VPC that lives in **one Availability Zone** (one data centre). Instances are launched into subnets.
*How:* Subnets → Create subnet → your VPC → name → **Availability Zone: choose one explicitly** (e.g. `us-east-1a`) → `10.50.1.0/24`.
Then *Edit subnet settings → enable auto-assign public IPv4*.
*Check:* **Available**, 251 free addresses (256 − 5 reserved).

> **Lesson learned (real mistake):** a subnet's zone **cannot be changed**. Ours first landed in `us-east-1e`, which does not offer
> many instance types (the console greyed them out with "not supported in us-east-1e"). Fix = delete and recreate in `us-east-1a`.
> Never leave "No preference" without checking where it ended up.

## Step 3 — The internet gateway
*What:* the door between the VPC and the internet. Free.
*How:* Internet gateways → Create → then **Actions → Attach to VPC**. A new gateway is *Detached* and does nothing.
*Check:* state **Attached**.

## Step 4 — The route table (this is what makes a subnet "public")
A route table is a list of directions: *"traffic for this destination goes to this next hop."* Rules are matched **most specific first**.

| Destination | Target | Meaning |
|---|---|---|
| `10.50.0.0/16` | `local` | traffic inside the VPC stays inside (AWS adds this; cannot be removed) |
| `0.0.0.0/0` | `igw-…` | everything else goes to the internet gateway |

*How:* Route tables → Create (in your VPC) → add the `0.0.0.0/0 → internet gateway` route → **Subnet associations → tick your subnet**.
*Check:* both routes **Active**; the subnet shows under *Explicit subnet associations*.

> A subnet is *public* because its route table has a route to an internet gateway — **not** because of its name or the
> auto-assign-IP setting. A new subnet uses the **main** route table (no internet route) until you associate it.
>
> **Compare with OPNsense:** the route table is like a router's routing table (default route + connected networks).
> AWS has one table *per subnet*; OPNsense has one table for the whole box.

## Step 5 — The security group (the instance firewall)
Facts to remember:
- It filters traffic at the **instance**, and only **allows** (there are no deny rules). Anything not allowed is blocked.
- It is **stateful**: replies to allowed traffic are let back automatically.
- A new group starts with **no inbound rules** and **allow-all outbound**.

For this project: **no inbound rules are needed for Tailscale** (it only connects outward). Keep the default outbound rule
(*All traffic → 0.0.0.0/0*) — the instance must reach the internet to install packages and talk to Tailscale.
We used a temporary SSH rule only to bootstrap, then removed it (see [troubleshooting](../TROUBLESHOOTING.md)).

**Network ACL?** The subnet also has a default network ACL that allows everything. It is *stateless* (you must allow both directions),
so it is easy to break things with. We left it alone.

## Step 6 — The EC2 instance (the Tailscale router)
Ubuntu Server 24.04 LTS, a small instance type that your zone supports (an Arm `t4g.*` needs the **Arm** image; `t3.*` needs **x86**),
your subnet, **auto-assign public IP on**, your security group, an ED25519 key pair (`chmod 600` the `.pem`).

Then: **Actions → Networking → Change source/destination check → Stop**.
*Why:* AWS normally drops packets that are not to or from the instance itself. A router forwards other machines' packets, so
without this the link silently fails.
*Check:* Running with **2/2 status checks passed**.

## Step 7 — Verify before moving on
| Check | Expected |
|---|---|
| Route table | `local` and `0.0.0.0/0 → igw` both Active |
| Subnet | in the zone you chose, auto-assign public IPv4 = Yes |
| Gateway | Attached |
| Instance | Running, 2/2 checks, has a public IPv4 |

Next: [3. Tailscale on the AWS router](03-tailscale-subnet-router.md)
