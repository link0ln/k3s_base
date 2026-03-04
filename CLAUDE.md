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

**Role-based playbook design** — `playbook.yml` is a thin orchestrator (~45 lines) with `pre_tasks` and a list of roles. Each role handles one concern:

1. **pre_tasks** (in playbook) — SSH keypair generation (local), PermitRootLogin, sshd restart
2. **common** — apt packages, SSH key deployment, WSL detection (`is_wsl` fact), rp_filter workaround
3. **k3s** — config via blockinfile, k3s install, service start, kubeconfig fetch, Helm install
4. **cilium** — Helm install, idempotency check, BPF cleanup on upgrade, system pod recovery
5. **storage** — patches local-path-provisioner configmap to use `/k3s-storage`
6. **nvidia** — GPU detection (lspci + WSL fallback), driver, container toolkit, containerd config, RuntimeClass, device plugin Helm
7. **argocd** — Helm install with password management, deploys ArgoCD Application pointing to `argocd/` dir in this repo
8. **postchecks** — cleanup stale pods, node readiness, deployment/daemonset rollouts, comprehensive pod health check

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
- Handler `Restart sshd` in playbook, `Restart k3s` in k3s role — both globally visible within the play
- `flush_handlers` used in nvidia role to trigger k3s restart mid-play

**Inter-role dependencies:**
- `is_wsl` fact: set in `common`, used in `nvidia`
- `nvidia_gpu_present` fact: set in `nvidia`, used in `postchecks` (with `default(false)`)
- `Restart k3s` handler: defined in `k3s` role, notified from `nvidia` role
- All static config lives in each role's `defaults/main.yml`

## Key Files

- `playbook.yml` — pre_tasks + role list (~45 lines)
- `inventory.ini` — target host definition (IP, credentials, SSH args)
- `ansible.cfg` — points to inventory, sets private key path, enables pipelining
- `argocd/kustomization.yml` — Kustomize root for ArgoCD Application (add manifests here)
- `roles/` — all logic split by concern:
  - `common/` — packages, SSH, WSL detect, rp_filter fix (with `files/cilium-rp-filter-fix.service`)
  - `k3s/` — install, config (`templates/k3s-config.yaml.j2`), kubeconfig, Helm
  - `cilium/` — CNI via Helm, BPF cleanup, system pod recovery
  - `storage/` — local-path-provisioner config
  - `nvidia/` — GPU driver, toolkit, device plugin
  - `argocd/` — Helm install, password management, Application deployment (`templates/argocd-application.yml.j2`)
  - `postchecks/` — health checks and final state display
- `artifacts/` — generated SSH keys, kubeconfig, and ArgoCD admin password (sensitive files are git-ignored)

## Sensitive Files

`artifacts/id_ed25519`, `artifacts/kubeconfig`, and `artifacts/argocd-admin-password` are git-ignored and contain cluster credentials. The public key `artifacts/id_ed25519.pub` and `.gitkeep` are tracked.
