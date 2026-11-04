# Glossary

| Term | Meaning |
|---|---|
| **CIDR / prefix** | Notation `address/length`; the length is how many leading bits are the network. `/24` = 256 addresses |
| **Octet** | One of the four 8-bit numbers in an IPv4 address |
| **VPC** | Virtual Private Cloud: your private network inside AWS |
| **Subnet** | A slice of a VPC in one Availability Zone |
| **Availability Zone (AZ)** | One data centre within a region; a subnet's AZ cannot be changed |
| **Internet gateway (IGW)** | The VPC's door to the internet |
| **Route table** | List of "destination → next hop" rules; the most specific match wins |
| **Security group** | Stateful, allow-only firewall attached to an instance |
| **Network ACL** | Stateless firewall at the subnet edge (rules needed in both directions) |
| **Stateful** | Replies to allowed traffic are allowed automatically |
| **Source/destination check** | EC2 setting that drops packets not to/from the instance; must be off for a router |
| **Tailnet** | Your private Tailscale network |
| **WireGuard** | The encrypted tunnel technology Tailscale uses |
| **DERP** | Tailscale's relay servers, used when a direct path is not possible |
| **Subnet router** | A Tailscale node that lets the tailnet reach a whole network behind it |
| **Advertise routes** | "I have this network behind me" |
| **Accept routes** | "I will use routes other nodes advertise" |
| **Route approval** | Admin-console step required before an advertised route is used |
| **NAT / source NAT** | Rewriting a packet's source address so replies return to a different address |
| **VLAN** | A separate layer-2 network segment on shared hardware, usually its own `/24` |
| **TTL** | Counter reduced by each router hop; used to tell how many routers a packet crossed |
| **tcpdump** | Packet-capture tool used to see what actually crosses an interface |
