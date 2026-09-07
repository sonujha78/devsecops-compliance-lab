# DevSecOps Compliance Lab

Security & Compliance Hardening — a production-style DevSecOps pipeline covering CIS
Benchmark compliance, container vulnerability scanning, SIEM deployment, and live
attack detection on a small Linux fleet.

**Stack:** OpenSCAP · Trivy · Wazuh (SIEM/XDR) · Ansible · Docker · Grafana · GitHub Actions

**Full evidence log (before/after data for every phase):** [`docs/evidence/progress-log.md`](docs/evidence/progress-log.md)

---

## Objective

Take a fleet of Linux servers and containers left at **default, insecure settings**,
and:

1. Audit them against the CIS Ubuntu Linux Benchmark (OpenSCAP)
2. Remediate the failed checks **at scale** with Ansible — not by hand
3. Scan all container images for CVEs (Trivy), fix the worst offenders, and gate CI
   builds on Critical vulnerabilities
4. Deploy a SIEM (Wazuh) that detects real attacker behavior — SSH brute-force,
   unauthorized file changes — in near real time
5. Prove detection works with an actual simulated attack, not a hypothetical
6. Visualize the whole security posture on a live Grafana dashboard

Every phase below produced **before/after evidence**, not just a "the script ran"
checkmark.

---

## Architecture

```
                          ┌─────────────────────────┐
                          │   Control / Base Machine │
                          │   (Ansible orchestrator, │
                          │    KVM/libvirt host)     │
                          └────────────┬─────────────┘
                                       │ SSH / Ansible
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
    ┌─────────▼─────────┐    ┌─────────▼─────────┐    ┌─────────▼─────────┐
    │      server1       │    │      server2       │    │      server3       │
    │  192.168.122.254   │    │  192.168.122.235   │    │  192.168.122.138   │
    │  4 vCPU / 4GB RAM  │    │  2 vCPU / 2GB RAM  │    │  2 vCPU / 2GB RAM  │
    │  49GB disk         │    │  20GB disk         │    │  20GB disk         │
    │                     │    │                     │    │                     │
    │  Docker +           │    │  Docker +           │    │  Docker +           │
    │  vulnerable         │    │  vulnerable         │    │  vulnerable         │
    │  containers          │    │  containers          │    │  containers          │
    │  (nginx/node/python) │    │  (nginx/node/python) │    │  (nginx/node/python) │
    │                     │    │                     │    │                     │
    │  OpenSCAP + Trivy    │    │  OpenSCAP + Trivy    │    │  OpenSCAP + Trivy    │
    │                     │    │                     │    │                     │
    │  Wazuh Manager +     │◄───┤  Wazuh Agent         │    │  Wazuh Agent         │
    │  Indexer + Dashboard │◄───┼──────────────────────┼────┤  (FIM + log forward) │
    │  (SIEM core)          │    │  (FIM + log forward) │    │                     │
    │                     │    │                     │    │                     │
    │  Grafana              │    │                     │    │                     │
    │  (security posture     │    │                     │    │                     │
    │  dashboard, reads from │    │                     │    │                     │
    │  Wazuh indexer)         │    │                     │    │                     │
    └─────────────────────┘    └─────────────────────┘    └─────────────────────┘

    GitHub Actions CI ──► builds fixed images ──► Trivy scan ──► FAILS build on
                                                                    Critical CVE
```

All three VMs are provisioned via cloud-init on KVM/libvirt. `server1` doubles as the
central Wazuh manager + Grafana dashboard host since it needed the most resources
anyway (indexer is memory/disk heavy).

---

## Repository Layout

```
├── vms/                          cloud-init seed configs for server1-3
├── ansible/
│   ├── inventory/hosts.ini       (not committed — contains credentials)
│   └── playbooks/                every playbook used across all 9 phases
├── openscap/
│   ├── baseline-reports/         "before" CIS scan (HTML + summary)
│   └── post-remediation-reports/ "after" CIS scan (HTML + summary)
├── docker/
│   ├── vulnerable-images/        docker-compose for outdated images
│   └── fixed-images/             Dockerfiles for the rebuilt, patched images
├── trivy/reports/                Trivy scan output (table format)
├── .github/workflows/
│   └── trivy-scan.yml            CI gate: fails build on Critical CVE
├── wazuh/                        (Wazuh config notes)
├── grafana/                      (Grafana dashboard notes)
└── docs/evidence/
    └── progress-log.md           full phase-by-phase evidence log
```

---

## Step-by-Step: How This Was Built

### Phase 0 — Project & Repo Setup

```bash
mkdir -p ~/security-compliance-hardening
cd ~/security-compliance-hardening
mkdir -p ansible/{roles,inventory,playbooks} openscap/{baseline-reports,post-remediation-reports} \
         trivy/{reports,ci} wazuh grafana/dashboards docs/evidence docker/vulnerable-images

git init
git branch -M main
gh repo create devsecops-compliance-lab --private --source=. --remote=origin
git push -u origin main
```

### Phase 1 — Environment Setup (3 VMs via KVM + cloud-init)

```bash
# KVM / libvirt
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients \
                     bridge-utils virtinst virt-manager cloud-image-utils
sudo systemctl enable --now libvirtd

# Ubuntu 22.04 cloud image
cd vms
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

for i in 1 2 3; do
  cp jammy-server-cloudimg-amd64.img server$i.qcow2
  qemu-img resize server$i.qcow2 10G
  mkdir -p seed$i
  # user-data / meta-data with SSH key + ubuntu user, see vms/seed*/
  cloud-localds seed$i/seed$i.iso seed$i/user-data seed$i/meta-data
done

for i in 1 2 3; do
virt-install --name server$i --memory 2048 --vcpus 2 \
  --disk path=$(pwd)/server$i.qcow2,format=qcow2 \
  --disk path=$(pwd)/seed$i/seed$i.iso,device=cdrom \
  --os-variant ubuntu22.04 --network network=default \
  --graphics none --import --noautoconsole
done

# Ansible on the control machine
sudo apt install -y ansible sshpass
ansible -i ansible/inventory/hosts.ini servers -m ping
```

### Phase 2 — Docker Install (Ansible, all 3 servers)

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-docker.yml
```

### Phase 3 — Deploy Intentionally Vulnerable Containers

`docker/vulnerable-images/docker-compose.yml` deploys `nginx:1.14.0`, `node:10.15.0`,
`python:3.6.9` — old images chosen specifically for known CVEs.

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/deploy-vulnerable-containers.yml
```

### Phase 4 — OpenSCAP Baseline Audit (CIS Ubuntu 22.04 Level 1 Server)

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-openscap.yml
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/openscap-baseline-scan.yml
```

**Result: ~68% pass rate** across all 3 servers (identical, since they share a base
image). Full HTML reports in `openscap/baseline-reports/`.

### Phase 5 — Remediation at Scale (Ansible, idempotent)

`ansible/playbooks/cis-remediation.yml` covers SSH hardening, UFW firewall
(default-deny), password/account policy, disabling unused services, and file
permission fixes on `/etc/passwd`, `/etc/shadow`, `/etc/gshadow`, `/etc/group`.

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/cis-remediation.yml
# Run again to prove idempotency:
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/cis-remediation.yml
# 1st run: changed=15   2nd run: changed=1 (sshd restart handler only)

ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/openscap-postremediation-scan.yml
```

**Result: 68.05% → 68.70%** measurable improvement across all servers.

### Phase 6 — Vulnerability Scanning (Trivy) + CI Gate

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-trivy.yml
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/trivy-scan.yml
```

| Image | Critical | High | Medium | Low |
|---|---|---|---|---|
| nginx:1.14.0 | 39 | 107 | 65 | 69 |
| node:10.15.0 | 268 | 1295 | 1570 | 584 |
| python:3.6.9 | 208 | 1672 | 2156 | 538 |

Rebuilt `nginx` and `node` on minimal Alpine base images
(`docker/fixed-images/Dockerfile.*`), then re-scanned:

| Image | Critical (Before → After) | High (Before → After) |
|---|---|---|
| nginx (1.14.0 → 1.27-alpine) | 39 → 2 (95% ↓) | 107 → 32 (70% ↓) |
| node (10.15.0 → 20-alpine) | 268 → 1 (99.6% ↓) | 1295 → 23 (98% ↓) |

`.github/workflows/trivy-scan.yml` builds the image and fails the pipeline
(`exit-code: 1`) on any CRITICAL CVE. **This gate has fired for real** on this repo —
CVE-2026-31789 (OpenSSL/libcrypto3 heap buffer overflow) in the rebuilt nginx image,
correctly blocking the build.

### Phase 7 — SIEM Setup (Wazuh)

`server1` was upgraded to 4GB RAM / 49GB disk to host the Wazuh manager, indexer, and
dashboard (single-node install):

```bash
# on server1
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Agents (version-pinned to `4.9.2-1` to match the manager) deployed to server2/server3:

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-wazuh-agent.yml
```

File Integrity Monitoring enabled via a shared agent group config on
`/etc/passwd`, `/etc/shadow`, `/etc/ssh`, `/etc/sudoers` (real-time). The default Wazuh
ruleset already ships SSH brute-force detection rules (5710/5712/5760, MITRE
ATT&CK T1110) — no custom rule authoring was needed.

### Phase 8 — Attack Simulation

**SSH brute-force (Hydra):**

```bash
hydra -l ubuntu -P /tmp/passwords.txt ssh://192.168.122.138 -t 4 -f
```

→ Wazuh Rule 5760 fired 5 times (one per attempt), MITRE categories "Password
Guessing" / "SSH" populated in the dashboard within seconds.

**File integrity tampering:**

```bash
ssh ubuntu@192.168.122.235 "echo '# unauthorized test change' | sudo tee -a /etc/ssh/sshd_config"
```

→ Wazuh Rule 550 ("Integrity checksum changed") fired immediately, with the old
SHA1 checksum logged as evidence.

Both test changes were reverted right after capturing evidence.

### Phase 9 — Security Posture Dashboard (Grafana)

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/playbooks/install-grafana.yml
```

Connected to the Wazuh indexer (OpenSearch) as a data source and built a 3-panel
dashboard: alerts over time, alert severity distribution, and a Trivy vulnerability
summary panel. Live at `http://<server1-ip>:3000`.

---

## Challenges & Troubleshooting

This project hit a fair number of real environment issues — documenting them here
since the fixes aren't always obvious from official docs.

**Package name drift across Ubuntu releases.**
`openscap-scanner` doesn't exist in Ubuntu 22.04's repos (it's the Ubuntu 24.04+ /
RHEL name). The actual package that ships the `oscap` CLI on 22.04 is `libopenscap8`.
Verified with `apt-cache search`, not the docs.

**SCAP Security Guide content isn't in apt at all.**
Ubuntu's repos don't carry the full CIS benchmark content packages. Downloaded the
SCAP Security Guide release directly from the
[ComplianceAsCode/content](https://github.com/ComplianceAsCode/content) GitHub
releases via Ansible's `get_url` + `unarchive`.

**Trivy scan timeouts on large image layers.**
The default 5-minute Trivy timeout wasn't enough to analyze the `nginx` image's
layers on constrained VM disk I/O. Fixed with `--timeout 20m`.

**Repeated "no space left on device."**
This came up three separate times — once for Trivy's vulnerability DB, twice during
Wazuh installation (Indexer + Manager + Filebeat + Dashboard together need several
GB). The fix each time was the same pattern: `virsh shutdown` → `qemu-img resize
+NG` → `virsh start` → `growpart` / `resize2fs` (which cloud-init actually runs
automatically on next boot in most cases — the manual `growpart` often reported
`NOCHANGE` because it had already happened). Final sizing: server1 grew from 10GB →
20GB → 29GB → 49GB across the project as Wazuh's footprint became clear; the other
two went from 10GB → 20GB.

**Wazuh agent version mismatch.**
The Wazuh apt "stable" repo always installs the *latest* agent version, but the
manager here was pinned to 4.9.2. A newer agent talking to an older manager gets
rejected outright: `Agent version must be lower or equal to manager version`. Fixed
by pinning the install: `apt: name: wazuh-agent=4.9.2-1` (note: the corresponding
`systemd` task must reference the *unversioned* service name — a version suffix
there breaks the module).

**Installing the Wazuh agent on the manager node itself broke the manager.**
Running the generic "install agent on all servers" playbook against `server1`
(which is also the Wazuh manager) overwrote shared `/var/ossec` files and killed
`wazuh-remoted`/`wazuh-analysisd`/`wazuh-authd`. Fixed by splitting the Ansible
inventory into a `servers` group (all 3, for Docker/OpenSCAP/Trivy) and a narrower
`agent_servers` group (server2 + server3 only) for the Wazuh agent playbook.

**A partial/interrupted Wazuh install left the system in an inconsistent, half-removed
state** (`dpkg` pre/post-removal scripts themselves failing with exit code 127,
because they called a binary — `wazuh-keystore` — that a previous failed install had
already deleted). Recovered by force-purging with the offending maintainer scripts
removed (`rm /var/lib/dpkg/info/wazuh-manager.prerm` /`.postrm`) before retrying
`dpkg --purge`, then reinstalling clean.

**Grafana plugin signature mismatch on manual install.**
Manually downloading and unzipping the OpenSearch datasource plugin (needed because
Grafana's plugin catalog wasn't reachable from this environment) left a
`MANIFEST.txt` that Grafana's signature verifier flagged as "Modified signature" —
a *stricter* check than "unsigned," so `allow_loading_unsigned_plugins` in
`grafana.ini` alone wasn't enough. Deleting the stale `MANIFEST.txt` made Grafana
treat it as a plain unsigned plugin, which the config setting did correctly allow.

**Ansible `fetch` module and relative paths.**
Several `fetch` tasks used relative destination paths (`dest: "openscap/..."`).
Ansible resolves these relative to the **current working directory the playbook was
launched from**, not the playbook's own location — running from
`ansible/playbooks/` instead of the project root silently created a parallel
`ansible/playbooks/openscap/...` tree instead of the intended `openscap/...` at the
repo root. Always run `ansible-playbook` from the project root.

**CI gate legitimately failing is not a bug.**
The GitHub Actions Trivy workflow shows a red ✗ on this repo's Actions tab, and it's
expected: the rebuilt `nginx-fixed` image still carries 2 Critical CVEs (down from
39), and the gate is configured to fail on *any* Critical CVE. That's the gate doing
its job — proof the CI control is real and not just theater.

---

## Credentials & Secrets

Nothing sensitive is committed. `ansible/inventory/hosts.ini` (SSH passwords) and all
raw scan output (`*.xml`, `*.json` — large and non-human-friendly) are excluded via
`.gitignore`. Only human-readable HTML/table reports and summaries are versioned.

**Lab-only default credentials used throughout (`ubuntu` / `ubuntu123`) are for this
disposable KVM lab environment only and should never be reused anywhere real.**

---

## Reproducing This

1. Provision 3 Ubuntu 22.04 VMs (KVM shown above, but any hypervisor works) with SSH
   key access for an Ansible control node.
2. Fill in `ansible/inventory/hosts.ini` (see the group structure referenced in the
   playbooks — `servers` for all three, `agent_servers` for the two non-manager
   nodes).
3. Run the playbooks in `ansible/playbooks/` in the order described above.
4. Everything else (Wazuh manager/dashboard install, Grafana data source wiring) was
   done directly on `server1` per the commands in Phase 7 and Phase 9 — these aren't
   Ansible-driven since they're one-time interactive installers.
