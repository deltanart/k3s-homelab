# K3s Homelab

A fully automated homelab Kubernetes setup running k3s on virtual machines hosted on a single Proxmox node. The stack follows a GitOps approach: Ansible provisions and configures the VMs and installs k3s, then Flux CD takes over to continuously reconcile the desired cluster state from a Git repository.

**Stack:** Proxmox → Ansible → k3s → Flux CD (GitOps)

---

## Repository Structure

```
.
├── ansible/                    # Infrastructure automation
│   ├── inventory/
│   │   ├── hosts.yml           # Node IPs and connection info
│   │   └── group_vars/
│   │       ├── all.yml         # Global variables (customize!)
│   │       ├── masters.yml     # Master VM specs
│   │       ├── workers.yml     # Worker VM specs
│   │       └── vault.yml       # Secrets (encrypt with ansible-vault!)
│   ├── roles/
│   │   ├── proxmox_vm/         # Create VMs on Proxmox
│   │   ├── k3s_common/         # OS preparation (all nodes)
│   │   ├── k3s_server/         # Install control plane
│   │   ├── k3s_agent/          # Join worker nodes
│   │   └── post_install/       # Local tools (kubectl, helm, flux)
│   ├── playbooks/
│   │   ├── 01_provision_vms.yml
│   │   ├── 02_install_k3s.yml
│   │   ├── 03_bootstrap_flux.yml
│   │   └── 04_upgrade_k3s.yml
│   └── requirements.yml
│
└── gitops/                     # GitOps (same repo, watched by Flux)
    ├── clusters/homelab/       # Flux entry point
    │   ├── flux-system/        # Flux components (auto-generated)
    │   ├── infrastructure.yaml
    │   └── apps.yaml
    ├── infrastructure/         # Cluster infrastructure
    │   ├── sealed-secrets/
    │   ├── metallb/
    │   ├── cert-manager/
    │   ├── traefik/
    │   ├── longhorn/
    │   └── monitoring/
    └── apps/                   # Applications
        ├── namespaces/
        └── vaultwarden/        # Example app
```

---

## Setup

### Step 0 – Prerequisites (once)

```bash
# Install Ansible collections
cd ansible
ansible-galaxy collection install -r requirements.yml

# Populate and encrypt the vault file
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml
# Fill in values, then:
ansible-vault encrypt inventory/group_vars/vault.yml

# Create a Cloud-Init template on Proxmox (once, manually — see comment in proxmox_vm/tasks/main.yml)
```

### Step 1 – Provision VMs

```bash
ansible-playbook playbooks/01_provision_vms.yml --ask-vault-pass
```

### Step 2 – Install k3s

```bash
ansible-playbook playbooks/02_install_k3s.yml --ask-vault-pass
# kubeconfig is written to ~/.kube/homelab-k3s.yaml

export KUBECONFIG=~/.kube/homelab-k3s.yaml
kubectl get nodes -o wide
```

### Step 3 – Bootstrap Flux CD

```bash
# GitHub PAT with permissions: repo (full), workflow
export GITHUB_TOKEN=ghp_xxxxxxxxxxxx

ansible-playbook playbooks/03_bootstrap_flux.yml --ask-vault-pass

# Verify Flux status
flux get all -A
```

### Step 4 – Point Flux at This Repository

Flux watches the `gitops/` directory in this repository. After bootstrapping, changes pushed to this directory are reconciled automatically:

```bash
git add gitops/
git commit -m "feat: initial infrastructure stack"
git push

# Flux syncs automatically (within ~1 minute)
flux get kustomizations -A --watch
```

---

## Day-to-Day Commands

```bash
# Cluster status
kubectl get nodes -o wide
kubectl get pods -A

# Flux status
flux get all -A
flux logs --follow

# Trigger manual sync
flux reconcile kustomization flux-system --with-source

# Seal a new secret
kubectl create secret generic my-secret \
  --namespace my-namespace \
  --from-literal=KEY="VALUE" \
  --dry-run=client -o yaml | \
  kubeseal --format yaml > gitops/apps/my-app/sealed-secret.yaml

# Upgrade k3s (rolling update)
ansible-playbook playbooks/04_upgrade_k3s.yml \
  --ask-vault-pass \
  -e k3s_version=v1.31.0+k3s1
```

---

## Configuration Reference

Files marked with `# <-- customize` that need updating before first run:

| File | Variable |
|---|---|
| `ansible/inventory/hosts.yml` | Proxmox host IP and VM IPs |
| `ansible/inventory/group_vars/all.yml` | `proxmox_host`, `vm_nameservers`, `metallb_ip_range`, `flux_github_owner`, `flux_github_repo` |
| `ansible/inventory/group_vars/vault.yml` | All passwords and tokens |
| `gitops/infrastructure/cert-manager/helmrelease.yaml` | Email address for Let's Encrypt |
| `gitops/infrastructure/traefik/helmrelease.yaml` | Domain (`home.local` or custom) |
| `gitops/infrastructure/longhorn/helmrelease.yaml` | Ingress hostname |
| `gitops/infrastructure/monitoring/helmrelease.yaml` | Master IP for etcd scraping |

---

## Local Prerequisites

- Ansible >= 2.15
- Python 3.10+
- SSH key at `~/.ssh/id_ed25519`
- GitHub account with PAT (permissions: `repo`, `workflow`)
