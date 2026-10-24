# Hardware

## Overview

The router was built around a Raspberry Pi 5 with separate hardware for Ethernet switching and wireless access. This kept routing, switching, and Wi-Fi functions independent and made it possible to use OpenWrt for the network configuration without depending on an all-in-one consumer router.

The final system used:

- Raspberry Pi 5 Model B
- WD Green SN350 250 GB NVMe SSD
- Pimoroni NVMe Base
- Pimoroni NVMe Base Case
- Raspberry Pi Active Cooler
- Official Raspberry Pi 27 W USB-C power supply
- Cable Matters USB 3.0 Gigabit Ethernet adapter
- TP-Link TL-SG108E managed switch
- NETGEAR WAX210 Wi-Fi 6 access point
- ARRIS G34 cable gateway/modem
- microSD card for installation and recovery

## Raspberry Pi 5

The Raspberry Pi 5 serves as the main router and runs OpenWrt.

The final system reported:

| Property | Value |
|---|---|
| Board | Raspberry Pi 5 Model B Rev 1.1 |
| Architecture | ARMv8 |
| Memory | Approximately 8 GB |
| OpenWrt Target | `bcm27xx/bcm2712` |
| OpenWrt Version | 25.12.6 |
| Kernel | 6.12.112 |

The Pi handles routing, firewalling, DHCP, DNS, WireGuard, ad blocking, threat filtering, dynamic DNS, and traffic monitoring.

The available processing power and memory were significantly greater than required for the tested workload. During final testing, CPU utilization was effectively idle and only a small portion of the available memory was in use.

## NVMe Storage

A WD Green SN350 250 GB NVMe SSD was connected using a Pimoroni NVMe Base.

OpenWrt was initially brought up using microSD storage while the network configuration and boot process were being tested. The completed installation was then moved to the NVMe drive for normal operation.

Using NVMe provided a more permanent storage configuration while allowing the microSD card to remain available for recovery or reinstallation.

### NVMe Hardware

- **SSD:** WD Green SN350 250 GB
- **NVMe Adapter:** Pimoroni NVMe Base (PIM699)
- **Case:** Pimoroni NVMe Base Case (PIM771A)

The case accommodates the Raspberry Pi and NVMe Base as one assembly.

## Cooling

The Raspberry Pi 5 used the official Raspberry Pi Active Cooler.

During final router testing, CPU temperature was approximately **41.9 °C** under normal network operation.

No thermal limitation was observed during the project.

## Power Supply

The system used the official Raspberry Pi 27 W USB-C power supply.

Using the official supply provided the Pi 5 with its supported 5 V / 5 A power capability and avoided relying on a lower-power USB-C supply while also operating the NVMe hardware and USB Ethernet interface.

## Ethernet Interfaces

Two independent Gigabit Ethernet interfaces were required because the Raspberry Pi 5 includes only one built-in Ethernet port.

### Built-in Ethernet

The Pi's integrated Gigabit Ethernet interface was configured as `eth0` and used for the WAN connection.

During household deployment, this interface connected directly to the ARRIS G34 while the G34 was operating in bridge mode.

### USB Ethernet

A Cable Matters USB 3.0 Gigabit Ethernet adapter was used as `eth1`.

This interface served as the LAN connection and carried the 802.1Q VLAN trunk to the managed switch.

Both `eth0` and `eth1` successfully negotiated at **1 Gbit/s** during final testing.

### USB Ethernet Adapter Selection

A TP-Link UE306 was originally tested for the second Ethernet interface. The particular adapter used an ASIX AX88179B chipset and did not bind correctly to the available OpenWrt driver during the initial setup.

It was replaced with a Cable Matters USB 3.0 Gigabit Ethernet adapter, which worked correctly with OpenWrt and was used for the final system.

This was an important hardware compatibility consideration because adapters sold under similar product categories can use different Ethernet controller chipsets and therefore require different Linux/OpenWrt driver support.

## Managed Switch

A TP-Link TL-SG108E 8-port managed Gigabit switch was used to distribute the LAN networks.

The switch supports IEEE 802.1Q VLAN configuration, allowing the Raspberry Pi to use one physical LAN interface for multiple logical networks.

The switch was configured with:

- A tagged VLAN trunk to the Raspberry Pi
- A tagged VLAN connection to the wireless access point
- Untagged access ports for Trusted, IoT, and Guest wired devices
- Management/recovery ports on VLAN 1

The switch operates at Layer 2. Routing and firewall decisions remain on the Raspberry Pi.

## Wireless Access Point

A NETGEAR WAX210 was used as the Wi-Fi 6 access point.

The WAX210 operated as an access point only. DHCP, routing, DNS, and firewall functions remained on OpenWrt.

The AP provided separate wireless networks mapped to:

- VLAN 10 — Trusted
- VLAN 20 — IoT
- VLAN 30 — Guest

The connection between the WAX210 and TL-SG108E carried the wireless VLANs as tagged traffic while retaining VLAN 1 for management.

The access point was running firmware version **V1.1.0.31** during final testing.

## Wireless Performance

A laptop located near the WAX210 connected over 5 GHz 802.11ax and negotiated approximately 1201 Mbps receive and transmit PHY rates.

The measured Internet performance near the AP was approximately:

| Measurement | Result |
|---|---:|
| Download | 533 Mbps |
| Upload | 180 Mbps |
| Latency | 13 ms |

Wireless coverage, rather than router processing performance, became the main limitation during household deployment.

The single WAX210 provided strong performance near the access point but did not provide sufficient coverage throughout the multi-story house. Throughput in more distant areas fell below roughly 50 Mbps.

For a larger deployment, additional access points with wired backhaul would be preferable.

## ARRIS G34

An ARRIS G34 cable gateway was used for the Comcast/Xfinity Internet connection.

For normal household use, the G34 functions as a combined modem, router, and wireless gateway. During this project it was placed into bridge mode, disabling its normal routing function and allowing the Raspberry Pi to operate as the Internet-facing router.

An ISP DHCP issue was encountered after enabling bridge mode. This was eventually resolved by cloning the G34 WAN MAC address onto the Pi's WAN interface and performing a full modem power cycle.

Actual MAC addresses and public network information are intentionally not included in this repository.

## Managed Network Hardware

The final deployed hardware path was:

**ARRIS G34 → Raspberry Pi 5 → TP-Link TL-SG108E → wired devices / NETGEAR WAX210**

This arrangement separates the major network functions:

- **ARRIS G34:** cable modem / ISP connection
- **Raspberry Pi 5:** router, firewall, DNS, DHCP, VPN, and security services
- **TL-SG108E:** managed Layer 2 switching and VLAN distribution
- **WAX210:** Wi-Fi 6 wireless access

Keeping these functions separate made each part of the network independently configurable and easier to troubleshoot.

## Recovery Hardware

A microSD card was retained after the completed system was moved to NVMe.

The card can be used to reinstall or recover OpenWrt if the NVMe installation becomes inaccessible.

The final OpenWrt configuration was also backed up separately so the router can be restored without recreating the entire network configuration manually.

The production backup is not included in this repository because it contains private configuration data, including VPN and dynamic DNS credentials.

## Final Hardware Configuration

| Function | Hardware |
|---|---|
| Routing / Firewall | Raspberry Pi 5 |
| Primary Storage | WD Green SN350 NVMe SSD |
| WAN | Raspberry Pi built-in Gigabit Ethernet |
| LAN / VLAN Trunk | Cable Matters USB 3.0 Gigabit Ethernet |
| Switching | TP-Link TL-SG108E |
| Wireless | NETGEAR WAX210 |
| Internet Connection | ARRIS G34 in bridge mode |
| Cooling | Raspberry Pi Active Cooler |
| Power | Raspberry Pi 27 W USB-C PSU |
| Recovery | microSD card + configuration backup |
