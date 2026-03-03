# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible-based infrastructure automation that bootstraps a K3s (lightweight Kubernetes) cluster with Cilium CNI and optional NVIDIA GPU support. Targets single-node or multi-node deployments.

## Commands

```bash
# Run the full playbook
ansible-playbook playbook.yml

# Verbose run
ansible-playbook playbook.yml -v

# Syntax check only
ansible-playbook playbook.yml --syntax-check

# Verify cluster after deployment
export KUBECONFIG=/opt/gitrepo/k3s_base/artifacts/kubeconfig
kubectl get nodes
kubectl get pods -A -o wide

# SSH into the target host
ssh -i artifacts/id_ed25519 -o StrictHostKeyChecking=no root@192.168.196.67
```

## Architecture

**Single playbook design** — `playbook.yml` is the sole orchestration file (~550 lines) with this deployment flow:

1. **SSH key generation** (local) — Ed25519 keypair in `artifacts/`
2. **System setup** (remote) — apt packages, root SSH key deployment, PermitRootLogin enabled
3. **K3s install** (remote) — config from `templates/k3s-config.yaml.j2`, then install via get.k3s.io
4. **kubeconfig fetch** (local) — pulled from remote, server address corrected from 127.0.0.1 to real IP
5. **Cilium CNI** (remote) — installed via Helm, replaces both Flannel and kube-proxy (eBPF-based)
6. **System pod recovery** — restarts coredns, metrics-server, local-path-provisioner after CNI swap
7. **Storage config** — patches local-path-provisioner to use `/k3s-storage`
8. **NVIDIA GPU** (conditional) — detects GPU via `lspci`, installs driver + container toolkit + device plugin
9. **Post-checks** — verifies all nodes Ready, all deployments/daemonsets rolled out, all kube-system pods Running with all containers ready (retries up to 10 times)

**Key design decisions:**
- Flannel disabled (`flannel-backend: "none"`) — Cilium is the sole CNI
- kube-proxy disabled — Cilium handles service routing via eBPF
- Traefik disabled — no default ingress controller installed
- Cilium operator set to 1 replica (single-node optimization)
- `blockinfile` used for k3s config — allows other tools to also write to `/etc/rancher/k3s/config.yaml`

**Idempotency patterns used throughout:**
- `creates:` arguments on commands to skip re-runs
- `when:` conditions gating conditional blocks (e.g., NVIDIA tasks only run when GPU detected)
- `changed_when:` / `failed_when:` to accurately report task state
- Two handlers (`Restart sshd`, `Restart k3s`) with `flush_handlers` to apply restarts at specific points in the flow

## Key Files

- `playbook.yml` — all tasks, pre-tasks, and handlers in one file
- `inventory.ini` — target host definition (IP, credentials, SSH args)
- `ansible.cfg` — points to inventory, sets private key path, enables pipelining
- `templates/k3s-config.yaml.j2` — K3s server config (TLS SAN, disabled components)
- `artifacts/` — generated SSH keys and kubeconfig (sensitive files are git-ignored)

## Sensitive Files

`artifacts/id_ed25519` and `artifacts/kubeconfig` are git-ignored and contain cluster credentials. The public key `artifacts/id_ed25519.pub` and `.gitkeep` are tracked.
