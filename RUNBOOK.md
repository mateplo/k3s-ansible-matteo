# Runbook — déployer un cluster k3s + plateforme GitOps « de 0 »

Procédure de bout en bout : de **machines Linux nues** à un **cluster k3s** avec la
**couche plateforme GitOps** ([classlab-infra](https://github.com/mateplo/classlab-infra) :
ArgoCD + ApplicationSet infrastructure) opérationnelle.

Cible de ce runbook : **serveurs / VMs créés manuellement** (VirtualBox, Proxmox,
bare-metal, cloud…). Ansible prend le relais dès que les machines sont joignables en SSH.

## Vue d'ensemble

```
[0] Machines Linux (toi)           →  VMs / serveurs, IP statiques, SSH
[1] Control node (une fois)        →  WSL/Linux + Ansible + collections + clés SSH
[2] site.yml                       →  installe k3s (servers + agents), rapatrie le kubeconfig
[3] classlab-infra.yml             →  clone classlab-infra + bootstrap.sh → ArgoCD
[4] ArgoCD (auto)                  →  déploie toute la plateforme depuis Git
[5] Apps métier (repos séparés)    →  leurs propres Applications ArgoCD
```

Les étapes [2]→[5] sont automatisées. Les gestes manuels restants : créer les machines [0]
et le setup ponctuel du control node [1].

---

## Étape 0 — Préparer les machines

| Élément | Recommandation |
|---|---|
| Nombre | mini **1 server + 1 agent** ; HA = **3 servers** (nombre impair) |
| OS | Debian 12 ou Ubuntu 22.04 / 24.04 |
| Ressources | **4 GB RAM mini / nœud** (la plateforme embarque Prometheus + Grafana + Loki + CNPG), 2 vCPU |
| Réseau | IP **statiques**, même réseau privé (ex. `192.168.56.0/24`), accès internet sortant |
| À désactiver | **swap** (`swapoff -a` + `/etc/fstab`) et pare-feu (ou ouvrir les ports k3s) |
| Accès | un user avec `sudo`, SSH activé |

---

## Étape 1 — Control node (une seule fois)

> Ansible ne tourne **pas** nativement sous Windows → utilise **WSL** ou une VM Linux.

```bash
sudo apt update && sudo apt install -y ansible git
ansible --version                       # ansible-core 2.15+ requis

# kubectl sur le control node (REQUIS pour rapatrier le kubeconfig, cf. étape 4)
sudo snap install kubectl --classic     # ou apt / curl, au choix

cd k3s-ansible-matteo
ansible-galaxy collection install -r collections/requirements.yml
```

---

## Étape 2 — Clés SSH + sudo (une fois par nœud)

« SSH sans mot de passe » = **authentification par clé**. La clé **privée** reste sur le
control node, la clé **publique** est déposée sur chaque nœud.

```bash
ssh-keygen -t ed25519 -C ansible-classlab        # si pas déjà une clé (~/.ssh/id_ed25519)

ssh-copy-id ton_user@<IP_du_nœud>                # dépose la clé publique (demande le mdp 1 fois)
ssh ton_user@<IP_du_nœud> hostname               # test : doit passer SANS mot de passe

# sudo sans mot de passe (ou utiliser --ask-become-pass au lancement) :
ssh ton_user@<IP_du_nœud> \
  'echo "ton_user ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible && sudo chmod 440 /etc/sudoers.d/ansible'
```

---

## Étape 3 — Inventaire

```bash
cp inventory-sample.yml inventory.yml
```

```yaml
k3s_cluster:
  children:
    server:
      hosts:
        192.168.56.11:        # control-plane
    agent:
      hosts:
        192.168.56.12:        # worker
  vars:
    ansible_user: ton_user
    k3s_version: v1.31.12+k3s1
    # token : openssl rand -base64 64   (ou omis → généré automatiquement)
    token: "<secret>"
    api_endpoint: "{{ hostvars[groups['server'][0]]['ansible_host'] | default(groups['server'][0]) }}"
    # Si la clé n'est pas ~/.ssh/id_ed25519 :
    # ansible_ssh_private_key_file: ~/.ssh/id_ed25519
    # REQUIS pour classlab-infra : MetalLB (GitOps) remplace le LB intégré de k3s
    server_config_yaml: |
      disable:
        - servicelb
```

Variables optionnelles utiles : `extra_server_args`, `use_external_database`,
`server_config_yaml`, `registries_config_yaml`, `airgap_dir`… (voir `inventory-sample.yml`).

---

## Étape 4 — Installer k3s

```bash
ansible-playbook playbooks/site.yml -i inventory.yml --syntax-check
ansible all -i inventory.yml -m ping                       # SSH OK ?
ansible-playbook playbooks/site.yml -i inventory.yml       # → installe le cluster
# sudo avec mot de passe : ajouter --ask-become-pass
```

### Où tombe le kubeconfig ?

| Emplacement | Quand |
|---|---|
| `/etc/rancher/k3s/k3s.yaml` sur **chaque serveur** | toujours (emplacement natif k3s) |
| `~ton_user/.kube/config` sur **le nœud** | si `user_kubectl: true` (défaut) |
| `~/.kube/config` du **control node**, contexte **`k3s-ansible`** | fin de run, **si `kubectl` est installé sur le control node** |

Le rapatriement vers le control node réécrit l'adresse `127.0.0.1` → `api_endpoint:api_port`.
⚠️ **Sans `kubectl` sur le control node, ce rapatriement est sauté** (étape 1 → installe kubectl).

```bash
kubectl config use-context k3s-ansible
kubectl get nodes                          # doit lister servers + agents
```

Repli manuel si besoin :

```bash
scp ton_user@192.168.56.11:/etc/rancher/k3s/k3s.yaml ./kubeconfig
# remplacer 127.0.0.1 par l'IP du serveur dans ./kubeconfig
```

---

## Étape 4bis — Adapter le pool MetalLB au réseau des nœuds

MetalLB (contrôleur **+** pool) est géré **en GitOps** dans classlab-infra. En mode L2,
la plage d'IP du pool **doit être dans le même sous-réseau que tes nœuds**, sinon les
services `LoadBalancer` (Traefik, etc.) restent injoignables.

Avant le bootstrap, édite la plage dans **le repo classlab-infra** (source de vérité
GitOps — pas dans le clone côté Ansible, qu'ArgoCD écraserait) puis pousse :

```bash
# classlab-infra/infrastructure/metallb-config/manifests/pool.yaml
#   addresses:
#     - 10.0.0.240-10.0.0.250        # ← plage libre DANS le sous-réseau des nœuds
git -C classlab-infra add infrastructure/metallb-config/manifests/pool.yaml
git -C classlab-infra commit -m "metallb: pool adapté au réseau des nœuds"
git -C classlab-infra push origin main
```

> `--disable=servicelb` (étape 3) est **obligatoire** : le LB intégré de k3s (Klipper)
> entrerait en conflit avec MetalLB.

---

## Étape 5 — Bootstrap de la plateforme GitOps (classlab-infra)

Ce job tourne **sur le 1er nœud server** (il y a déjà `kubectl` + le kubeconfig local),
clone `classlab-infra` et rejoue son `bootstrap.sh` idempotent (installe les CRD ArgoCD →
attend `Established` → applique les root-apps). Il **ne dépend pas** du kubeconfig du control node.

```bash
ansible-playbook playbooks/classlab-infra.yml -i inventory.yml
#   → répondre "yes" au prompt

# non-interactif :
ansible-playbook playbooks/classlab-infra.yml -i inventory.yml -e classlab_infra_deploy=yes
```

Prérequis côté nœud (remplis après l'étape 4) : `kubectl`, `/etc/rancher/k3s/k3s.yaml`,
et **accès sortant vers github.com** (le `git clone`).

Overrides possibles (voir `inventory-sample.yml`) : `classlab_infra_repo`,
`classlab_infra_version`, `classlab_infra_dest`, `classlab_infra_kubeconfig`.

---

## Étape 6 — Suivre la convergence ArgoCD

```bash
kubectl -n argocd get applications          # attendre tout Synced / Healthy

# mot de passe admin initial ArgoCD
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d ; echo

# UI (user: admin)
kubectl -n argocd port-forward svc/argocd-server 8080:443   # → https://localhost:8080
```

L'ordre inter-Applications est **best-effort** : certaines apps passent brièvement
`Degraded`/`OutOfSync` (ex. `cert-manager-issuers` avant les CRD cert-manager) puis
convergent — ArgoCD retente automatiquement. Détail des composants et de l'exploitation :
voir le README et `docs/TUTORIAL.md` de **classlab-infra**.

---

## Opérations courantes

| Besoin | Commande |
|---|---|
| Re-récupérer le kubeconfig | `ansible-playbook playbooks/site.yml -i inventory.yml --tags kubeconfig` |
| Monter k3s de version | éditer `k3s_version` → `ansible-playbook playbooks/upgrade.yml -i inventory.yml` |
| Désinstaller k3s | `ansible-playbook playbooks/reset.yml -i inventory.yml` |
| Reboot ordonné | `ansible-playbook playbooks/reboot.yml -i inventory.yml` |
| Rejouer le bootstrap GitOps | `ansible-playbook playbooks/classlab-infra.yml -i inventory.yml -e classlab_infra_deploy=yes` (idempotent) |

---

## Dépannage

| Symptôme | Cause probable / fix |
|---|---|
| `ansible ... -m ping` échoue | clé SSH pas déployée, mauvais `ansible_user`, host inconnu → refaire l'étape 2 |
| Demande un mot de passe sudo | pas de `NOPASSWD` → ajouter `--ask-become-pass` ou le sudoers de l'étape 2 |
| Rien dans `~/.kube/config` du control node | `kubectl` absent du control node → installer (étape 1) puis `--tags kubeconfig` |
| `classlab-infra.yml` ne fait rien | prompt laissé à `no` (défaut) → répondre `yes` ou `-e classlab_infra_deploy=yes` |
| bootstrap échoue sur kubeconfig | cluster pas prêt → lancer `site.yml` d'abord (l'`assert` du rôle le signale) |
| bootstrap échoue sur le `git clone` | pas d'accès github.com depuis le nœud → vérifier la sortie internet / proxy |
| Pods pod-à-pod inter-nœuds cassés | flannel sur la mauvaise interface (NAT partagée) → forcer `node-ip`/`flannel-iface`, cf. TUTORIAL classlab-infra |
