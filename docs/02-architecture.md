# 02 — Architecture (vue globale)

## 1) Vue logique
L’infrastructure est découpée en 4 couches :

1. **Réseau & sécurité**
   - pfSense assure le routage inter-VLAN et le filtrage des flux.
   - Les VLAN séparent les plans d’administration, de cluster, de 
services et de stockage.

2. **Virtualisation**
   - Proxmox est utilisé pour exécuter les VMs et gérer le cluster.
   - Les nœuds Proxmox hébergent les VMs applicatives et 
d’orchestration.

3. **Stockage distribué (Ceph)**
   - Ceph fournit un stockage résilient et distribué.
   - Le trafic de stockage passe par un VLAN dédié (VLAN 30) pour 
limiter l’impact sur le reste.

4. **Services applicatifs**
   - Les services sont conteneurisés (Docker) puis orchestrés par K3s.
   - Traefik (Ingress) expose le service Web via reverse proxy.

## 2) Rôles des VLAN
- **VLAN 20 (Admin/MGMT)** : administration (poste admin, switch, 
pfSense).
- **VLAN 10 (Proxmox MGMT)** : gestion Proxmox (GUI/SSH, API).
- **VLAN 11 (Cluster)** : trafic interne Proxmox (cluster, HA).
- **VLAN 12 (VM/Services)** : trafic applicatif et K3s (pods, services, 
ingress).
- **VLAN 30 (Storage)** : trafic Ceph (réplication, I/O).

## 3) Chemin d’accès utilisateur
Flux simplifié :
1. L’utilisateur accède au service web via le **reverse proxy Traefik**.
2. Traefik route vers le **service web** dans le cluster K3s.
3. Le service web communique avec la **base de données**.
4. Les données persistantes sont stockées sur **Ceph** (stockage 
distribué).

## 4) Pourquoi Docker + K3s + Traefik
- **Docker** : isolation, portabilité, reproductibilité des services.
- **K3s** : orchestration légère adaptée à des machines modestes.
- **Traefik** : reverse proxy intégré au cluster, gestion simple des 
routes/ingress.

## 5) Points de contrôle (validation)
- Proxmox : chaque nœud visible dans le cluster.
- Ceph : état `HEALTH_OK` et stockage utilisable.
- K3s : nodes `Ready` (1 server + 2 agents).
- Traefik : ingress fonctionnel (accès web OK).
- BDD : persistance validée après redémarrage/panne simulée.
