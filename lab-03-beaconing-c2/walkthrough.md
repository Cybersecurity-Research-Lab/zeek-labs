# Walkthrough: Beaconing / C2 Investigation

## Step 1: Notice this requires looking across multiple entries, not one

Unlike Labs 01 and 02, no single line in `conn.log.json` looks alarming by itself. A single ~300ms SSL connection transferring a few hundred bytes to a real-looking external IP on port 443 is completely unremarkable — this is what makes beaconing genuinely harder to catch than a scan or a tunneling burst. The signal only appears when you group connections by source+destination pair and look at the sequence over time.

## Step 2: Group by source/destination pair and list timestamps

Filtering the log to just `10.0.0.62 → 185.199.108.201`, the timestamps are:

```
1752920000.201
1752920060.198
1752920120.205
1752920180.199
1752920240.203
1752920300.200
1752920360.206
```

Subtracting consecutive timestamps gives the intervals:

```
60.0s → 59.997s → 60.007s → 59.994s → 60.004s → 59.997s → 60.006s
```

Every interval lands within about 10 milliseconds of exactly 60 seconds. Compare this to the two "normal" entries in the sample, to a different destination (`140.82.112.3`) and another (`151.101.65.140`) — each appears only once, at no particular regular interval relative to anything else. That contrast is the core of this lab: **repeated connections at a near-constant interval, to the same destination, is not how humans or typical applications behave.**

## Step 3: Understand "jitter" and why its absence matters

Real, human-driven or even most legitimate automated traffic has **jitter** — natural variation in timing, because it's triggered by unpredictable events (a user clicking something, a queue processing a batch, network conditions varying). Malware beacon implementations, especially simpler or older ones, often use a fixed `sleep()` interval between check-ins, which produces suspiciously *low* jitter — intervals that cluster tightly around one value.

Here, the intervals vary by less than 15 milliseconds out of a 60-second window — a jitter of roughly 0.02%. That's far tighter than almost any organic process would naturally produce. (More sophisticated malware deliberately adds randomized jitter specifically to evade this kind of detection — worth noting as a limitation of interval-regularity detection alone, covered in Step 6.)

## Step 4: Check byte-count consistency

The `orig_bytes` value is exactly `214` in every single one of the seven connections, and `resp_bytes` stays within a narrow band (347-351). Real interactive traffic to the same service typically varies in size depending on what's actually being requested or transferred. Near-identical payload sizes across repeated connections suggests a fixed, templated request — consistent with an automated check-in message rather than varied human or application activity.

## Step 5: Fields to check before concluding

| Field | Why it matters |
|---|---|
| Interval regularity (jitter) between repeated connections to the same destination | Very low jitter around a fixed interval is atypical of organic traffic and typical of scripted/malware beaconing |
| Byte-count consistency (`orig_bytes`/`resp_bytes`) across the sequence | Near-identical sizes suggest a fixed, automated request rather than varied real usage |
| Destination reputation | Is `185.199.108.201` a known, reputable service (note: this specific IP range is used by GitHub Pages in the real world — always check reputation before concluding, since a known-good destination changes the picture significantly) or unfamiliar/unestablished? |
| Connection duration consistency | Similarly tight durations (all ~300ms here) reinforce the same "scripted, not human" pattern |
| Volume of data relative to purpose | Small, fixed check-ins (a few hundred bytes) are consistent with "any new commands?" polling, not real data transfer or legitimate application use |

## Step 6: Verdict — with an important caveat

The pattern here (regular ~60s interval, near-zero jitter, fixed byte counts) is a strong behavioral match for beaconing. **However**, this exact pattern can also be produced by entirely legitimate software: update checkers, license validation pings, monitoring/heartbeat agents, and some SaaS "keep-alive" connections all beacon on fixed intervals too. This is the single most important thing to internalize about this lab: **timing regularity alone is suspicious, not conclusive.**

Before escalating, check:
- What process on the host is generating this traffic? (requires host-level correlation — this is a good example of where a network-only view needs to be paired with something like the Wazuh labs in this lab collection)
- Is the destination IP/domain known and expected for this organization's software stack?
- Has this pattern existed for a long time (more likely legitimate/known) or did it start recently (more suspicious)?

Given the caveat, the honest verdict here is: **suspicious, warranting further investigation** — not an automatic true positive the way Labs 01 and 02 were. Appropriate next step: correlate with host-level process/execution logs for `10.0.0.62` to identify what's actually initiating these connections, and check threat intelligence on the destination IP before deciding whether to block.

## Step 7: Building a basic detection approach

A simple detection heuristic: for each source/destination pair, if there are more than N connections within a time window, compute the standard deviation of the intervals between them. A very low standard deviation relative to the mean interval (low jitter) combined with similar byte counts across connections is a reasonable flag for further review — but, per Step 6, this should route to investigation, not automatic blocking, given the real risk of false positives from legitimate periodic software.

## Takeaway

Beaconing detection is fundamentally different from the previous two labs: it's a **sequence-level** pattern, not something visible in any single log line, and it's also the first lab in this collection where the correct verdict is "investigate further" rather than a confident true/false positive. That distinction — knowing when a pattern is suggestive versus conclusive — is itself one of the most important judgment calls a SOC analyst (or an AI system supporting one) has to make.
