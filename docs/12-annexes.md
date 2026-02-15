# 12 — Annexes : configurations, scripts, commandes de référence

Objectif : regrouper proprement les éléments “preuves/config” sans 
surcharger les chapitres principaux.

> Règle : repo public → aucun secret réel (mots de passe, clés 
privées).

---

## A) Adressage VLAN (rappel)

- VLAN 10 — Proxmox Management : `10.0.10.8/29`
  - pve1 `10.0.10.11` · pve2 `10.0.10.12` · pve3 `10.0.10.13` · pfSense 
`10.0.10.14`
- VLAN 11 — Proxmox Cluster : `10.0.11.8/29`
  - pve1 `10.0.11.11` · pve2 `10.0.11.12` · pve3 `10.0.11.13` · pfSense 
`10.0.11.14`
- VLAN 12 — VM / Services : `10.0.12.8/29`
  - k3s-server `10.0.12.11` · worker1 `10.0.12.12` · worker2 
`10.0.12.13` · pfSense `10.0.12.14`
- VLAN 20 — Admin/MGMT : `10.0.20.0/29` (ex : switch mgmt `10.0.20.2`, 
gateway `10.0.20.6`)
- VLAN 30 — Storage Ceph : `10.0.30.8/29`
  - pve1 `10.0.30.11` · pve2 `10.0.30.12` · pve3 `10.0.30.13` · pfSense 
`10.0.30.14`

---

## B) Configuration switch (extrait)
Fichier recommandé :
- `configs/switch/switch_config.txt`

Contient :
- création VLAN 10/11/12/20/30
- trunks (ports cluster + uplink)
- IP management VLAN20
- SSH activé
- ports inutilisés shutdown

---

## C) Configuration Proxmox (réseau)
Fichiers recommandés :
- `configs/proxmox/pve1_interfaces.txt`
- `configs/proxmox/pve2_interfaces.txt`
- `configs/proxmox/pve3_interfaces.txt`

Contenu :
- `vmbr0` VLAN-aware (trunk)
- `vmbr0.10` gateway `10.0.10.14`
- `vmbr0.11` cluster
- `vmbr0.30` storage

---

## D) Ceph (commandes utiles)
```bash
ceph -s
ceph health detail
ceph osd tree
ceph df
```

Pool RBD :
```bash
ceph osd pool ls
rbd ls vm-pool
```

---

## E) K3s (commandes utiles)
Noeuds :
```bash
kubectl get nodes -o wide
```

Pods :
```bash
kubectl get pods -A
kubectl -n app get pods,svc,ingress
```

Traefik :
```bash
kubectl -n kube-system get deploy traefik
kubectl -n kube-system logs deploy/traefik --tail=100
```

---

## F) Registry Docker (commandes utiles)
Vérifier catalogue :
```bash
curl http://10.0.12.11:5000/v2/_catalog
```

---

## G) Diagrammes

### 12.1.1 Diagramme de Gantt
Le diagramme de Gantt ci-dessous présente la planification globale du 
projet sur la période **01/11/2025 → 08/01/2026**.

![Diagramme de Gantt](../diagrams/Diagramme_Gantt copie.png)

### H)Topologie réseau
Le schéma ci-dessous illustre l’architecture réseau finale 
(WAN/LAN/pfSense, trunk vers le switch, segmentation VLAN, nœuds K3s et 
poste d’administration).

![Topologie réseau](../diagrams/Topologie_Reseau.png)

---

## I)Configurations (PDF)

Les configurations sont fournies au format PDF pour garantir la 
lisibilité et conserver une version “référence” stable.

### J)Configuration du switch
- PDF : [Config switch](../configs/switch/Config_switch.pdf)

### K)Configuration Docker & K3s
- PDF : [Config Docker & K3s](../configs/k3s/Config_Docker_K3s.pdf)

---

## L)Références rapides

- Diagrammes :
  - `diagrams/Diagramme_Gantt copie.png`
  - `diagrams/Topologie_Reseau.png`
- Configurations :
  - `configs/switch/Config_switch.pdf`
  - `configs/k3s/Config_Docker_K3s.pdf`

## Fin
Ces annexes complètent les chapitres principaux sans alourdir le guide, 
et permettent de retrouver rapidement les configurations et commandes de 
référence.

