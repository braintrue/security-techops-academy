
# Day 05 — Linux Networking & Firewall Fundamentals

## 1. Learning Objectives

- Linux network interface configuration
- IPv4 and IPv6 routing table analysis
- Default gateway and next-hop identification
- ARP / IPv6 Neighbor Discovery
- DNS resolution and systemd-resolved
- Listening socket analysis
- UFW firewall policy inspection
- nftables ruleset analysis

## 2. Network Interface

### Interface Information

| Item | Value |
|---|---|
| Interface | enp0s1 |
| Status | UP |
| IPv4 | 192.168.64.3/24 |
| MAC | 14:05:ed:fb:96:98 |
| IPv6 | fd31:bc20:a82a:c13c:1405:edff:fefb:9698/64 |

Loopback interface: lo

## 3. IPv4 Routing

### Routing Table

- Default Gateway: 192.168.64.1
- Network: 192.168.64.0/24
- Interface: enp0s1
- Metric: 100

### Route Lookup

Command:

```bash
ip route get 1.1.1.1
```

Result:

```text
1.1.1.1 via 192.168.64.1 dev enp0s1
src 192.168.64.3
```

The kernel selects 192.168.64.1 as the next hop
and enp0s1 as the outgoing interface.

## 4. IPv6 Routing

Observed routes:

```text
fd31:bc20:a82a:c13c::/64 dev enp0s1
proto ra metric 100

fe80::/64 dev enp0s1
proto kernel metric 256
```

- The first route is an IPv6 network prefix learned through Router Advertisement.
- The second route represents the link-local network.
- No IPv6 default route was observed in the routing table.

An assigned IPv6 address does not independently prove
that external IPv6 connectivity is available.

## 5. Neighbor Discovery

IPv4 neighbor:

- Gateway: 192.168.64.1
- MAC: 1e:57:dc:54:be:64
- State: REACHABLE

IPv6 neighbor entries included STALE state.

STALE indicates that reachability has not recently been
reconfirmed. It does not automatically mean connectivity failure.

## 6. DNS Configuration

DNS manager: systemd-resolved

Observed configuration:

| Item | Value |
|---|---|
| DNS IPv4 | 192.168.64.1 |
| DNS IPv6 | fe80::1c57:dcff:fe54:be64 |
| Interface | enp0s1 |
| DNS mode | stub |
| Default DNS Route | yes |
| LLMNR | Disabled |
| mDNS | Disabled |
| DNS-over-TLS | Disabled |
| DNSSEC | no/unsupported |

`getent ahosts ubuntu.com` returned IPv4 and IPv6
addresses, confirming successful name resolution.

## 7. Listening Sockets

Command:

```bash
sudo ss -tulpen
```

Observed services:

| Port | Protocol | Process | Purpose |
|---|---|---|---|
| 53 | TCP/UDP | systemd-resolved | DNS |
| 68 | UDP | systemd-networkd | DHCP client |
| 323 | UDP | chronyd | NTP/chrony |
| 22 | TCP | sshd | SSH |

SSH listens on:

- 0.0.0.0:22
- [::]:22

These indicate listening on all local IPv4 and IPv6 addresses.
They do not independently establish public Internet exposure.

## 8. UFW Firewall

Status: Active

Default policies:

- Incoming: deny
- Outgoing: allow
- Routed: disabled

SSH rules:

- 22/tcp ALLOW IN Anywhere
- 22/tcp ALLOW IN Anywhere (v6)

The firewall blocks unsolicited incoming traffic by default,
with explicit SSH exceptions for IPv4 and IPv6.

## 9. nftables

UFW manages firewall rules through the iptables-nft compatibility layer.

Observed policies:

| Chain | Policy |
|---|---|
| IPv4 INPUT | drop |
| IPv4 OUTPUT | accept |
| IPv4 FORWARD | drop |
| IPv6 INPUT | drop |
| IPv6 OUTPUT | accept |
| IPv6 FORWARD | drop |

Additional observations:

- Established and related connections are accepted.
- SSH traffic is accepted by ufw-user-input.
- The input policy drop counter recorded 249 packets / 18084 bytes.
- SSH accept counter recorded 5 packets / 320 bytes.

Packet counters alone are not sufficient to classify traffic as malicious.

The nftables ruleset was inspected without directly modifying
UFW-managed tables.

## 10. Security Observations

1. UFW is active with a default-deny inbound policy.
2. SSH is explicitly allowed from any source at the host firewall level.
3. SSH listens on both IPv4 and IPv6.
4. Public exposure cannot be concluded from the host firewall alone.
5. IPv4 routing and DNS resolution were confirmed.
6. An IPv6 default route was not observed.
7. Firewall drop counters require contextual log analysis.

## 11. Commands Practiced

```bash
ip addr
ip route
ip route get 1.1.1.1
ip -6 route
ip neigh show
getent ahosts ubuntu.com
resolvectl status
sudo ss -tulpen
sudo ufw status verbose
sudo nft list ruleset
```

## 12. Key Takeaways

Network security analysis requires more than checking
whether an interface is UP.

The relationship between interfaces, routes, gateways,
DNS resolvers, listening sockets, and firewall policies
must be examined together.

This establishes a foundation for network troubleshooting,
attack-surface analysis, and security monitoring.
