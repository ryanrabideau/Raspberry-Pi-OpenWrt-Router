# Testing and Validation

## Overview

The completed router was tested at multiple levels before and during household deployment.

Testing covered:

- WAN connectivity
- VLAN addressing and DHCP
- Inter-VLAN isolation
- Wired and wireless connectivity
- DNS resolution
- DNSSEC validation
- DNS-over-TLS
- AdBlock and BanIP
- WireGuard remote access
- Dynamic DNS
- Firewall exposure
- Reboot persistence
- Ethernet link speed
- Wi-Fi performance
- CPU, memory, and thermal behavior
- Real-world wireless coverage

The goal was not only to confirm that individual services started successfully, but also to verify that the complete network behaved as intended.

## WAN Connectivity

After the ARRIS G34 bridge-mode issue was resolved, the Raspberry Pi received a public IPv4 lease from the ISP on `eth0`.

The WAN interface had:

- A valid public IPv4 address
- An ISP-provided default gateway
- Working IPv4 Internet connectivity

The actual public address and gateway are intentionally omitted from this repository.

Basic external connectivity and DNS resolution were tested after the WAN connection was established.

## Ethernet Link Validation

Both Raspberry Pi Ethernet interfaces successfully negotiated at:

```text
1000 Mb/s
```

This included:

- `eth0` — built-in Raspberry Pi Gigabit Ethernet used for WAN
- `eth1` — USB 3.0 Gigabit Ethernet adapter used for the LAN VLAN trunk

This confirmed that neither physical link had unexpectedly fallen back to 100 Mb/s.

## VLAN and DHCP Testing

The Trusted, IoT, and Guest networks were tested to verify that clients received addresses from the correct subnet.

Expected addressing was:

| Network | Expected Subnet |
|---|---|
| Trusted | `10.0.10.0/24` |
| IoT | `10.0.20.0/24` |
| Guest | `10.0.30.0/24` |
| VPN | `10.0.40.0/24` |
| Management / Recovery | `192.168.1.0/24` |

Clients connected through the appropriate switch ports or wireless networks were checked to ensure that VLAN tagging and DHCP assignment were working together correctly.

This tested the complete path through the client, access port or wireless SSID, managed switch, VLAN trunk, OpenWrt bridge, and DHCP service.

## Inter-VLAN Isolation

Network segmentation was tested to confirm that placing devices on different VLANs resulted in the intended firewall behavior.

IoT and Guest clients retained Internet connectivity while being restricted from initiating unrestricted connections into the Trusted network.

Required router services such as DHCP and DNS remained available.

This confirmed that the VLANs were not simply separate Layer 2 broadcast domains, but were also being enforced by the Layer 3 firewall policy.

## Guest Client Isolation

Guest wireless client isolation was enabled on the WAX210.

This added an additional restriction within the Guest wireless network so clients associated with the same Guest SSID could not directly communicate with one another.

The Guest network therefore used both:

- OpenWrt firewall isolation from other network segments
- Wireless client isolation between guest devices

## DNS Resolution

Normal DNS resolution was tested through the router.

Client requests followed the configured path:

**Client → dnsmasq → Unbound → Quad9**

Normal domains resolved successfully while the router remained the client-facing DNS server.

## DNSSEC Validation

DNSSEC was tested using an intentionally invalid DNSSEC domain:

```text
dnssec-failed.org
```

A lookup through the local resolver returned:

```text
SERVFAIL
```

This was the expected result.

A normal domain such as `example.com` continued to resolve successfully.

Testing both cases provided stronger validation than checking only successful DNS resolution because it demonstrated that invalid DNSSEC data was actually being rejected.

## DNS-over-TLS

Unbound was configured to forward upstream DNS traffic to Quad9 using DNS-over-TLS on TCP port 853.

The resolver configuration and active DNS operation were checked after deployment to confirm that DNS remained functional through the encrypted upstream path.

DNS-over-TLS protected the connection between the router and Quad9. Local clients still communicated with the router over the LAN using the standard DNS service.

## AdBlock

AdBlock was checked after the router had been fully configured.

During final testing, approximately:

```text
334,000
```

domains were present in the active block configuration.

Normal DNS resolution continued to function while the block lists were active.

The test confirmed that the service loaded successfully alongside the Unbound-based DNS configuration.

## BanIP

BanIP was also checked during final validation.

Approximately:

```text
1,900
```

network elements were active in the BanIP configuration.

The service remained active alongside normal Internet connectivity and the rest of the firewall configuration.

## WireGuard Remote Access

WireGuard was tested from outside the local network rather than from another device on the same Wi-Fi.

An iPhone 15 was disconnected from local Wi-Fi and used its cellular connection to establish the VPN.

The external test confirmed:

- A successful WireGuard handshake
- Connectivity to the router through the VPN
- Access to the OpenWrt management interface
- DNS resolution
- Internet connectivity using the configured routes

Testing from cellular data confirmed that the VPN was actually reachable through the WAN and dynamic DNS configuration rather than only working internally.

## Dynamic DNS

Dynamic DNS was used for the WireGuard endpoint because the residential public IPv4 address was not assumed to remain static.

The DDNS configuration was checked together with the external WireGuard test.

Successful VPN access using the configured hostname demonstrated that the remote endpoint could be reached without requiring the client configuration to depend directly on a fixed public IP address.

The production hostname and authentication token are omitted from this repository.

## WAN Exposure

The WAN-facing configuration was reviewed to ensure that administrative services were not unintentionally exposed.

LuCI and SSH were not configured for general WAN access.

WireGuard UDP port 51820 was intentionally permitted for remote VPN connectivity.

The live firewall configuration was also checked to confirm that obsolete disabled rules were not active in the resulting nftables ruleset.

## Reboot Persistence

After the router was configured, it was deliberately rebooted and checked again.

The system successfully returned with:

- WAN connectivity
- VLAN interfaces
- DHCP
- DNS
- Firewall rules
- AdBlock
- BanIP
- WireGuard
- Dynamic DNS

This verified that the completed configuration was persistent rather than depending on temporary runtime commands.

## CPU and Memory

The Raspberry Pi 5 had substantial unused processing capacity during normal router operation.

During final testing:

- CPU load was effectively idle
- Reported load averages were approximately `0.00 0.00 0.00`
- Only a small fraction of the available memory was in use
- Several gigabytes of RAM remained available

The tested workload therefore did not approach the processing or memory limits of the Raspberry Pi 5.

## Thermal Testing

The Raspberry Pi reported a temperature of approximately:

```text
41.9 °C
```

during final testing.

The official Raspberry Pi Active Cooler was installed.

No thermal throttling or temperature-related instability was observed.

## Wi-Fi Performance

A Wi-Fi 6 client was tested near the NETGEAR WAX210.

The client used:

- 5 GHz Wi-Fi
- 802.11ax
- Channel 36
- Approximately 1201 Mbps negotiated receive/transmit PHY rates

Measured Internet performance was approximately:

| Measurement | Result |
|---|---:|
| Download | 533 Mbps |
| Upload | 180 Mbps |
| Latency | 13 ms |

This demonstrated that the router, managed switch, VLAN configuration, and access point were capable of delivering several hundred megabits per second to a nearby wireless client.

## Coverage Testing

Real-world deployment also revealed the primary limitation of the system.

Although performance near the WAX210 was strong, throughput dropped below approximately 50 Mbps in parts of the multi-story house farther from the access point.

Basic browsing and streaming remained functional, but the wireless performance was significantly lower than near the AP and lower than the household's existing multi-device wireless setup.

This result helped distinguish two separate questions:

1. Could the Raspberry Pi router and network architecture handle the traffic?
2. Could one access point provide adequate RF coverage for the entire building?

Testing showed that the routing architecture worked correctly, while the single-AP wireless design was not ideal for the size and layout of the deployment environment.

## Deployment Evaluation

The router was successfully used as the Internet-facing router during the deployment test.

The following major functions operated together successfully:

- ISP WAN connectivity
- Gigabit Ethernet
- VLAN segmentation
- Inter-VLAN firewall isolation
- Wi-Fi VLAN mapping
- DNSSEC
- DNS-over-TLS
- AdBlock
- BanIP
- WireGuard
- Dynamic DNS
- Network monitoring

The system was eventually removed from active household deployment because of Wi-Fi coverage rather than a routing, firewall, or Raspberry Pi performance problem.

The original household network was restored and the completed OpenWrt configuration was preserved for future deployment.

## Recovery Validation

After the testing period, the Raspberry Pi was shut down cleanly before being disconnected.

The household network was returned to its original ARRIS G34 configuration and Internet connectivity was verified after the rollback.

The final OpenWrt configuration was backed up privately so the tested router state can be restored without rebuilding the entire configuration from scratch.

The backup itself is not included in this repository because it contains private keys and authentication credentials.

## Test Result Summary

| Test | Result |
|---|---|
| WAN IPv4 connectivity | Pass |
| Gigabit WAN link | Pass |
| Gigabit LAN link | Pass |
| Trusted VLAN | Pass |
| IoT VLAN | Pass |
| Guest VLAN | Pass |
| Inter-VLAN isolation | Pass |
| Guest client isolation | Pass |
| DHCP | Pass |
| Normal DNS resolution | Pass |
| DNSSEC validation | Pass |
| DNS-over-TLS | Pass |
| AdBlock | Pass |
| BanIP | Pass |
| WireGuard external access | Pass |
| Dynamic DNS | Pass |
| Firewall persistence | Pass |
| Reboot persistence | Pass |
| CPU / memory headroom | Pass |
| Thermal operation | Pass |
| Near-AP Wi-Fi performance | Pass |
| Whole-home single-AP coverage | Limited |

The final coverage result represents a deployment limitation rather than a failure of the OpenWrt router itself.
