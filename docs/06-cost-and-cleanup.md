# 6. Cost and clean-up

Prices change and vary by region — **check the AWS pricing pages**; the figures below are rough orders of magnitude.

## 6.1 What costs money
| Item | Rough cost | Notes |
|---|---|---|
| Small EC2 instance (`t4g.nano`–`t4g.micro` / `t3.micro`) | a few dollars per month if left running | A free-tier label may apply to some types |
| Public IPv4 address | a few dollars per month while allocated | AWS now charges for public IPv4 |
| EBS root disk (8 GiB) | cents per month | exists while the instance exists, or stopped |
| **NAT Gateway** | **~$30+ per month** | the classic accidental cost — **not needed** here |
| VPC, subnet, route table, internet gateway, security group, network ACL | **free** | |
| Tailscale (personal use) | free tier | |

Set a **budget alert** before you start.

## 6.2 Stop vs terminate
| Action | What remains | Cost |
|---|---|---|
| **Stop** | instance definition + disk; the auto-assigned public IP is released | only the small disk charge |
| **Terminate** | nothing (root disk deleted with it by default) | none |

## 6.3 Clean-up checklist
1. EC2 → terminate the instance; confirm **Volumes** is empty.
2. EC2 → **Elastic IPs** empty (none were created).
3. VPC → **NAT gateways** empty (none were created).
4. Optionally delete the VPC (it deletes its subnets/route tables when empty) — or keep it, it is free.
5. Tailscale admin console → remove the `aws-router` machine.
6. OPNsense → untick *Accept Subnet Routes* and disable the Source NAT rule if nothing else uses them.
7. Check **Billing → Cost Explorer** after a day for stray charges.

Back to the [overview](../README.md) · Glossary: [07-glossary.md](07-glossary.md)
