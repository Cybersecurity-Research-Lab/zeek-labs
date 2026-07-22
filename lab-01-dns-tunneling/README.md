# Lab 01: DNS Tunneling Detection

## Scenario

A host on the monitored network is generating an unusually high volume of DNS queries to a single external domain, with unusually long, encoded-looking subdomain labels. This is a classic pattern for **DNS tunneling** — a technique attackers use to exfiltrate data or maintain command-and-control communication by encoding information inside DNS queries, since DNS traffic is rarely blocked or closely inspected by default.

DNS tunneling is a good first network anomaly lab because, unlike a lot of network attacks, it's detectable almost entirely from `dns.log` alone, with no need for full packet payload inspection — the query patterns themselves are the signal.

## Files in this lab

- [`sample-logs/dns.log.json`](/dns.log.json) — sanitized Zeek DNS log output for this scenario
- [`walkthrough.md`](/walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

You can simulate basic DNS tunneling traffic patterns (for lab purposes only, on an isolated network) using a tool like `dnscat2` or `iodine`, or more simply, by generating queries manually to observe the log pattern:

```bash
for i in {1..30}; do
  dig $(head -c 40 /dev/urandom | base64 | tr -dc 'a-z0-9' | head -c 40).tunnel-test.example.com
done
```

Then process the resulting traffic with Zeek:

```bash
zeek -r capture.pcap
cat dns.log | zeek-cut id.orig_h query qtype_name
```

You should see many queries to the same base domain, each with a long, random-looking subdomain.

## What you'll learn

- How to read Zeek's `dns.log` fields relevant to anomaly detection
- What makes a DNS query pattern suspicious: query volume, subdomain length/entropy, query type distribution, and destination domain diversity (or lack thereof)
- How to distinguish DNS tunneling from legitimate high-DNS-volume services (e.g. CDNs, ad networks)
- A basic detection approach: entropy/length thresholds combined with query frequency
