# 00 — Overview

## Ce que vous allez construire
Une infrastructure d’hébergement basée sur :
- un cluster Proxmox (3 nœuds)
- un stockage distribué Ceph (VLAN 30)
- un routage/filtrage inter-VLAN via pfSense
- un cluster K3s (1 server + 2 workers) sur VLAN 12
- un reverse proxy Traefik (Ingress K3s) exposant le service Web

## Principes
- Segmentation stricte (MGMT / Cluster / VM / Storage)
- Reproductibilité : commandes + résultats attendus + points de contrôle
- Validation : tests fonctionnels + persistance + résilience

## Chemin critique (ordre obligatoire)
1) Réseau/VLAN (pfSense + switch)  
2) Proxmox sur 3 nœuds  
3) Cluster Proxmox  
4) Ceph (stockage)  
5) Création des VMs (VLAN 12)  
6) Installation K3s (server + agents)  
7) Déploiement applicatif (Web + BDD)  
8) Exposition via Traefik  
9) Tests de validation  
