

# Day 09 — Mac Network Monitoring

## 1. Overview

- **Date:** 2026-11-26
- **Environment:** MacBook Air M2
- **OS:** macOS 27.2.1
- **Focus:** Network Interface, TCP Connections, Listening Ports, Routing, DNS
- **Purpose:** Understand host network activity and establish a baseline for security monitoring.

---

## 2. Network Interface Analysis

### Command

```bash
ifconfig
```

### Observed Results

| Interface | Status | Observation |
|---|---|---|
| lo0 | Active | Loopback, 127.0.0.1 |
| en0 | Active | IPv4 192.168.45.90 |
| awdl0 | Active | Apple Wireless Direct Link related interface |
| llw0 | Active | Low Latency WLAN related interface |
| en1, en2 | Inactive | Inactive at the time of observation |
| en3, en4 | Inactive | Inactive at the time of observation |
| bridge0 | Inactive | Bridge interface |
| utun0–utun3 | Present | Tunnel interfaces |

### IPv4 Configuration

- IP Address: `192.168.45.90`
- Netmask: `255.255.255.0`
- Broadcast: `192.168.45.255`
- Network: `192.168.45.0/24`

The `en0` interface was active and had an IPv4 address assigned.

---

## 3. TCP Connection Analysis

### Command

```bash
netstat -anv -p tcp | head -30
```

### Observed Established Connections

| Process | Remote IP | Port | State |
|---|---|---:|---|
| Brave Browser | 44.253.14.244 | 443 | ESTABLISHED |
| Brave Browser | 172.64.155.209 | 443 | ESTABLISHED |
| Brave Browser | 3.171.185.56 | 443 | ESTABLISHED |
| Brave Browser | 104.18.39.21 | 443 | ESTABLISHED |
| Brave Browser | 203.251.233.184 | 443 | ESTABLISHED |
| adprivacyd | 203.251.233.184 | 443 | ESTABLISHED |
| apsd | 17.57.145.152 | 5223 | ESTABLISHED |

Multiple HTTPS connections were observed from Brave Browser.

The `apsd` process also had an established connection to a remote endpoint on port 5223.

An established connection alone does not indicate malicious activity. Process identity, remote endpoint, connection purpose, and traffic characteristics must be reviewed together.

---

## 4. Listening Port Analysis

### Observed Results

| Port | Process | Binding |
|---:|---|---|
| 55920 | astxAgent (PID 1216) | Wildcard |
| 21300 | NWSDaemon (PID 550) | Wildcard |
| 49152 | rapportd (PID 646) | IPv4 / IPv6 |
| 5000 | ControlCenter (PID 661) | IPv4 / IPv6 |
| 7000 | ControlCenter (PID 661) | IPv4 / IPv6 |
| 8021 | launchd (PID 1) | Loopback |
| 22 | launchd (PID 1) | Wildcard |

### Security Interpretation

- Listening ports represent services waiting for connection requests.
- Wildcard bindings indicate that the service is not restricted to one specific local IP address at the binding level.
- Loopback bindings restrict listening to local loopback interfaces.
- A listening port does not automatically mean that it is accessible from the Internet.
- Firewall policies and network configuration must also be considered.

---

## 5. Routing Table Analysis

### Command

```bash
netstat -rn -f inet
```

### Observed Results

| Destination | Gateway | Interface |
|---|---|---|
| default | 192.168.45.1 | en0 |
| 127 | 127.0.0.1 | lo0 |
| 192.168.45 | link#11 | en0 |
| 192.168.45.1 | Local gateway | en0 |
| 192.168.45.90/32 | link#11 | en0 |

The default gateway was `192.168.45.1`.

Traffic to destinations without a more specific routing entry is forwarded through the default route.

---

## 6. DNS Configuration

### Command

```bash
scutil --dns | head -40
```

### Observed Results

| Item | Value |
|---|---|
| DNS Server 1 | 210.220.163.82 |
| DNS Server 2 | 219.250.36.130 |
| Interface | en0 |
| Reachability | Reachable |

Additional mDNS resolvers were observed for local and reverse lookup-related domains.

---

## 7. Errors & Troubleshooting

| Command | Error | Resolution |
|---|---|---|
| `head-40` | Command not found | Corrected to `head -40` |
| `ip adrr` | Unknown object | Corrected to `ip addr` |
| SSH connection | Initial attempt interrupted | Reconnected successfully |

These errors reinforced the importance of command syntax and spacing.

---

## 8. Security Takeaways

1. Identify the active network interface before analyzing traffic.
2. Distinguish `LISTEN` from `ESTABLISHED`.
3. Identify the process responsible for a network connection.
4. Review wildcard bindings carefully.
5. Identify the default gateway and local subnet.
6. Understand the host DNS configuration.
7. Do not classify a connection as malicious based only on its remote IP or port.

---

## 9. Next Steps

- Analyze Ubuntu Server network connections.
- Identify listening services and associated processes.
- Compare host and VM network configuration.
- Investigate TCP connections using packet capture.
- Continue toward Windows network monitoring after VM recovery.

