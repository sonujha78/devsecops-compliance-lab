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

## Phase 3: Vulnerable Containers Setup — NEXT

**What's planned:**
- Deploy 2-3 containers using deliberately outdated base images on the servers (for later Trivy scanning)

---
