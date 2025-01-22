
---
## Host and Port Scanning Notes/Guide

After confirming that a target host is alive, the next step is to gather a more detailed picture of its exposed services and vulnerabilities. This stage involves:

- Identifying open ports and their associated services.
- Determining service versions.
- Extracting information provided by services.
- Identifying the target operating system.

### Port States:

Before diving into specific port scanning techniques, it's crucial to understand the different states a scanned port can have:

- **Open:** The port is actively accepting connections. This could involve TCP connections, UDP datagrams, or SCTP associations.
- **Closed:** For TCP, the port is accessible but not accepting connections, indicated by an RST flag in the response packet. Closed ports can be used to verify if a target is alive.
- **Filtered:** Nmap can't definitively determine whether the port is open or closed due to no response or an error code from the target. This usually indicates a firewall or packet filter is blocking the probe.
- **Unfiltered:** This state appears only during TCP ACK scans. It means the port is reachable but its open/closed status is uncertain.
- **Open|Filtered:** Nmap assigns this state when no response is received, suggesting a firewall or packet filter might be protecting the port.
- **Closed|Filtered:** This state is exclusive to IP ID idle scans and indicates the inability to determine if the port is closed or filtered.

### Discovering Open TCP Ports:

Nmap provides various methods for scanning TCP ports. By default, when running as root, Nmap utilizes the SYN scan (`-sS`) to scan the top 1000 TCP ports. If not running as root, the TCP connect scan (`-sT`) is used by default.

Several options exist for specifying the ports to scan:

- Individual ports: `-p 22,25,80,139,445`
- Port range: `-p 22-445`
- Top ports: `--top-ports=10` (scans the top 10 ports defined as most frequent in Nmap's database)
- All ports: `-p-`
- Fast scan (top 100 ports): `-F`

#### Scanning Top 10 TCP Ports:

The following command scans the top 10 most frequent TCP ports on the target 10.129.2.28:

```
sudo nmap 10.129.2.28 --top-ports=10
```

- `--top-ports=10`: Scans the top 10 ports from Nmap's database.

Analyzing the packet trace for closed ports (e.g., port 21) reveals an RST flag in the response, confirming the closed state. To gain a clearer understanding of the SYN scan, disable ICMP echo requests (`-Pn`), DNS resolution (`-n`), and ARP ping (`--disable-arp-ping`).

#### Packet Trace Analysis (SYN Scan):

This command performs a SYN scan on port 21 with packet tracing enabled:

```
sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping
```

- `-p 21`: Specifies the target port.
- `--packet-trace`: Enables packet tracing.
- `-Pn`: Disables ICMP echo requests.
- `-n`: Disables DNS resolution.
- `--disable-arp-ping`: Disables ARP ping.

**The SENT line shows a TCP packet with the SYN flag being sent to the target. The RCVD line shows the target responding with a TCP packet containing both RST and ACK flags. This indicates the port is closed.**

#### Connect Scan (-sT):

The TCP Connect Scan completes the three-way TCP handshake to determine a port's state. It's highly accurate but less stealthy compared to other methods like the SYN scan.

**Advantages:**

- **High Accuracy:** Completes the three-way handshake for definitive port state.
- **Bypass Personal Firewalls:** Effective against firewalls that drop incoming but allow outgoing packets.
- **Service Interaction:** Clean interaction with services, minimizing disruption.

**Disadvantages:**

- **Less Stealthy:** Establishes a full connection, potentially triggering logs and IDS/IPS alerts.
- **Slower:** Requires waiting for responses after each packet.

**Use Cases:**

- When accuracy is paramount.
- Mapping networks without causing significant service disruptions.
- Situations where stealth is not a primary concern.

#### Connect Scan Example:

The following command performs a Connect scan on port 443 with additional options for analysis:

```
sudo nmap 10.129.2.28 -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT
```

- `-p 443`: Target port.
- `--packet-trace`: Enables packet tracing.
- `--disable-arp-ping`: Disables ARP ping.
- `-Pn`: Disables ICMP echo requests.
- `-n`: Disables DNS resolution.
- `--reason`: Provides the reason for a port's state.
- `-sT`: Specifies the TCP Connect scan.

### Filtered Ports:

When a port is marked as filtered, it usually signifies a firewall blocking access. Firewalls can either drop packets silently or reject them with an RST flag, ICMP error codes, or no response at all.

#### Dropped Packets:

Nmap receives no response when packets are dropped, leading to a filtered state. The `--max-retries` option (default value 10) controls how many times Nmap resends packets to a port before giving up.

#### Rejected Packets:

Rejected packets provide more information, as they often include RST flags or ICMP error codes indicating the reason for rejection. Analyzing these responses can provide insights into firewall rules.

#### TCP ACK Scan (-sA):

This scan method sends TCP packets with only the ACK flag set. It's **often more effective at bypassing firewalls compared to SYN or Connect scans**. This is because firewalls often struggle to distinguish ACK packets from legitimate ongoing connections.

#### TCP ACK Scan Example:

The following commands compare SYN and ACK scans on ports 21, 22, and 25:

```
# SYN Scan
sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace

# ACK Scan
sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

- `-p 21,22,25`: Target ports.
- `-sS`: Specifies the SYN scan.
- `-sA`: Specifies the ACK scan.
- `-Pn`: Disables ICMP echo requests.
- `-n`: Disables DNS resolution.
- `--disable-arp-ping`: Disables ARP ping.
- `--packet-trace`: Enables packet tracing.

**By observing the RCVD packets and their flags, you can analyze how the firewall handles each scan type.** The SYN scan will likely result in SYN-ACK responses for open ports, while the ACK scan might return RST flags for both open and closed ports, depending on the firewall configuration.

**For dropped packets, no response is received, indicating a more restrictive firewall rule.**

### Discovering Open UDP Ports:

UDP scans (`-sU`) are generally slower and less reliable than TCP scans. This is because UDP is a connectionless protocol, meaning no handshake occurs, and responses are not guaranteed.

#### UDP Scan Behavior:

- Nmap sends empty datagrams to target UDP ports.
- Open UDP ports might respond with application-specific data if configured to do so.
- Closed ports might respond with ICMP "port unreachable" messages (type 3, code 3).
- If no response is received, the port is marked as `open|filtered`.

#### UDP Scan Example:

This command performs a UDP scan on the top 100 ports of the target:

```
sudo nmap 10.129.2.28 -F -sU
```

- `-F`: Scans the top 100 ports.
- `-sU`: Specifies the UDP scan.

### Key Considerations for UDP Scans:

- **Slowness:** UDP scans can be significantly slower than TCP scans.
- **Reliability:** Responses are not guaranteed, making results less definitive.
- **Open|Filtered:** A lack of response leads to an `open|filtered` state, requiring further investigation.

### Version Scan (-sV):

The version scan is essential for identifying the specific services and versions running on open ports. This information is crucial for vulnerability assessment, as different versions of services may have different vulnerabilities.

#### Version Scan Functionality:

- Nmap examines service banners (responses to connection attempts) to identify service types and versions.
- If banner analysis fails, Nmap uses a signature-based matching system, which can increase scan duration.

#### Version Scan Example:

This command performs a version scan on port 445 with additional options for analysis:

```
sudo nmap 10.129.2.28 -Pn -n --disable-arp-ping --packet-trace -p 445 --reason -sV
```

- `-Pn`: Disables ICMP echo requests.
- `-n`: Disables DNS resolution.
- `--disable-arp-ping`: Disables ARP ping.
- `--packet-trace`: Enables packet tracing.
- `-p 445`: Target port.
- `--reason`: Shows the reason for a port's state.
- `-sV`: Specifies the version scan.

### Important Resource:

For further information on Nmap's port scanning techniques, refer to the documentation:

```
https://nmap.org/book/man-port-scanning-techniques.html
```

---
