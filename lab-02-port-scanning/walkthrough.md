# Walkthrough: Port Scanning Investigation

## Step 1: Look at the spread of destination ports

In `conn.log.json`, one source IP, `198.51.100.23`, connects to `10.0.0.15` across a wide range of ports in under half a second: 21 (FTP), 22 (SSH), 23 (Telnet), 25 (SMTP), 80 (HTTP), 135, 139 (Windows RPC/NetBIOS), 443 (HTTPS), 445 (SMB), 3306 (MySQL), 3389 (RDP), 8080. That's 12 different ports, each targeted by a separate short connection, all within roughly 480 milliseconds.

Compare this to the last entry in the sample: a single connection from a different source (`203.0.113.77`) to port 443, lasting 12.4 seconds with substantial data transferred in both directions (1,840 bytes sent, 98,230 bytes received). That's what a normal HTTPS session (loading a real webpage) looks like — one port, real duration, real data.

The contrast is the core signal: **one legitimate connection has duration and data. Twelve scan probes have neither.**

## Step 2: Read the `conn_state` field

This field is the single most useful piece of information in a port scan investigation, since it tells you the outcome of each connection attempt without needing packet-level detail:

| `conn_state` | Meaning |
|---|---|
| `S0` | Connection attempt seen, no reply — the port is likely closed/filtered, or the target simply didn't respond (common in stealth/SYN scans) |
| `REJ` | Connection attempt rejected (e.g. TCP RST received) — the port is closed and actively refusing connections |
| `SF` | Normal establishment and termination — a real, completed connection |

In this sample, 11 of the 12 scan connections show `S0` (no response), one shows `REJ` (port 25 actively refused the connection), and only two (ports 80 and 443) show `SF` — meaning those two services actually responded and completed a handshake. This pattern — mostly `S0`/`REJ` with a couple of `SF` — is extremely characteristic of a scan: the attacker is probing broadly and only a few ports happen to be open and responsive.

## Step 3: Check `history` and byte counts

The `history` field summarizes the TCP flag sequence observed. For the scan entries, `history: "S"` means only a SYN was sent with no further exchange — consistent with a SYN/half-open scan (`nmap -sS`) or a full connect scan where the target never replied. For the two `SF` connections, `history` shows a fuller sequence (`ShAdDaFf`), indicating an actual data exchange occurred — worth a closer look on its own, since those two ports responding successfully means the attacker now knows HTTP and SSL/TLS services are running and reachable.

`orig_bytes`/`resp_bytes` being `0` across the scan attempts confirms no actual application data was exchanged — these were connection probes, not real service usage.

## Step 4: Fields to check before concluding

| Field | Why it matters |
|---|---|
| Number of unique destination ports from one source | A handful of ports over time could be normal; a dozen+ in under a second is not |
| Time window | Real users/services don't hit a dozen unrelated ports in half a second — this speed strongly implies automation |
| `conn_state` distribution | Mostly `S0`/`REJ` with minimal `SF` is the signature of a scan; mostly `SF` with real byte counts is normal usage |
| Destination port selection | Does the port list look like a targeted probe of common services (21, 22, 23, 80, 443, 445, 3389, etc. — as here) or a truly random spread? Either can indicate a scan, but a "greatest hits of common services" list suggests a deliberate, informed scan rather than noise |
| Source reputation/context | Is this source IP known for legitimate scanning (e.g. a known vulnerability-scanning service, if your org uses one) or unfamiliar/external with no context? |

## Step 5: Verdict

Given:
- 12 unique destination ports probed by one source in under half a second
- Overwhelmingly `S0`/`REJ` connection outcomes with no data exchanged
- Port selection matching common, high-value services (SSH, RDP, SMB, MySQL, HTTP/S)
- Two ports (80, 443) confirmed open and responsive

This is a confirmed **true positive for port scanning / reconnaissance**. The two responsive ports are the most actionable finding — they tell you exactly what an attacker now knows about `10.0.0.15`: it's running a web server and an SSL/TLS service, both reachable from the internet. Appropriate response: rate-limit or block the source IP at the firewall, and treat ports 80/443 on this host as confirmed externally-visible attack surface worth reviewing (patch status, unnecessary services, WAF coverage).

## Step 6: Building a basic detection rule

A simple, effective heuristic: flag any source IP that connects to more than N unique destination ports on the same target within a short time window (e.g. more than 10 ports in under 5 seconds). This is exactly the kind of "unique count over a time window" logic that Zeek's built-in `scan.zeek` policy script already implements — worth reading Zeek's own scan-detection script as a reference once you're comfortable with this lab, since it solves this same problem at scale across all traffic, not just a curated sample.

## Takeaway

Port scans are one of the more mechanically obvious network anomalies once you know what to look for: they show up as many short, dataless connection attempts across many ports from one source, in a very tight time window. The two fields that do the most work here are `conn_state` (tells you the outcome) and destination port count over time (tells you the pattern) — both available without ever looking at packet payloads.
