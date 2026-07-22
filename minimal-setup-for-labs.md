# Minimal Setup for Zeek Labs

This is *not* a production sensor deployment guide. It's the smallest setup that lets you capture traffic and generate real Zeek logs to follow along with the labs in this repo. For production deployment, see the [official Zeek documentation](https://docs.zeek.org/).

## What you need

- A Linux VM (Zeek runs natively on Linux; a VM keeps this isolated from your main machine)
- Either a live network interface to monitor, or a `.pcap` file to analyze offline (recommended for labs — repeatable and doesn't require live traffic)

## Installing Zeek

On Debian/Ubuntu:

```bash
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list
curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null
sudo apt update
sudo apt install zeek -y
```

Add Zeek to your PATH:

```bash
export PATH=/opt/zeek/bin:$PATH
echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
```

Confirm it installed correctly:

```bash
zeek --version
```

## Option A: Analyze a pcap file (recommended for labs)

This is the easiest way to reproduce labs consistently — no live traffic needed, no risk of capturing something unintended.

```bash
mkdir zeek-lab-output && cd zeek-lab-output
zeek -r /path/to/capture.pcap
```

This generates a set of `.log` files in the current directory (`conn.log`, `dns.log`, `http.log`, etc.) depending on what protocols appear in the pcap.

Sample pcaps for practicing are available from sources like [Malware-Traffic-Analysis.net](https://www.malware-traffic-analysis.net/) (widely used for exactly this kind of training — read their site's usage guidelines before downloading).

## Option B: Live monitoring

To monitor a live interface instead:

```bash
sudo zeek -i eth0
```

Replace `eth0` with your actual interface name (check with `ip a`). Requires root or appropriate capture permissions.

## Reading Zeek logs

Zeek logs are tab-separated by default, with a header describing each column:

```bash
cat conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto duration
```

`zeek-cut` is a helper utility (included with Zeek) for extracting specific fields without manually parsing the TSV header — much easier than `awk`/`cut` for these files.

For JSON-formatted logs (often easier to work with programmatically, and what this repo's sample files use), add this to `local.zeek` before running Zeek:

```
@load policy/tuning/json-logs.zeek
```

## Confirming logs are generating correctly

After running Zeek against a pcap or live interface, you should see `conn.log` at minimum (it logs every connection Zeek observes). If you see traffic-specific logs like `dns.log` or `http.log`, that confirms Zeek is parsing those protocols correctly.

```bash
ls *.log
```

If you only see `conn.log` and expected other logs are missing, check that the pcap/traffic actually contains that protocol — Zeek only generates a log file for protocols it actually observes.

## Notes

- This setup is for learning purposes only. If capturing live traffic, only do so on networks/systems you have explicit permission to monitor.
- Working from pcap files is strongly recommended for these labs — it makes results reproducible and avoids any ambiguity about consent/scope.
