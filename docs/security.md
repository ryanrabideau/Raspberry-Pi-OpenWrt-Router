# Security

## Overview

The router was configured around a default-deny approach for traffic entering the router or crossing between network segments.

OpenWrt provided the primary firewall and network segmentation, while additional services handled encrypted DNS, DNSSEC validation, domain filtering, IP-based threat filtering, and encrypted remote access.

The security design included:

- Separate Trusted, IoT, Guest, and VPN networks
- Firewall isolation between network segments
- REJECT as the default input and forwarding policy
- No LuCI or SSH exposure to the WAN
- DNSSEC validation
- DNS-over-TLS to Quad9
- Network-wide ad and tracker blocking
- BanIP threat filtering
- WireGuard remote access
- Guest wireless client isolation

## Firewall Defaults

The OpenWrt firewall used the following default policies:

| Traffic Type | Default Policy |
|---|---|
| Input | REJECT |
| Output | ACCEPT |
| Forward | REJECT |

This means traffic directed at the router or forwarded between interfaces was denied unless it matched an allowed zone policy or explicit firewall rule.

Outbound traffic originating from the router itself was permitted.

## Firewall Zones

Separate firewall zones were used to represent different levels of trust.

### Trusted

The Trusted network contained personal computers, phones, and other devices considered part of the primary internal network.

Trusted devices were allowed to reach the Internet and internal resources as required.

### IoT

The IoT network contained smart-home and embedded devices.

IoT devices were permitted to reach the Internet and required router services such as DHCP and DNS, but unrestricted access into the Trusted network was not permitted.

This reduces the impact that a compromised or poorly secured IoT device could have on trusted systems.

### Guest

The Guest network was configured for Internet access without unrestricted access to the Trusted network.

Guest devices were allowed to use required router services such as DHCP and DNS.

Wireless client isolation was also enabled on the Guest SSID so guest devices could not directly communicate with one another over the access point.

### WAN

The WAN zone rejected unsolicited inbound traffic by default.

LuCI and SSH were not exposed to the WAN.

The externally reachable service intentionally configured for remote access was WireGuard over UDP port 51820, in addition to normal network infrastructure and ICMP traffic required for correct WAN operation.

### VPN

WireGuard used a separate firewall zone associated with `wg0`.

This allowed VPN traffic to be controlled independently from locally connected devices.

Remote VPN clients were given access only to the internal services required for remote use and administration rather than being treated as unrestricted WAN traffic.

## Inter-VLAN Isolation

The VLANs provide Layer 2 separation, while the OpenWrt firewall provides Layer 3 policy enforcement between the networks.

This distinction is important because creating separate VLANs and subnets alone does not prevent a router from forwarding traffic between them.

The firewall was therefore used to control communication between:

- Trusted and IoT
- Trusted and Guest
- VPN and internal networks
- Local networks and WAN

IoT and Guest devices could still reach the Internet while remaining restricted from initiating unrestricted connections into the Trusted network.

## DNS Security

DNS services were centralized on OpenWrt.

The DNS path was:

**Client → dnsmasq → Unbound → Quad9 over DNS-over-TLS**

Clients communicated with the router using normal local DNS. dnsmasq then forwarded the requests locally to Unbound.

Unbound forwarded upstream requests to Quad9 using DNS-over-TLS on TCP port 853.

This encrypted the DNS traffic between the router and upstream resolver.

## DNSSEC

DNSSEC validation was enabled to allow the resolver to detect invalid DNSSEC signatures rather than accepting improperly authenticated DNS responses.

Validation was tested using `dnssec-failed.org`, a domain intentionally configured to fail DNSSEC validation.

The test returned:

```text
SERVFAIL
```

Normal domains continued to resolve successfully.

This provided a functional negative test showing that invalid DNSSEC data was being rejected rather than simply confirming that ordinary DNS resolution worked.

## Ad Blocking

AdBlock was installed on OpenWrt to provide centralized domain-based filtering.

At final testing, the active configuration contained approximately **334,000 blocked domains**.

The configured feeds included sources for advertising, tracking, and certificate-related filtering.

Because filtering occurs at the router, supported clients receive the benefit without requiring a separate browser extension or filtering application on every device.

Domain blocking is used as an additional network filtering layer and is not treated as a replacement for endpoint security.

## BanIP

BanIP was used for IP-based filtering using external threat intelligence feeds.

The active BanIP configuration contained approximately **1,900 network elements** during final testing.

Configured sources included feeds covering categories such as compromised hosts, malware infrastructure, spam-related networks, and known hostile addresses.

BanIP complements domain-based filtering by operating on network addresses rather than DNS names.

It is used as an additional filtering mechanism rather than being described as a full intrusion prevention system.

## WireGuard

WireGuard provided encrypted remote access to the network.

The VPN used:

- UDP port `51820`
- VPN subnet `10.0.40.0/24`
- Router address `10.0.40.1`
- Dedicated OpenWrt firewall zone
- Dynamic DNS for the residential WAN connection

Remote connectivity was tested from an iPhone 15 using cellular data rather than the local Wi-Fi network.

The test confirmed:

- Successful WireGuard handshake
- Remote access to the OpenWrt management interface
- DNS resolution
- Internet connectivity through the configured tunnel routes

The production WireGuard private key is not stored in this repository.

## Dynamic DNS

The residential public IPv4 address was not assumed to remain permanently fixed.

Dynamic DNS was therefore used to provide a stable hostname for the WireGuard endpoint.

The production hostname and authentication token are not included in the repository. Sanitized examples use placeholder values instead.

## WAN Exposure

The WAN interface was checked to ensure that management services were not unintentionally exposed.

The intended externally reachable service was WireGuard on UDP port 51820.

LuCI and SSH remained accessible only from appropriate internal networks and were not intentionally opened to the public Internet.

This reduces the number of services directly exposed to unsolicited WAN traffic.

## Wireless Security

The WAX210 used WPA2/WPA3 security for the configured wireless networks.

The Trusted, IoT, and Guest SSIDs were mapped to separate VLANs so wireless devices inherited the same network segmentation used for wired clients.

Guest client isolation was enabled to prevent wireless guest devices from directly communicating with one another.

Client isolation was not enabled on the IoT network because some IoT devices may require local communication depending on their function.

## Secrets and Repository Sanitization

The operational router configuration contains information that should not be published in a public repository.

The following information is intentionally excluded or replaced with placeholders:

- WireGuard private keys
- Dynamic DNS authentication tokens
- Wi-Fi passwords
- Production credentials
- Public WAN addresses
- Device MAC addresses
- Other authentication secrets

WireGuard public keys and public endpoint information are also omitted from the example configuration where they are not required to demonstrate the design.

The repository contains sanitized examples rather than a direct copy of the production configuration.

## Configuration Backup

A final OpenWrt configuration backup was created after the system had been validated.

That backup contains sensitive configuration information, including VPN and dynamic DNS credentials, and is therefore intentionally not included in the repository.

The backup is retained privately for system recovery.

## Security Limitations

The security configuration improves segmentation and reduces unnecessary exposure, but it does not make the network immune to compromise.

AdBlock and BanIP provide additional filtering but do not replace:

- Endpoint security
- Software updates
- Strong authentication
- Secure device configuration
- Application-level security

Likewise, VLAN segmentation limits network access according to firewall policy but does not inherently make devices within a VLAN trustworthy.

The design therefore uses multiple security layers rather than depending on any single feature.
