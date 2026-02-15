# Mini-Hébergeur Associatif — Guide Reproductible (BUT3)

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

Guide public de déploiement d’un mini-hébergeur associatif basé sur :
- **Proxmox** (virtualisation) + **Ceph** (stockage distribué)
- **Docker** + **K3s** (orchestration Kubernetes légère)
- **Traefik** (reverse proxy) via K3s
- **pfSense** (pare-feu, routage inter-VLAN)

## Documentation (pas à pas)
La documentation complète est dans `docs/` :

1. [Vue d’ensemble](docs/00-overview.md)  
2. [Prérequis](docs/01-prerequis.md)  
3. [Architecture](docs/02-architecture.md)  
4. [Réseau & VLANs](docs/03-reseau-vlans.md)  
5. [Cluster Proxmox](docs/04-proxmox-cluster.md)  
6. [Stockage Ceph](docs/05-stockage-ceph.md)  
7. [K3s](docs/06-k3s.md)  
8. [Registry Docker](docs/07-docker-registry.md)  
9. [Déploiement applicatif](docs/08-deploiement-app.md)  
10. [Sécurité](docs/09-securite.md)  
11. [Tests & validation](docs/10-tests-validation.md)  
12. [Troubleshooting](docs/11-troubleshooting.md)  
13. [Annexes (diagrammes + configs)](docs/12-annexes.md)

## Diagrammes & configurations
- Diagrammes : `diagrams/`
- Configurations : `configs/`

## Licence
Voir [LICENSE](LICENSE).
