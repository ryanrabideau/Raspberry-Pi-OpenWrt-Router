# Network Design

## Overview

The network was designed around multiple isolated IPv4 subnets rather than placing every device on a single LAN.

OpenWrt performs routing between the networks and applies firewall policy at each boundary. A TP-Link TL-SG108E managed switch distributes the Ethernet VLANs, while a NETGEAR WAX210 maps separate wireless networks to the same VLAN infrastructure.

The primary networks were:

| Network | VLAN | Subnet | Router Address | Purpose |
|---|---:|---|---|---|
| Trusted | 10 | `10.0.10.0/24` | `10.0.10.1` | Personal and trusted devices |
| IoT | 20 | `10.0.20.0/24` | `10.0.20.1` | Smart-home and IoT devices |
| Guest | 30 | `10.0.30.0/24` | `10.0.30.1` | Guest devices |
| VPN | — | `10.0.40.0/24` | `10.0.40.1` | WireGuard remote clients |
| Management / Recovery | 1 | `192.168.1.0/24` | `192.168.1.1` | Network infrastructure and recovery |

## Interface Assignment

The Raspberry Pi used its two Ethernet interfaces for separate WAN and LAN roles.

| Interface | Function |
|---|---|
| `eth0` | WAN connection |
| `eth1` | LAN VLAN trunk |
| `br-lan.1` | Management / Recovery |
| `br-lan.10` | Trusted |
| `br-lan.20` | IoT |
| `br-lan.30` | Guest |
| `wg0` | WireGuard VPN |

The built-in Raspberry Pi Ethernet controller was used for WAN, while the USB Gigabit Ethernet adapter carried all LAN VLANs to the managed switch.

## VLAN Design

OpenWrt used bridge VLAN filtering on `br-lan` to create the Ethernet VLANs.

The LAN Ethernet interface, `eth1`, was a member of the bridge. VLANs 10, 20, and 30 were tagged on this interface so they could share the same physical Ethernet connection.

The management/recovery VLAN remained untagged on the trunk.

Conceptually, the LAN bridge was configured as:

| VLAN | `eth1` |
|---:|---|
| 1 | Untagged / PVID |
| 10 | Tagged |
| 20 | Tagged |
| 30 | Tagged |

The resulting OpenWrt interfaces were:

- `br-lan.1` — `192.168.1.1/24`
- `br-lan.10` — `10.0.10.1/24`
- `br-lan.20` — `10.0.20.1/24`
- `br-lan.30` — `10.0.30.1/24`

The WireGuard interface was independent of `br-lan` and used `10.0.40.1/24`.

## DHCP Design

OpenWrt provided DHCP service for the local networks.

The Trusted, IoT, and Guest networks each used their own DHCP scope within their respective `/24` subnet.

The primary dynamic address range was `.100` through `.249` on each VLAN.

For example:

| Network | Example DHCP Range |
|---|---|
| Trusted | `10.0.10.100` – `10.0.10.249` |
| IoT | `10.0.20.100` – `10.0.20.249` |
| Guest | `10.0.30.100` – `10.0.30.249` |

Keeping the lower portion of each subnet outside the primary dynamic pool leaves addresses available for infrastructure, static assignments, or reserved devices.

## Managed Switch Configuration

The TL-SG108E was configured to distribute tagged and untagged VLAN traffic depending on the device connected to each port.

| Port | Connected Device / Purpose | VLAN 1 | VLAN 10 | VLAN 20 | VLAN 30 | PVID |
|---:|---|---|---|---|---|---:|
| 1 | Raspberry Pi LAN trunk | Untagged | Tagged | Tagged | Tagged | 1 |
| 2 | Trusted access port | Untagged | Untagged | — | — | 10 |
| 3 | IoT access port | Untagged | — | Untagged | — | 20 |
| 4 | Guest access port | Untagged | — | — | Untagged | 30 |
| 5 | WAX210 | Untagged | Tagged | Tagged | Tagged | 1 |
| 6 | Management / Recovery | Untagged | — | — | — | 1 |
| 7 | Management / Recovery | Untagged | — | — | — | 1 |
| 8 | Management / Recovery | Untagged | — | — | — | 1 |

The PVID determines which VLAN is assigned to untagged traffic entering a port.

Ports 2, 3, and 4 therefore behave as ordinary access ports for Trusted, IoT, and Guest devices respectively. Devices connected to those ports do not need to understand VLAN tagging.

Ports 1 and 5 carry tagged traffic because the Raspberry Pi and WAX210 need access to multiple VLANs.

## Wireless VLAN Mapping

The WAX210 used a single Ethernet connection to carry multiple wireless networks.

| SSID Role | VLAN |
|---|---:|
| Home / Trusted | 10 |
| IoT | 20 |
| Guest | 30 |

Traffic from each SSID was tagged with its assigned VLAN before being sent through the Ethernet uplink.

The managed switch preserved those tags on the connection between the WAX210 and Raspberry Pi.

This allowed wireless and wired devices belonging to the same logical network to use the same subnet and firewall policy.

## Management Network

VLAN 1 was retained as a management and recovery network using `192.168.1.0/24`.

During the final configuration:

- OpenWrt was reachable at `192.168.1.1`
- The WAX210 management interface was on the same management subnet
- The TL-SG108E management interface was on the same management subnet

This network was particularly useful during initial configuration and troubleshooting because it provided an untagged path to the infrastructure devices.

Exact infrastructure addresses beyond the router are not required to reproduce the design and can be assigned as appropriate for another deployment.

## Routing

OpenWrt maintained directly connected routes for each local network.

The routing table included:

- `10.0.10.0/24` through `br-lan.10`
- `10.0.20.0/24` through `br-lan.20`
- `10.0.30.0/24` through `br-lan.30`
- `10.0.40.0/24` through `wg0`
- `192.168.1.0/24` through `br-lan.1`

The WAN default route was obtained from the ISP.

Public WAN addressing is intentionally omitted from this repository.

## Firewall Segmentation

Although OpenWrt has routes to all local networks, routing alone does not imply that every network is allowed to communicate with every other network.

Firewall zones were used to enforce the intended trust boundaries.

The general design was:

- Trusted devices could access the Internet and internal resources as required.
- IoT devices could access the Internet and required router services but were restricted from initiating unrestricted connections into the Trusted network.
- Guest devices could access the Internet and required router services but were isolated from the Trusted network.
- Guest wireless clients were additionally isolated from one another at the access point.
- WireGuard clients were placed in a dedicated VPN zone and given only the required internal access.
- Unsolicited WAN traffic was rejected except for explicitly permitted services such as WireGuard.

This separates network reachability from network authorization: OpenWrt knows how to route between the subnets, while the firewall determines whether that traffic is permitted.

## DNS

Clients used the Raspberry Pi as their DNS server rather than communicating directly with public DNS resolvers.

The DNS path was:

**Client → dnsmasq → Unbound → Quad9 over DNS-over-TLS**

This centralized DNS policy across the Trusted, IoT, Guest, and VPN networks.

It also allowed DNSSEC validation and domain-based filtering to be applied at the router.

## WireGuard Network

WireGuard used the dedicated `10.0.40.0/24` subnet.

The OpenWrt router used:

`10.0.40.1/24`

A remote client used an address from the same VPN subnet.

Unlike the Ethernet networks, this subnet did not require an 802.1Q VLAN because WireGuard creates its own Layer 3 tunnel interface.

The VPN interface was assigned to a dedicated firewall zone so remote access could be controlled separately from local LAN traffic.

## WAN Design

The ARRIS G34 was placed into bridge mode during deployment.

This removed the G34's normal routing role and allowed the Raspberry Pi WAN interface to receive the ISP connection directly.

The Pi therefore became responsible for:

- WAN connectivity
- NAT
- Firewalling
- Local routing
- DHCP
- DNS
- VPN termination

An ISP DHCP issue was encountered during initial bridge-mode deployment. Packet capture showed DHCP Discover traffic leaving the Pi without a corresponding DHCP Offer.

The same behavior occurred when a laptop was connected directly to the bridged modem, helping distinguish the problem from the OpenWrt LAN/VLAN configuration.

The issue was resolved by using the gateway's WAN MAC address on the Pi WAN interface and performing a full modem power cycle.

The actual MAC address and public IPv4 information are intentionally omitted.

## Design Rationale

The VLAN design separates devices according to trust rather than physical connection type.

For example, a wired IoT device and a wireless IoT device can both belong to VLAN 20 and receive the same firewall policy even though they reach the router through different physical paths.

Likewise, Trusted devices can use either Ethernet or Wi-Fi while remaining on VLAN 10.

This makes the network easier to expand because security policy is associated with logical networks instead of individual switch ports or wireless devices.

The design also keeps the routing and security policy centralized on OpenWrt while allowing the managed switch and access point to focus on Layer 2 connectivity.
