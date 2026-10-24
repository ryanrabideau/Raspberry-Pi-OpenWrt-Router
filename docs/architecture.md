# System Architecture

## Overview

The router was designed around a Raspberry Pi 5 running OpenWrt, with routing and security functions separated from the Ethernet switching and wireless access hardware.

The Raspberry Pi handles Layer 3 routing, firewalling, DHCP, DNS, VPN access, and security services. A managed Ethernet switch handles VLAN distribution, while a separate Wi-Fi 6 access point provides wireless connectivity for the Trusted, IoT, and Guest networks.

During deployment, the overall network path was:

**Internet → ARRIS G34 → Raspberry Pi 5 → TP-Link TL-SG108E → wired devices / NETGEAR WAX210**

The ARRIS G34 was placed in bridge mode so that the Raspberry Pi received the public WAN connection and operated as the primary router.

## Router Interfaces

The Raspberry Pi used two Gigabit Ethernet interfaces:

| Interface | Role | Description |
|---|---|---|
| `eth0` | WAN | Raspberry Pi 5 built-in Gigabit Ethernet interface connected to the bridged ARRIS G34 |
| `eth1` | LAN | USB 3.0 Gigabit Ethernet adapter connected to the managed switch |

The LAN interface carried multiple networks over a single 802.1Q VLAN trunk.

Both interfaces successfully negotiated at 1 Gbit/s during final testing.

## VLAN Architecture

The network was divided into separate VLANs based on device trust and purpose.

| Network | VLAN | Subnet | Router Address |
|---|---:|---|---|
| Trusted | 10 | `10.0.10.0/24` | `10.0.10.1` |
| IoT | 20 | `10.0.20.0/24` | `10.0.20.1` |
| Guest | 30 | `10.0.30.0/24` | `10.0.30.1` |
| VPN | — | `10.0.40.0/24` | `10.0.40.1` |
| Management / Recovery | 1 | `192.168.1.0/24` | `192.168.1.1` |

VLAN 10 was used for trusted personal devices. VLAN 20 isolated IoT and smart-home devices, while VLAN 30 provided a separate guest network.

The WireGuard VPN used its own routed subnet rather than an Ethernet VLAN.

VLAN 1 remained available as a management and recovery network. This provided a known path to the router, switch, and access point while the other VLANs were being configured or troubleshot.

## OpenWrt Bridge

The LAN-side Ethernet adapter was added to the OpenWrt `br-lan` bridge. VLAN filtering was then used to create the routed VLAN interfaces:

- `br-lan.10` — Trusted
- `br-lan.20` — IoT
- `br-lan.30` — Guest
- `br-lan.1` — Management / Recovery

The Trusted, IoT, and Guest VLANs were tagged on the connection between the Raspberry Pi and managed switch. VLAN 1 was retained as the untagged management network.

The WireGuard interface, `wg0`, was separate from the Ethernet bridge and used `10.0.40.1/24`.

## Managed Switch

A TP-Link TL-SG108E managed switch distributed the VLANs between the router, wired devices, and wireless access point.

The final port layout was:

| Port | Purpose | VLAN Configuration |
|---:|---|---|
| 1 | Raspberry Pi LAN trunk | VLANs 10, 20, and 30 tagged; VLAN 1 untagged |
| 2 | Trusted wired devices | VLAN 10 untagged |
| 3 | IoT wired devices | VLAN 20 untagged |
| 4 | Guest wired devices | VLAN 30 untagged |
| 5 | NETGEAR WAX210 | VLANs 10, 20, and 30 tagged; VLAN 1 untagged |
| 6-8 | Management / recovery | VLAN 1 untagged |

This allowed ordinary wired devices to connect without VLAN configuration while still carrying multiple VLANs between the router, switch, and access point.

## Wireless Architecture

The NETGEAR WAX210 operated as a standalone Wi-Fi 6 access point rather than as a router.

Three wireless networks were mapped to the corresponding VLANs:

| Wireless Network | VLAN | Purpose |
|---|---:|---|
| Home | 10 | Trusted devices |
| IoT | 20 | Smart-home and IoT devices |
| Guest | 30 | Guest devices |

The access point did not provide DHCP or routing for these networks. Those functions remained centralized on OpenWrt.

Guest wireless client isolation was enabled so devices connected to the Guest SSID could not directly communicate with one another.

## Routing and Firewall Boundaries

OpenWrt was the Layer 3 boundary between all network segments.

The general policy was to allow devices to reach the Internet while preventing unnecessary communication between networks.

Trusted devices had the greatest level of access. IoT and Guest devices were restricted from initiating unrestricted connections into the Trusted network, while DHCP and DNS access to the router remained available.

The WireGuard network was also treated as its own firewall zone. Remote VPN clients were given access only to the internal resources required for remote administration and use.

More detailed firewall and security information is documented separately in `security.md`.

## DNS Architecture

DNS was centralized on the Raspberry Pi.

Client devices sent DNS requests to the router using the standard DNS port. dnsmasq handled the client-facing DNS service and forwarded requests locally to Unbound.

Unbound then forwarded upstream DNS requests to Quad9 over DNS-over-TLS.

The path was:

**Client → dnsmasq → Unbound → Quad9 over TLS**

This configuration allowed DNSSEC validation, encrypted upstream DNS, and network-wide domain filtering to be handled centrally instead of configured separately on every client.

## Remote Access

WireGuard provided encrypted remote access to the network.

The router used the `10.0.40.0/24` VPN subnet, with the OpenWrt WireGuard interface assigned `10.0.40.1`.

Dynamic DNS was used so remote clients could reach the router without depending on a fixed residential public IPv4 address.

The VPN was tested from an external cellular connection and successfully provided remote access to the router and permitted internal network resources.

## Design Considerations

Separating routing, switching, and wireless access into different devices made the network more configurable than an all-in-one consumer router.

The Raspberry Pi could be configured and upgraded independently of the switch and access point. VLANs could also be extended to both wired and wireless devices without requiring separate physical router interfaces for each network.

The design also made troubleshooting easier because routing, Ethernet switching, and Wi-Fi could be tested independently.

One tradeoff became apparent during real-world deployment. Although the router and VLAN architecture worked correctly, a single WAX210 did not provide sufficient wireless coverage throughout the multi-story test environment. A larger deployment would benefit from multiple access points with wired backhaul.

The routing architecture itself does not depend on a single access point, so additional APs could be added without changing the core OpenWrt design.
