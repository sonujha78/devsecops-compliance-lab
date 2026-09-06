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
- Deployed containers: nginx:1.14.0, node:10.15.0, python:3.6.9 — chosen for known CVEs, to be scanned with Trivy in Phase 4

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

## Phase 5: Remediation at Scale via Ansible — NEXT

**What's planned:**
- Identify highest-severity failed CIS checks (SSH hardening, firewall, password policy, unused services, file permissions)
- Write an idempotent Ansible playbook to remediate them across all 3 servers
- Re-run the OpenSCAP scan and record the improved ("after") compliance score
