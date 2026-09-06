# OpenSCAP Baseline Compliance Scan — Summary

**Profile used:** CIS Ubuntu Linux 22.04 LTS Benchmark for Level 1 - Server
(`xccdf_org.ssgproject.content_profile_cis_level1_server`)

**Date:** 2026-09-06

| Server  | Pass | Fail | Not Applicable | Compliance Score |
|---------|------|------|-----------------|-------------------|
| server1 | 1740 | 817  | 58              | ~68% |
| server2 | 1740 | 817  | 58              | ~68% |
| server3 | 1740 | 817  | 58              | ~68% |

All 3 servers show identical results since they were provisioned from the same
base image with no hardening applied yet — this is the pre-remediation ("before")
baseline required by the task.

Full HTML reports (visual, per-check detail) are in this folder:
- server1-baseline-report.html
- server2-baseline-report.html
- server3-baseline-report.html

Raw XML results (large files, excluded from git) are kept locally at
`openscap/baseline-reports/*.xml` for reference during remediation.
