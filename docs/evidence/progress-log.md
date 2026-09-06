# Security & Compliance Hardening — Progress & Evidence Log

Project: DevSecOps Compliance Lab
Stack: OpenSCAP + Trivy + Wazuh + Ansible + Docker + Grafana

---

## Phase 1: Environment Setup — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Created 3 KVM-based Ubuntu 22.04 VMs (server1, server2, server3) using cloud-init for automated provisioning
- Each VM: 2 vCPU, 2GB RAM, 10GB disk
- Configured DHCP-assigned IPs on the virbr0 libvirt network:
  - server1: 192.168.122.254
  - server2: 192.168.122.235
  - server3: 192.168.122.138
- Set up Ansible on the base/control machine and verified connectivity to all 3 servers

**Evidence:**

    $ ansible -i ansible/inventory/hosts.ini servers -m ping
    server3 | SUCCESS => { "ping": "pong" }
    server1 | SUCCESS => { "ping": "pong" }
    server2 | SUCCESS => { "ping": "pong" }

    $ virsh list --all
     Id   Name      State
    ----------------------------
     1    server1   running
     2    server2   running
     3    server3   running

**Repo artifacts:** vms/seed1-3/ (cloud-init configs), ansible/inventory/hosts.ini (not committed — contains credentials, excluded via .gitignore)

---

## Phase 2: Docker Installation via Ansible — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Wrote an idempotent Ansible playbook (`ansible/playbooks/install-docker.yml`) to install Docker CE on all 3 servers at scale (no manual per-server setup)
- Playbook adds Docker's official GPG key + apt repo, installs docker-ce/docker-ce-cli/containerd.io, adds the ubuntu user to the docker group, and ensures the service is enabled/running

**Evidence:**

    $ ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-docker.yml
    PLAY RECAP
    server1  : ok=8  changed=6  unreachable=0  failed=0  skipped=0
    server2  : ok=8  changed=6  unreachable=0  failed=0  skipped=0
    server3  : ok=8  changed=6  unreachable=0  failed=0  skipped=0

**Repo artifacts:** ansible/playbooks/install-docker.yml

---

## Phase 3: Vulnerable Containers Deployment — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Wrote Ansible playbook (`ansible/playbooks/deploy-vulnerable-containers.yml`) to deploy intentionally outdated Docker images across all 3 servers using docker-compose
- Deployed containers: nginx:1.14.0, node:10.15.0, python:3.6.9 — chosen for known CVEs, to be scanned with Trivy in Phase 6

**Evidence:**

    $ ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/deploy-vulnerable-containers.yml
    PLAY RECAP
    server1  : ok=5  changed=3  unreachable=0  failed=0  skipped=0
    server2  : ok=5  changed=3  unreachable=0  failed=0  skipped=0
    server3  : ok=5  changed=3  unreachable=0  failed=0  skipped=0

    $ ssh ubuntu@192.168.122.254 "docker ps"
    CONTAINER ID   IMAGE          COMMAND                     STATUS         PORTS                     NAMES
    0a4a1b8c2da4   python:3.6.9   "sleep infinity"            Up 5 minutes                             vuln-python
    349125d6921d   node:10.15.0   "sleep infinity"            Up 5 minutes                             vuln-node
    049da8eaa670   nginx:1.14.0   "nginx -g 'daemon of...'"   Up 5 minutes   0.0.0.0:8081->80/tcp     vuln-nginx

**Repo artifacts:** docker/vulnerable-images/docker-compose.yml, ansible/playbooks/deploy-vulnerable-containers.yml

---

## Phase 4: OpenSCAP Baseline Compliance Audit — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Installed OpenSCAP (`libopenscap8`) and downloaded SCAP Security Guide v0.1.78 content on all 3 servers via Ansible
- Ran a baseline compliance scan against the **CIS Ubuntu Linux 22.04 LTS Benchmark for Level 1 - Server** profile (`xccdf_org.ssgproject.content_profile_cis_level1_server`) on all 3 servers — this is the pre-remediation ("before") evidence
- Fetched HTML/XML reports back to the control machine

**Evidence — Baseline Compliance Score (before remediation):**

| Server  | Pass | Fail | Not Applicable | Compliance Score |
|---------|------|------|-----------------|-------------------|
| server1 | 1740 | 817  | 58              | ~68% |
| server2 | 1740 | 817  | 58              | ~68% |
| server3 | 1740 | 817  | 58              | ~68% |

All 3 servers show identical results since they were provisioned from the same base image with no hardening applied.

**Repo artifacts:**
- ansible/playbooks/install-openscap.yml
- ansible/playbooks/openscap-baseline-scan.yml
- openscap/baseline-reports/SUMMARY.md
- openscap/baseline-reports/server{1,2,3}-baseline-report.html (raw XML results excluded from git — too large, kept locally)

---

## Phase 5: Remediation at Scale via Ansible — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Wrote a single idempotent Ansible playbook (`ansible/playbooks/cis-remediation.yml`) covering the required CIS remediation areas at scale (no manual per-server fixes):
  - SSH hardening (disable root login, disable password auth, disable empty passwords, LoginGraceTime, MaxAuthTries)
  - Firewall (UFW default-deny incoming, SSH explicitly allowed, enabled)
  - Password/account policy (min length 14, complexity rules, max/min/warn age)
  - Disabled unused services where present (avahi-daemon, cups, rpcbind)
  - File permission fixes on /etc/passwd, /etc/shadow, /etc/gshadow, /etc/group + world-writable file audit (none found)
- Verified idempotency by running the playbook twice: first run `changed=15`, second run `changed=1` (only the sshd restart handler) — no false changes or breakage
- Re-ran the OpenSCAP CIS Level 1 Server scan post-remediation to capture "after" evidence

**Evidence — Before vs After Compliance Score:**

| Server  | Score (Before) | Score (After) |
|---------|------------------|------------------|
| server1 | 68.05% | 68.70% |
| server2 | 68.05% | 68.70% |
| server3 | 68.05% | 68.70% |

**Repo artifacts:**
- ansible/playbooks/cis-remediation.yml
- ansible/playbooks/openscap-postremediation-scan.yml
- openscap/post-remediation-reports/SUMMARY.md
- openscap/post-remediation-reports/server{1,2,3}-post-report.html

---

## Phase 6: Vulnerability Scanning with Trivy — COMPLETE

**Date:** 2026-09-06

**What was done:**
- Installed Trivy v0.71.2 on all servers via Ansible
- Scanned all 3 running images (nginx:1.14.0, node:10.15.0, python:3.6.9) for CVEs, broken down by severity
- Rebuilt 2 images on updated, minimal base images (nginx:1.27-alpine, node:20-alpine) and re-scanned to prove vulnerability reduction
- Added a GitHub Actions workflow (`.github/workflows/trivy-scan.yml`) that builds an image and fails the pipeline (`exit-code: 1`) if any CRITICAL severity CVE is found — vulnerable images never reach a registry

**Evidence — CVE counts by severity (before rebuild):**

| Image | Critical | High | Medium | Low |
|-------|----------|------|--------|-----|
| nginx:1.14.0 | 39 | 107 | 65 | 69 |
| node:10.15.0 | 268 | 1295 | 1570 | 584 |
| python:3.6.9 | 208 | 1672 | 2156 | 538 |

**Evidence — CVE counts after rebuilding on updated base images:**

| Image | Critical (Before → After) | High (Before → After) |
|-------|------------------------------|---------------------------|
| nginx (1.14.0 → 1.27-alpine) | 39 → 2 (95% reduction) | 107 → 32 (70% reduction) |
| node (10.15.0 → 20-alpine) | 268 → 1 (99.6% reduction) | 1295 → 23 (98% reduction) |

**Repo artifacts:**
- ansible/playbooks/install-trivy.yml
- ansible/playbooks/trivy-scan.yml
- ansible/playbooks/trivy-rescan-fixed.yml
- docker/fixed-images/Dockerfile.nginx-fixed
- docker/fixed-images/Dockerfile.node-fixed
- .github/workflows/trivy-scan.yml (CI gate on CRITICAL CVEs)
- trivy/reports/*.txt (human-readable scan reports; raw JSON excluded from git — large files, kept locally)

---

**CI Pipeline Gate Evidence:** the GitHub Actions workflow correctly failed a real build when it detected a CRITICAL CVE (CVE-2026-31789, libcrypto3/OpenSSL heap buffer overflow) in the rebuilt nginx-fixed image, even after the earlier vulnerability reduction — proving the gate blocks any image with unresolved Critical vulnerabilities before it could reach a registry. Exit code 1, build correctly rejected.

---

## Phase 7: SIEM Setup & Detection with Wazuh — IN PROGRESS

**Date:** 2026-09-06

**What was done:**
- Increased server1's RAM to 4GB and disk to 49GB to host the Wazuh manager, indexer, and dashboard (single-node all-in-one install)
- Installed Wazuh 4.9.2 manager + indexer + dashboard on server1 using the official Wazuh installation assistant
- Installed Wazuh agent v4.9.2 (version-pinned to match the manager) on server2 and server3 via Ansible, configured to report to the manager at 192.168.122.254
- Verified both agents successfully enrolled and connected (Active: 2, Disconnected: 0 in the Wazuh dashboard)
- Opened required firewall ports on the manager (1515, 1514, 443) via UFW

**Evidence:**

    $ sudo cat /var/ossec/etc/client.keys
    001 server3 any <key>
    002 server2 any <key>

Wazuh dashboard "Agents Summary" widget confirms: Active (2), Disconnected (0).
Last 24 hours alerts already populating (Medium: 316, Low: 290) from default rulesets, confirming log ingestion and rule evaluation are working end-to-end.

**Troubleshooting notes (for reproducibility):** the Wazuh agent's "stable" apt repo installs the latest agent version by default, which is incompatible with an older pinned manager version (manager rejects newer agents with "Agent version must be lower or equal to manager version"). Fixed by pinning the agent package version to match the manager (`wazuh-agent=4.9.2-1`).

**Repo artifacts:**
- ansible/playbooks/install-wazuh-agent.yml
- ansible/inventory/hosts.ini (agent_servers group — not committed, contains credentials)

**Still to do in this phase:**
- Enable File Integrity Monitoring (FIM) on sensitive paths (/etc/passwd, SSH config directory)
- Configure a detection rule for SSH brute-force login attempts
