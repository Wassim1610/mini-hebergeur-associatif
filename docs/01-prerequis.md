# 01 — Pré-requis

## 1) Matériel minimum
- **3 machines physiques** (nœuds Proxmox) : CPU x86_64, 8 Go RAM 
minimum, stockage local.
- **1 machine pfSense** (physique ou VM dédiée) : 2 interfaces réseau 
minimum (WAN/LAN), idéalement 3+ si séparation.
- **1 poste d’administration** (PC admin) : accès au réseau MGMT (VLAN 
20).
- **1 switch manageable** (trunk VLAN) si déploiement physique complet 
(recommandé, mais une alternative 100% virtuelle existe).

## 2) Services retenus
- **Cluster Proxmox** : virtualisation et haute disponibilité des VMs.
- **Ceph** : stockage distribué, persistance des données et tolérance 
aux pannes.
- **Docker** : empaquetage des services applicatifs.
- **K3s** : orchestration légère Kubernetes sur machines modestes.
- **Traefik (Ingress K3s)** : reverse proxy pour exposer les services 
web.
- **Application multi-tiers** :
  - **Web**
  - **Base de données**
  - **Reverse proxy (Traefik)**

## 3) Réseau (adressage final)
> Tous les réseaux sont segmentés en VLAN. L’adressage ci-dessous est 
celui utilisé dans la solution finale.

- **VLAN 10 — Proxmox Management** : `10.0.10.8/29`
  - PC1 `10.0.10.11` · PC2 `10.0.10.12` · PC3 `10.0.10.13` · pfSense 
`10.0.10.14`
- **VLAN 11 — Cluster Proxmox** : `10.0.11.8/29`
  - PC1 `10.0.11.11` · PC2 `10.0.11.12` · PC3 `10.0.11.13` · pfSense 
`10.0.11.14`
- **VLAN 12 — Réseau VM / Services (K3s)** : `10.0.12.8/29`
  - VM1 (K3s server) `10.0.12.11` · VM2 `10.0.12.12` · VM3 `10.0.12.13` 
· pfSense `10.0.12.14`
- **VLAN 20 — Admin / MGMT** : `10.0.20.0/29`
  - PC Admin `10.0.20.1` · Switch `10.0.20.2` · pfSense `10.0.20.6`
- **VLAN 30 — Stockage Ceph** : `10.0.30.8/29`
  - PC1 `10.0.30.11` · PC2 `10.0.30.12` · PC3 `10.0.30.13` · pfSense 
`10.0.30.14`

## 4) Conventions de nommage
Pour garantir la reproductibilité, les noms suivants sont utilisés dans 
les exemples :
- Nœuds Proxmox : `pve1`, `pve2`, `pve3`
- VMs K3s :
  - `k3s-server` (VM1)
  - `k3s-worker-1` (VM2)
  - `k3s-worker-2` (VM3)

## 5) Accès et outils côté admin
- SSH (recommandé) pour administrer Linux et équipements.
- Accès interface web Proxmox : `https://<ip_mgmt_proxmox>:8006`
- Accès interface web pfSense : `https://<ip_vlan20_pfsense>`

## 6) Check-list avant de commencer
- [ ] VLAN configurés (pfSense + switch trunk)
- [ ] Les machines Proxmox sont joignables sur VLAN 10
- [ ] Les réseaux Cluster (VLAN 11) et Ceph (VLAN 30) sont 
routables/isolés selon besoin
- [ ] Les VMs (VLAN 12) peuvent sortir sur Internet si nécessaire 
(packages, dépôts)
