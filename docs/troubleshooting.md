# Troubleshooting

## Overview

Several issues were encountered while building and deploying the router. The most significant involved Ethernet adapter compatibility, network addressing during initial OpenWrt setup, and ISP DHCP behavior after placing the ARRIS G34 into bridge mode.

These problems were diagnosed by isolating individual parts of the network rather than changing multiple configurations simultaneously.

## Initial LAN Address Conflict

During early OpenWrt setup, the router used the default `192.168.1.1` LAN address.

This created the possibility of an address or subnet conflict with existing network equipment while the Raspberry Pi was being configured alongside the household network.

The solution was to separate the OpenWrt test network from the existing network and establish a known management path before configuring the final VLAN interfaces.

The completed design retained `192.168.1.0/24` as a management/recovery network while placing normal client traffic on separate `10.0.x.0/24` networks.

This provided a predictable recovery path if VLAN configuration prevented access through the primary networks.

## USB Ethernet Adapter Compatibility

The Raspberry Pi 5 has one built-in Gigabit Ethernet interface, so a second Ethernet interface was required for the LAN side of the router.

A TP-Link UE306 was initially tested for this purpose.

The adapter was detected at the USB level, but its Ethernet controller did not bind correctly to the expected OpenWrt network driver in the tested configuration.

The particular adapter used an ASIX AX88179B controller.

Rather than continuing to modify the OpenWrt installation around an uncertain adapter/driver combination, the adapter was replaced with a Cable Matters USB 3.0 Gigabit Ethernet adapter.

The replacement adapter worked correctly and appeared as:

```text
eth1
```

It was then used for the final LAN VLAN trunk.

### Lesson

USB Ethernet adapters that appear functionally identical can use different controller chipsets.

Linux/OpenWrt compatibility should therefore be checked at the chipset and driver level rather than relying only on the retail product name.

## VLAN Configuration and Management Access

VLAN configuration can temporarily remove access to the router if the computer, switch port, and OpenWrt interface are not using compatible tagging and addressing.

To reduce this risk, VLAN 1 was retained as an untagged management/recovery network.

The final LAN trunk carried:

- VLAN 1 untagged
- VLAN 10 tagged
- VLAN 20 tagged
- VLAN 30 tagged

This meant a known untagged management path remained available while the tagged networks were being configured.

The approach was especially useful while coordinating settings between:

- OpenWrt bridge VLAN filtering
- TL-SG108E VLAN membership
- TL-SG108E PVID configuration
- WAX210 VLAN mapping

### Lesson

When introducing VLANs into an existing management path, retaining a known recovery network can make configuration errors much easier to recover from.

## ARRIS G34 Bridge Mode DHCP Failure

The most significant deployment issue occurred after placing the ARRIS G34 into bridge mode.

The intended connection was:

**ISP → ARRIS G34 → Raspberry Pi `eth0`**

The Raspberry Pi successfully established the physical Ethernet link, but initially failed to receive a public IPv4 address from the ISP.

## Initial Symptoms

The WAN interface had physical link but no usable IPv4 DHCP lease.

OpenWrt repeatedly attempted DHCP without receiving the expected response.

At this point, possible causes included:

- OpenWrt WAN configuration
- Raspberry Pi Ethernet configuration
- DHCP client behavior
- ARRIS G34 bridge-mode behavior
- ISP provisioning or DHCP behavior

Rather than assuming the problem was inside OpenWrt, the WAN traffic was inspected directly.

## Packet Capture

A packet capture was performed on the WAN interface while DHCP was running.

The capture showed DHCP Discover packets leaving the Raspberry Pi, but no corresponding DHCP Offer returning from the ISP.

Conceptually, the observed sequence was:

```text
Pi → DHCP Discover
ISP → no DHCP Offer observed
```

This showed that the Pi was attempting DHCP correctly enough to transmit discovery traffic.

The absence of an offer shifted attention away from the LAN/VLAN configuration and toward the modem/ISP side of the connection.

## Direct Laptop Test

To further isolate the issue, the Raspberry Pi was removed from the path and a laptop was connected directly to the bridged G34.

The laptop also failed to obtain a normal IPv4 DHCP lease and instead fell back to an automatically assigned local address.

Because two different DHCP clients showed the same behavior, the problem was unlikely to be caused solely by the Raspberry Pi's OpenWrt configuration.

This was an important troubleshooting step because it changed the problem from:

> Why is OpenWrt failing to get an address?

to:

> Why is the bridged modem/ISP path not providing an IPv4 lease to either client?

## MAC Address and ISP State

The next troubleshooting step was to account for the possibility that the modem or ISP DHCP system was retaining state associated with the previously connected WAN device.

The ARRIS G34 WAN MAC address was configured as the Raspberry Pi WAN interface MAC address.

In OpenWrt, this can be represented as:

```sh
uci set network.wan.macaddr='<G34-WAN-MAC>'
uci commit network
/etc/init.d/network restart
```

The actual MAC address is intentionally excluded from this repository.

Only the gateway's WAN MAC address was relevant to this configuration. Other modem hardware addresses were not used.

## Full Modem Power Cycle

Changing the WAN MAC alone was combined with a full power cycle of the ARRIS G34.

The modem was powered down long enough for the previous connection state to clear before being restarted with the Raspberry Pi connected as the WAN device.

After the modem restarted, OpenWrt successfully received a public IPv4 DHCP lease from Comcast/Xfinity.

The WAN interface then had:

- Public IPv4 addressing
- ISP default gateway
- Internet connectivity
- Working DNS

The exact public address and gateway are intentionally omitted.

## Why the Test Was Useful

The direct laptop test was particularly important because it prevented unnecessary changes to a working OpenWrt configuration.

Without that test, it would have been easy to continue changing:

- VLAN configuration
- Firewall rules
- DHCP client settings
- Ethernet drivers
- OpenWrt packages

even though those components were not the root cause of the missing ISP DHCP response.

Testing the same WAN path with a second device narrowed the problem to the modem/ISP side much more effectively.

## MAC Override Portability

The WAN MAC override was useful for this deployment, but it should not automatically be carried into every future deployment.

If the Raspberry Pi is moved to another modem or ISP, the override should be reviewed and removed or changed as appropriate.

A sanitized example for removing it is:

```sh
uci delete network.wan.macaddr
uci commit network
/etc/init.d/network restart
```

This prevents a deployment-specific workaround from becoming an unintended permanent dependency.

## Wireless Coverage

The router itself operated correctly during household deployment, but real-world testing identified a separate wireless coverage problem.

Near the WAX210, a Wi-Fi 6 client achieved approximately:

- 533 Mbps download
- 180 Mbps upload
- 13 ms latency

In more distant areas of the multi-story house, throughput dropped below approximately 50 Mbps.

The Raspberry Pi remained lightly loaded and both Ethernet links were operating at 1 Gbit/s, so the performance difference was not consistent with a router CPU or Ethernet bottleneck.

The change correlated with wireless distance and building layout.

### Resolution

The OpenWrt system was removed from active household deployment and the original network was restored.

The router configuration was preserved for future use.

For a larger environment, the preferred solution would be improved AP placement or multiple access points using wired backhaul rather than attempting to solve the coverage problem through router configuration.

### Lesson

Routing performance and wireless coverage are separate engineering problems.

A router can have substantial processing and Ethernet capacity while the overall user experience is still limited by RF propagation and access-point placement.

## Reboot Verification

After major configuration changes, the Raspberry Pi was rebooted intentionally.

This was important because a system can appear functional while relying on temporary commands or runtime state that will disappear after restart.

After reboot, the major services and interfaces returned successfully, including:

- WAN
- VLAN interfaces
- DHCP
- DNS
- Firewall
- AdBlock
- BanIP
- WireGuard
- Dynamic DNS

This confirmed that the final configuration had been saved persistently.

## Recovery Strategy

The project retained several recovery options:

- Management/recovery network on VLAN 1
- microSD card for reinstalling or recovering OpenWrt
- NVMe as the normal OpenWrt boot device
- Private final configuration backup

The production backup contains credentials and cryptographic material and is therefore not included in this repository.

Keeping recovery mechanisms separate from the primary configuration made it possible to experiment with routing, VLANs, and security settings without relying on a single access path.

## Troubleshooting Approach

The most useful troubleshooting pattern throughout the project was to isolate layers and components before changing configuration.

Examples included:

- Checking physical Ethernet link before debugging IP configuration
- Confirming the USB Ethernet chipset before changing network settings
- Testing VLAN membership independently of firewall policy
- Using packet capture to inspect DHCP behavior
- Connecting a laptop directly to the bridged modem to isolate OpenWrt
- Comparing near-AP and distant Wi-Fi performance to distinguish routing from RF limitations
- Rebooting after configuration to verify persistence

This approach reduced the number of variables involved in each test and made the resulting diagnosis more reliable.
