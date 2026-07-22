# Lab 03: Beaconing / Command-and-Control Detection

## Scenario

A host on the monitored network repeatedly connects to the same external IP at strikingly regular time intervals, each connection short and transferring a small, similar amount of data. This is the classic signature of **beaconing**: malware "phoning home" to a command-and-control (C2) server at a fixed interval to check for new instructions, rather than communicating in the organic, irregular pattern of human-driven network activity.

Beaconing is one of the harder network anomalies to catch by looking at any single connection in isolation. No individual connection here looks unusual on its own. The pattern only becomes visible when you look across many connections over time — this lab is as much about learning to think in terms of *sequences* of events as it is about the specific detection technique.

## Files in this lab

- [`sample-logs/conn.log.json`](conn.log.json) — sanitized Zeek connection log output for this scenario, spanning multiple connections over time
- [`walkthrough.md`](walkthrough.md) — full investigation, field by field

## How to reproduce this yourself

Simulate simple periodic beaconing traffic with a basic script (for lab purposes only, on an isolated network):

```bash
while true; do
  curl -s -o /dev/null "http://<test-c2-ip>/checkin?id=host01"
  sleep 60
done
```

Capture over a period of at least 10-15 minutes to get enough data points, then process with Zeek:

```bash
zeek -r capture.pcap
cat conn.log | zeek-cut ts id.orig_h id.resp_h id.resp_p duration orig_bytes resp_bytes
```

Plot or manually inspect the timestamps of connections to the same destination — look for a consistent interval between them.

## What you'll learn

- How to detect a pattern that spans many log entries, not just one
- How to measure interval regularity (jitter) between repeated connections
- Why byte-count consistency across connections is a useful secondary signal
- How to distinguish beaconing from legitimate periodic traffic (e.g. software update checks, heartbeat/keepalive traffic, monitoring agents)
