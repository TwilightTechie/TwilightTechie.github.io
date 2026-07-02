---
title: "Experience"
description: "My professional background and roles."
---

## Member of Technical Staff - 2 @ Nutanix
**November 2025 - Present | Bengaluru, India**

- Building distributed database internals and an AI-driven log analysis and triage platform over cluster telemetry, spanning the storage data path and ML-based observability at scale.
- Designed an anomaly-detection pipeline over high-volume system logs and metrics that flags node failures and performance regressions before customer impact, cutting mean-time-to-detection by ~45% across n-node production clusters.
- Optimized the distributed store layer to sustain higher query and ingestion throughput under load, reducing steady-state service CPU by ~1 core per node on large-scale clusters.
- Reduced read ops/sec against the backing key-value store through query optimization and access-pattern changes, improving overall query latency by ~40%.
- Built a CLI-based cluster-health diagnostic tool that shortened average oncall root-cause time by 2 hour per incident and reduced repeat escalations.
- Partner with AI and platform teams to move anomaly models from offline evaluation into the live distributed data path with acceptable latency budgets.

## Member of Technical Staff - 1 @ Nutanix
**November 2023 - November 2025 | Bengaluru, India**

- Core Data Path (Storage) team, building and debugging the distributed file system that backs Nutanix's hyperconverged storage, writing performance-critical C++ on the read/write I/O path.
- Cut p99 read latency and root-caused recurring production performance regressions oncall.
- Diagnosed and fixed a class of crash-consistency and data-integrity (corruption) bugs surfaced under node-failure and power-loss injection, hardening recovery correctness under high-concurrency conditions.
- Shipped durable fixes, reducing repeat incidents in the storage stack.

## Embedded Software Developer @ MaxLinear
**September 2022 - December 2023 | Bengaluru, India**

- Developed and maintained the Linux kernel device driver for MaxLinear's WiFi 6E chipsets, running low in the stack from mac80211 kernel interfaces down to firmware interaction.
- Implemented and upstreamed driver features for WiFi 6E (6 GHz band operation, 160 MHz channels), writing production C against the Linux mac80211 framework across 4 hardware revisions.
- Debugged complex issues (kernel panics, memory corruption, and race conditions on RX/TX paths) using kernel crash dumps, ftrace, and perf.
- Improved driver throughput and connection stability by tuning buffer management and interrupt handling, contributing to a measured ~15% throughput gain on 6 GHz benchmarks.
- Reduced driver initialization and firmware-load time by ~20% by restructuring the probe sequence, improving cold-boot experience on gateway devices.
- Collaborated across hardware, firmware, and QA teams to bring up and validate WiFi 6E functionality on 3 new board designs.
- Wrote regression tests and spoke at the Bangalore Linux Kernel Meetup on kernel driver work, alongside contributors from IBM LTC and AMD.

## GSoC Student @ Linux Foundation
**June 2023 - August 2023 | Online**

- Built a converter bridging the Linux `perf` tool and the Firefox Profiler. Profiles captured with `perf` can now be explored in the Firefox Profiler's interactive flame-graph UI.
- Designed and implemented a perf-to-Gecko converter that transforms `perf script` output into the Firefox Profiler's Gecko JSON profile format.
- Built the converter using the perf script Python API to walk samples, call chains, and symbol/DSO metadata.
- Handled symbol resolution, kernel vs. user-space frames, and deduplicated string/frame tables to keep output correct and compact.
- Added a lightweight local server to host the converted profile and open it directly in the Firefox Profiler, removing the manual export/import step.
- Worked in the open through the upstream kernel workflow: patch series on the mailing list, review cycles with maintainers. Merged and shipped in Linux 6.6.

## Kernel Bug Fixing Mentee @ Linux Foundation
**March 2023 - May 2023 | Remote**

- Modified driver code to accommodate dt-bindings and defined the necessary bindings for proper device tree integration.
- Improved understanding of networking code, learned how to use semantic patching tools, and developed driver code along with corresponding device tree bindings.
- Expanded knowledge and skills as a Linux kernel developer, preparing for future contributions to the open-source community.

## Project Intern @ Collins Aerospace
**April 2022 - September 2022 | Bengaluru, India**

- Studied and got familiar with the IEEE 802.15.4 Standard.
- Implemented multiple access techniques for wireless communication on MATLAB/Simulink.
