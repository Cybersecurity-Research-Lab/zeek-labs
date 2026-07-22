# Lab 02: Port Scanning Detection

## Scenario

A single external host rapidly attempts connections to many different ports on a monitored server in a short time window. This is classic network reconnaissance: an attacker or automated tool probing for open services before attempting exploitation. Unlike the DNS tunneling lab, this pattern lives entirely in `conn.log`, Zeek's connection summary log, which doesn't require any protocol-specific parsing to investigate.

Port scans are also a good lab for learning to separate genuine threats from harmless noise. The internet is full of background scanning traffic (research crawlers, vulnerability scanners run by security companies, misconfigured devices), so volume alone isn't always enough to justify escalation.

## Files in this lab

- [`sample-logs/conn.log.json`](conn.log.json) — sanitized Zeek connection log output for this scenario
- [`walkthrough.md`](walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

Using nmap against a test host you own or have explicit permission to scan (never scan systems you don't control):

```bash
nmap -sT -p 1-100 <target-ip>
```

Capture the traffic and process with Zeek:

```bash
zeek -r capture.pcap
cat conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto conn_state duration
```

You should see many short-lived connection attempts from the scanning host to a wide range of destination ports on the target.

## What you'll learn

- How to read Zeek's conn.log fields for connection-level analysis
- What conn_state codes mean (e.g. S0, REJ, SF) and why they matter for identifying scan behavior
- How to distinguish a real reconnaissance scan from routine background internet noise
- A basic detection approach: unique destination port count per source, within a time window
