<# Mini-Hébergeur Associatif — Guide Reproductible (BUT3)

Stack :
- Proxmox VE (cluster + virtualisation)
- Ceph (stockage distribué)
- pfSense (pare-feu + routage inter-VLAN)
- Docker (conteneurs)
- K3s (orchestration Kubernetes légère)
- Traefik (Ingress / reverse proxy K3s)

## Objectif
Reproduire une infrastructure capable d’héberger une application 
multi-tiers (Web + BDD) exposée via Traefik, avec stockage persistant 
sur Ceph, segmentation réseau par VLAN, et tests de validation 
(fonctionnel, disponibilité, persistance).

## Adressage VLAN (final)

### VLAN 10 — Proxmox Management (10.0.10.8/29)
PC1 10.0.10.11 · PC2 10.0.10.12 · PC3 10.0.10.13 · pfSense 10.0.10.14

### VLAN 11 — Cluster Proxmox (10.0.11.8/29)
PC1 10.0.11.11 · PC2 10.0.11.12 · PC3 10.0.11.13 · pfSense 10.0.11.14

### VLAN 12 — Réseau VM / Services (10.0.12.8/29)
VM1 (K3s server) 10.0.12.11 · VM2 (worker) 10.0.12.12 · VM3 (worker) 
10.0.12.13 · pfSense 10.0.12.14

### VLAN 20 — Admin / MGMT (10.0.20.0/29)
PC Admin 10.0.20.1 · Switch 10.0.20.2 · pfSense 10.0.20.6

### VLAN 30 — Stockage Ceph (10.0.30.8/29)
10.0.30.11 · 10.0.30.12 · 10.0.30.13 · pfSense 10.0.30.14

## Parcours du guide (linéaire)
1. docs/00-overview.md
2. docs/01-prerequis.md
3. docs/03-reseau-vlans.md
4. docs/04-proxmox-cluster.md
5. docs/05-stockage-ceph.md
6. docs/06-k3s.md
7. docs/08-deploiement-app.md
8. docs/10-tests-validation.md
9. docs/09-securite.md
10. docs/11-troubleshooting.md

