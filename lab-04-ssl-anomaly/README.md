# Lab 04: SSL/Certificate Anomaly Detection

## Scenario

A host on the monitored network establishes an SSL/TLS connection using a self-signed certificate with a suspicious subject name, issued moments before the connection occurred. Legitimate services almost always use certificates issued by trusted, well-known certificate authorities, with the domain names to prove ownership. A self-signed, freshly-issued certificate on an outbound connection is a common signature of malware C2 infrastructure, since standing up a legitimate CA-signed certificate takes time and leaves a paper trail attackers often prefer to avoid.

This lab uses Zeek's `ssl.log` and `x509.log`, which together let you inspect certificate metadata without needing to decrypt the traffic itself — the certificate exchange happens before encryption begins, so Zeek can observe and log it in the clear.

## Files in this lab

- [`sample-logs/ssl.log.json`](ssl.log.json) — sanitized Zeek SSL log output
- [`sample-logs/x509.log.json`](x509.log.json) — sanitized Zeek certificate log output
- [`walkthrough.md`](walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

Generate a self-signed certificate and serve HTTPS traffic with it (for lab purposes only):

```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 1 -nodes \
  -subj "/CN=update-service-secure.net"
python3 -m http.server 443 --bind 0.0.0.0 &
```

Connect to it and capture the traffic:

```bash
curl -k https://<server-ip>:443/
```

Process with Zeek and inspect both logs:

```bash
zeek -r capture.pcap
cat ssl.log | zeek-cut id.orig_h id.resp_h validation_status
cat x509.log | zeek-cut certificate.subject certificate.issuer certificate.not_valid_before certificate.not_valid_after
```

## What you'll learn

- How Zeek separates connection-level SSL metadata (`ssl.log`) from certificate details (`x509.log`), and how to join them
- What makes a certificate suspicious: self-signed status, very short validity windows, mismatched or generic subject names, recent issuance
- Why `validation_status` matters and what common failure values mean
- How this kind of detection complements (but doesn't replace) full traffic/payload inspection
