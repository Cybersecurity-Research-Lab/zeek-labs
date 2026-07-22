# Walkthrough: DNS Tunneling Investigation

## Step 1: Look at query volume and timing first

In [`dns.log.json`](/dns.log.json), five queries arrive from the same host (`10.0.0.55`) within about 900 milliseconds, all directed at the same base domain: `datax-relay.example.net`. Compare this to the two other entries in the sample — normal lookups for `www.google.com` and `outlook.office365.com` — which are isolated, single queries, minutes apart, to well-known domains.

High-frequency queries to a single, unfamiliar domain in a tight time window is the first signal worth flagging, before even looking at the query content itself.

## Step 2: Look at the subdomain content

This is where DNS tunneling becomes unmistakable. Each query's subdomain label looks like random noise:

```
a8f3k9d2m1x7q4w8z2v6b9n3.datax-relay.example.net
p3n7j2h9k4d1m8s5w3t6c0y4.datax-relay.example.net
q9x2v5b8n1m4k7j0h3d6s9w2.datax-relay.example.net
```

Legitimate domains almost never look like this. Real subdomains are human-chosen and meaningful (`mail.`, `www.`, `api.`, `cdn.`). A 24-character, seemingly-random alphanumeric string as a subdomain is a strong indicator that the "subdomain" isn't actually a hostname — it's being used to carry encoded data, since DNS query labels support up to 63 characters per label and get through most network filtering unmonitored.

**Two properties to check when judging subdomain suspicion:**
- **Length** — DNS tunneling implementations tend to maximize the encoded payload per query, so labels often approach or hit the 63-character limit (these samples are shorter for readability, but the principle holds)
- **Entropy/randomness** — a human- or CDN-generated subdomain has recognizable patterns (dictionary words, version numbers, geographic codes). A base32/base64-encoded payload looks statistically random — no repeated structure, roughly even distribution of characters

## Step 3: The query type matters too

All five suspicious queries use `qtype_name: "TXT"`. TXT records are a common tunneling vehicle because they're designed to carry arbitrary text data (originally meant for things like domain verification and SPF records) — a much larger payload capacity than a standard A record, which just returns an IP address. A sudden pattern of many TXT queries to one domain, especially from a host that doesn't normally use TXT lookups, is worth flagging on its own.

## Step 4: The answer content is the confirming signal

Look at the `answers` field in the suspicious entries:

```
"answers":["v=exfil;chunk=0142"]
"answers":["v=exfil;chunk=0143"]
"answers":["v=exfil;chunk=0144"]
```

This sample makes the payload obvious for teaching purposes — in a real attack, this would be encoded/obfuscated, not readable plaintext. But the *pattern* is the same regardless of encoding: sequential, chunked responses (`chunk=0142`, `0143`, `0144`...) strongly suggest data being transferred in pieces across multiple queries, which is exactly how DNS tunneling protocols work — legitimate DNS responses don't sequence like this.

## Step 5: Fields to check before concluding

| Field | Why it matters |
|---|---|
| Query frequency to a single domain | A burst of many queries to one unfamiliar domain in a short window is unusual for normal browsing/service behavior |
| Subdomain length and randomness | Long, high-entropy subdomains suggest encoded data rather than a real hostname |
| `qtype_name` | TXT (and sometimes NULL or CNAME) queries are more commonly abused for tunneling than standard A/AAAA lookups |
| Domain reputation/age | Is `datax-relay.example.net` a known, established domain, or something newly registered with no reputation history? |
| Sequential/patterned answer content | Chunked or sequenced-looking response data is a strong confirming signal once you already suspect tunneling from the query pattern |

## Step 6: Verdict

Given:
- Rapid, repeated queries to a single unfamiliar domain
- Long, high-entropy subdomain labels with no human-readable structure
- Heavy use of TXT record type
- Sequential chunk markers in the response data

This is a confirmed **true positive for DNS tunneling / data exfiltration**. Appropriate response in a real environment: block the destination domain at the DNS resolver/firewall level, isolate the source host (`10.0.0.55`) for forensic review, and check other hosts for queries to the same domain to scope how widespread the compromise is.

## Step 7: Building a basic detection rule

A simple heuristic combining what we found: flag any host making more than N queries per minute to the same second-level domain, where the subdomain label length exceeds a threshold (e.g. 30+ characters) and shows high character entropy. This is exactly the kind of multi-signal correlation — volume + structure + entropy — that a single-field check would miss, but that becomes very reliable once combined.

## Takeaway

No single field proves DNS tunneling on its own — volume alone could be a busy legitimate service, and random-looking subdomains alone could (rarely) be a legitimate CDN's routing scheme. It's the **combination** of high query volume, high-entropy long subdomains, an unusual query type, and sequenced response content that turns this from "unusual" into "confirmed."
