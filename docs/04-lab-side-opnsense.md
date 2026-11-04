# 4. The lab side: OPNsense as the second subnet router

OPNsense is already the router between your VLANs, so it is the natural Tailscale node for the lab: any lab VM that uses it
as gateway can reach the tunnel with no extra setup on the VM.

## 4.1 Settings (VPN → Tailscale)
| Where | Setting | Value |
|---|---|---|
| Settings tab | Enabled | on |
| Settings tab | **Accept Subnet Routes** | **on** — teaches OPNsense that `10.50.0.0/16` lives behind the AWS router |
| Advertised Routes tab | list | **only the lab VLANs**, e.g. `192.168.70.0/24` — **never** `10.50.0.0/16` |

After saving, confirm the route exists: **System → Routes → Status** should show
`10.50.0.0/16 → tailscale0`. If the box was unticked, that line is missing and traffic for AWS goes out the WAN and is lost
(we reproduced this: unticking it broke lab → AWS, ticking it fixed it).

Advertise **only what AWS should reach** (least exposure). Names here: `cloud`, `manager`, … are examples of VLAN names.

## 4.2 How VLANs talk to each other (context)
Each VLAN is its own `/24`. VMs send traffic for other networks to their VLAN gateway (OPNsense, `.1`); OPNsense checks its
firewall rules on the interface where the packet arrives, then routes it. Check a VM's gateway with `ip route get <target>`.

## 4.3 The Source NAT rule that made lab → AWS work
Without it, replies from AWS were lost (chapter 5 shows exactly how we proved this). The fix is **source NAT** on the
Tailscale interface: rewrite the lab VM's source address to OPNsense's own Tailscale address.

OPNsense 26.x: **Firewall → NAT → Source NAT** (called *Outbound* in older versions) → add:

| Field | Value |
|---|---|
| Interface | `tailscale` |
| TCP/IP version | IPv4 |
| Protocol | any |
| Source / Destination | `any` / `any` (the form rejected typed CIDRs; use `any` or create *Aliases* of type Network(s)) |
| Translate source IP | leave empty = the interface address |

Save, then **Apply**. Trade-off: AWS sees all lab traffic as coming from OPNsense.

## 4.4 Optional: web GUI on the Tailscale address after reboot
We found the GUI only answered on the Tailscale address after `configctl webgui restart` (a boot-order race: the GUI started
before `tailscale0` had its address). A small start-up script fixes it:
`/usr/local/etc/rc.syshook.d/start/99-webgui-after-tailscale` (executable) —
```sh
#!/bin/sh
( for i in $(seq 1 60); do
    ifconfig tailscale0 2>/dev/null | grep -q 'inet 100\.' && break
    sleep 5
  done
  /usr/local/sbin/configctl webgui restart ) &
```
OPNsense's root shell is `csh`: if you paste a heredoc and see `?` prompts, run `sh` first.

Next: [5. Packet flows and testing](05-packet-flows-and-testing.md)
