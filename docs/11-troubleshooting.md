# 11 — Troubleshooting : problèmes fréquents + diagnostic rapide

Objectif : fournir une section utile à quelqu’un qui reprend le projet, 
avec des pannes typiques et les commandes de diagnostic associées.

---

## 1) Réseau / VLAN

### 1.1 Proxmox inaccessible (UI 8006)
**Symptômes**
- `https://10.0.10.11:8006` ne répond pas

**Diagnostics**
Sur le poste admin :
```bash
ping 10.0.10.11
```

Sur le nœud Proxmox :
```bash
ip a
ip route
ss -lntp | grep 8006
```

**Causes fréquentes**
- passerelle mal configurée (doit être sur VLAN 10 → `10.0.10.14`)
- trunk VLAN incomplet (VLAN10 non transporté)
- pfSense bloque le flux VLAN20 → VLAN10

---

### 1.2 VMs VLAN 12 ne communiquent pas
**Diagnostics**
Sur la VM :
```bash
ip a
ip route
ping -c 2 10.0.12.14
```

Sur Proxmox (vérifier le tag VLAN) :
- interface VM connectée à `vmbr0` + VLAN tag `12`

**Causes fréquentes**
- VLAN tag absent ou mauvais
- trunk ne transporte pas VLAN12
- interface VLAN12 pfSense down

---

## 2) Cluster Proxmox

### 2.1 `pvecm status` montre “no quorum”
**Diagnostics**
```bash
pvecm status
pvecm nodes
journalctl -u corosync --no-pager | tail -n 80
```

**Causes fréquentes**
- lien VLAN11 KO (cluster network)
- mauvaise IP `--link0`
- horloge non synchronisée

**Correctifs**
- vérifier VLAN11 (ping entre `10.0.11.11/12/13`)
- activer NTP :
```bash
timedatectl set-ntp true
```

---

## 3) Ceph

### 3.1 `ceph -s` en `HEALTH_WARN`
**Diagnostics**
```bash
ceph -s
ceph health detail
ceph osd tree
```

**Causes fréquentes**
- un OSD down (disque, service)
- réseau storage instable (VLAN30)
- clock skew

**Commandes utiles**
```bash
systemctl status ceph-osd@* --no-pager
journalctl -u ceph-osd@* --no-pager | tail -n 80
```

---

### 3.2 OSD ne remonte pas après reboot
**Diagnostics**
```bash
lsblk
ceph osd tree
```

**Correctifs**
- vérifier que le disque est bien présent
- relancer service OSD si besoin :
```bash
systemctl restart ceph-osd@<ID>
```

---

## 4) K3s

### 4.1 Worker ne rejoint pas le cluster
**Diagnostics**
Sur le worker :
```bash
sudo systemctl status k3s-agent --no-pager
sudo journalctl -u k3s-agent --no-pager | tail -n 80
```

Vérifier connectivité :
```bash
ping -c 2 10.0.12.11
curl -k https://10.0.12.11:6443
```

**Causes fréquentes**
- token incorrect
- firewall bloque 6443
- IP node incorrecte (`--node-ip`)

---

### 4.2 Pods en `ImagePullBackOff` (registry local)
**Diagnostics**
```bash
sudo kubectl -n app describe pod <pod>
```

Vérifier `registries.yaml` :
- sur server : `/etc/rancher/k3s/registries.yaml`
- sur workers : même fichier + restart agent

**Correctif**
```bash
sudo systemctl restart k3s
sudo systemctl restart k3s-agent
```

---

## 5) Traefik / Ingress

### 5.1 Ingress créé mais site inaccessible
**Diagnostics**
```bash
sudo kubectl -n app get ingress
sudo kubectl -n kube-system get svc traefik
sudo kubectl -n kube-system logs deploy/traefik --tail=100
```

**Causes fréquentes**
- le hostname ne résout pas (DNS/hosts)
- mauvais service backend (nom/port)
- app web non prête

---

## 6) Checklist “sanity checks” (5 minutes)
Sur `k3s-server` :

```bash
sudo kubectl get nodes -o wide
sudo kubectl -n app get pods,svc,ingress
sudo kubectl -n kube-system get pods -l app.kubernetes.io/name=traefik
```

Sur un nœud Proxmox :

```bash
pvecm status
ceph -s
```

---

## Étape suivante
Chapitre **12 — Annexes** : configurations utiles (pfSense/switch), 
scripts, commandes de référence et liens internes vers les fichiers 
`configs/` et `scripts/`.

