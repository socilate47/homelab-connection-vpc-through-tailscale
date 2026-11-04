# Troubleshooting: the problems that mattered

Small slips (console form quirks, file names) are left out. These six cost the most time, in the order you will meet them.

| # | Problem | Fix in one line |
|---|---|---|
| 1 | Instance types are greyed out | Recreate the subnet in a zone that supports them |
| 2 | SSH times out although the rule matches your address | Your ISP shares addresses: allow its block briefly, then use SSH over Tailscale |
| 3 | The instance cannot reach the internet, so the install fails | Restore the security group's default outbound rule |
| 4 | The route exists but traffic never reaches the other side | Advertise only what you own, **accept** the other side's routes, and approve them |
| 5 | OPNsense web GUI does not answer on the Tailscale address after a reboot | Restart the GUI once Tailscale is up (start-up script) |
| 6 | Lab → AWS pings get no replies | Source NAT on OPNsense's Tailscale interface |

---

## 1. Instance types greyed out ("not supported in us-east-1e")

**Symptom.** In the launch screen every instance type was greyed out. The tooltip on one read: *"This instance type is not
supported in us-east-1e."* We first suspected the free plan or the account credits. **That was wrong.**

**Cause.** The subnet had been created with the zone left on "No preference" and landed in `us-east-1e`, an older zone that
does not offer those (Arm) instance types. A subnet's zone **cannot be changed** after creation.

**Fix.** Delete the empty subnet and create it again with the zone chosen explicitly (for example `us-east-1a`). Then turn
auto-assign public IPv4 back on and associate the subnet with the public route table again.

**Lesson.** Read the tooltip before theorising. Always pick the Availability Zone yourself.

---

## 2. SSH times out although the security-group rule matches your address

**Symptom.** `ssh` hung with no error, even though the rule allowed our public address as a single address (`/32`).

**Cause.** The ISP shares public addresses between customers, so each connection can leave from a different one. Asking
AWS's own checker five times returned several different addresses from the same `/24` block:
```sh
for i in 1 2 3 4 5; do curl -4 -s https://checkip.amazonaws.com; done | sort | uniq -c
```
A single-address rule matches only some connections, and the others are dropped silently. (A **timeout** means the packets were
dropped; **"connection refused"** would mean nothing is listening.)

**Fix.** Allow the ISP's `/24` block temporarily, connect, install Tailscale, then reach the instance **over Tailscale**
and **delete the public SSH rule**. Never leave port 22 open to `0.0.0.0/0`.

**Lesson.** Measure what the far end sees. Better still, avoid the problem: once Tailscale works, no public rule is needed.

---

## 3. The instance cannot reach the internet

**Symptom.** Downloading the Tailscale installer from the instance failed with a connection timeout after about two minutes,
although inbound SSH worked.

**Cause.** Outbound traffic was blocked. A new security group allows all outbound traffic by default, so a rule had been
removed or restricted (we did not record exactly which one).

**Fix.** Make sure the outbound rule is *All traffic → anywhere (`0.0.0.0/0`)*. Tell DNS from routing with two quick tests on
the instance: `getent hosts tailscale.com` (name lookup) and a `ping` to any well-known public address (routing).

**Lesson.** Inbound working proves nothing about outbound: the instance must reach the internet to install packages and to
talk to Tailscale.

---

## 4. Advertise vs accept subnet routes

**Symptom.** Lab machines could not reach AWS. In OPNsense's *Advertised Routes* list the AWS network had been added, and
unticking or leaving *Accept Subnet Routes* off made the lab lose the path entirely.

**Cause.** Two different settings were mixed up.

| Setting | Meaning | Who sets what |
|---|---|---|
| **Advertise routes** | "This network is *behind me*; send me its traffic" | AWS router: the VPC range. OPNsense: the lab VLANs |
| **Accept routes** | "I will *use* routes that other nodes advertise" | Both routers |
| **Approval** | Admin-console step before any advertised route is used | The Tailscale admin console |

Advertising a network from the wrong node claims a path it does not have. Without *accept*, a router never learns the other
side, so its packets for that side go out the WAN and are lost.

**Fix.**
1. AWS router: `sudo tailscale up --advertise-routes=10.50.0.0/16 --accept-routes --hostname=aws-router`, then **approve** the route in the Tailscale admin console.
2. OPNsense → VPN → Tailscale → **Advertised Routes**: list **only the lab VLANs**. Delete the AWS range if it is there.
3. OPNsense → VPN → Tailscale → **Settings**: tick **Accept Subnet Routes**, Apply.
4. Check: OPNsense → System → Routes → Status shows `10.50.0.0/16 → tailscale0`.

**Lesson.** Advertise what you own; accept what others own; approve in the console. We proved the dependency by switching
*Accept Subnet Routes* off (lab → AWS stopped) and on again (it worked).

---

## 5. Web GUI only reachable on the Tailscale address after a restart

**Symptom.** After a reboot the OPNsense web interface answered on its LAN address but not on its Tailscale address, until
`configctl webgui restart` was run.

**Likely cause (not proven).** A boot-order race: the GUI starts before the Tailscale interface has its address, so it never
listens on it.

**Fix.** A start-up script that waits for the Tailscale address, then restarts the GUI once. Create
`/usr/local/etc/rc.syshook.d/start/99-webgui-after-tailscale`:
```sh
#!/bin/sh
( for i in $(seq 1 60); do
    ifconfig tailscale0 2>/dev/null | grep -q 'inet 100\.' && break
    sleep 5
  done
  /usr/local/sbin/configctl webgui restart ) &
```
then `chmod 755` it.

> **Pasting tip.** OPNsense's root shell is `csh`. If a pasted block shows `?` prompts, press Ctrl+C, run `sh`, and paste again.

**Lesson.** After a reboot we saw the GUI answer on the Tailscale address with the script installed, but we did not isolate
the script as the reason, so treat it as a sensible workaround, not a proven fix.

---

## 6. Lab → AWS gets no replies (the NAT rule)

**Symptom.** From the AWS router, pings to a lab VM worked. From the lab VM, pings to the AWS router got **100% loss**.

**What we measured (packet captures on `tailscale0`, which sees packets before the firewall).**
| Source → destination | Delivered into OPNsense? |
|---|---|
| Tailscale address → lab VM | yes |
| AWS LAN address → OPNsense's own Tailscale address | yes |
| **AWS LAN address → lab VM** | **no** |

The request reached AWS and was answered, but the reply (a subnet address going to another subnet address) never arrived.
The AWS security group, network ACL, Tailscale access policy, OPNsense firewall log and the VM's own firewall were each
tested and ruled out.

**Fix.** Source NAT on OPNsense's Tailscale interface, so the lab VM's source becomes OPNsense's own Tailscale address.
OPNsense 26.x: **Firewall → NAT → Source NAT** (called *Outbound* in older versions) → add a rule:

| Field | Value |
|---|---|
| Interface | `tailscale` |
| Version / Protocol | IPv4 / any |
| Source, Destination | `any`, `any` (the form rejected typed masks; use `any` or *Aliases* of type Network(s)) |
| Translate source IP | empty = the interface address |

Save, then **Apply**. Result: 5 of 5 replies from the lab VM to the AWS router.

**Trade-off.** AWS sees every lab machine as one source (OPNsense's Tailscale address).

**Root cause: not proven.** Tailscale's site-to-site guide says *"Both subnet routers must use a Linux-based operating
system"*, and that the `--snat-subnet-routes` flag *"only works if the operating system is Linux"*. A request for FreeBSD/OPNsense
support (GitHub issue #7073) was closed as a duplicate, and we did not find it added. That fits the symptom but is an
explanation, not a confirmed cause.

**Lesson.** Pinging the router proves the tunnel, not forwarding. Capture on both sides of each hop, and test with a
**control** packet that you know should arrive.
