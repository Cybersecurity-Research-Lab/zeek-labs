# Zeek Labs

Hands-on network traffic anomaly detection labs built on [Zeek](https://zeek.org/) (formerly Bro), an open-source network security monitoring tool that turns raw traffic into rich, structured logs (`conn.log`, `dns.log`, `http.log`, `ssl.log`, etc.).

Where [wazuh-labs](../wazuh-labs) focuses on host-based telemetry (auth logs, file integrity, process execution), this repo focuses on the network layer — spotting anomalies in traffic patterns that host-based tools can't see.

## What's inside

Each lab presents a network-based attack or anomaly scenario, the relevant Zeek log output, and a full investigation walkthrough — how to read Zeek's structured logs, which fields matter, and how to distinguish an anomaly from normal (if unusual-looking) traffic.

| Lab | Scenario | Focus |
|---|---|---|
| [lab-01-dns-tunneling](/lab-01-dns-tunneling) | Data exfiltration via DNS queries | `dns.log` analysis |
| [lab-02-port-scanning](/lab-02-port-scanning) | Network reconnaissance / port scan | `conn.log` analysis |
| [lab-03-beaconing-c2](/lab-03-beaconing-c2) | Periodic command-and-control callback traffic | `conn.log` timing analysis |
| [lab-04-ssl-anomalies](/lab-04-ssl-anomalies) | Suspicious/self-signed certificate usage | `ssl.log` / `x509.log` analysis |

*(Labs are added incrementally — check back for updates.)*

## Lab format

Every lab follows the same structure:

1. **Scenario** — what's happening, in plain language
2. **Raw logs** — sanitized but realistic Zeek log output (TSV/JSON)
3. **Investigation walkthrough** — step-by-step analysis of the relevant fields
4. **Verdict** — anomalous or benign, and why
5. **Detection logic** — how this pattern could be flagged automatically (Zeek script logic or a simple detection rule)

## Getting started

See [00-setup/minimal-setup-for-labs.md](minimal-setup-for-labs.md) for a lightweight Zeek setup — enough to capture traffic and generate logs, not a full production sensor deployment.

## Why network-layer detection matters

Host-based tools (like Wazuh) see what happens *on* a machine. Zeek sees what happens *between* machines — which matters because several attack techniques are far more visible in network traffic than in host logs: data exfiltration, command-and-control beaconing, lateral movement, and reconnaissance scanning all leave a stronger signature at the network layer than on any single host's logs.

## About

Part of the [Research Cybersecurity Lab](../) — a collection of hands-on labs and research projects in cybersecurity and AI-driven security tooling.

## Contributing

Suggestions for new lab scenarios are welcome. Open an issue or PR.
