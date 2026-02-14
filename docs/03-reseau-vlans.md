# 03 — Réseau & VLAN (pfSense + switch)

Objectif : mettre en place une segmentation réseau reproductible, avec 
routage inter-VLAN contrôlé par pfSense, et trunk VLAN côté switch.

---

## 1) Adressage VLAN (final)

- **VLAN 10 — Proxmox Management** : `10.0.10.8/29`
  - pve1 `10.0.10.11` · pve2 `10.0.10.12` · pve3 `10.0.10.13` · pfSense 
`10.0.10.14`
- **VLAN 11 — Cluster Proxmox** : `10.0.11.8/29`
  - pve1 `10.0.11.11` · pve2 `10.0.11.12` · pve3 `10.0.11.13` · pfSense 
`10.0.11.14`
- **VLAN 12 — Réseau VM / Services (K3s)** : `10.0.12.8/29`
  - VM1 `10.0.12.11` · VM2 `10.0.12.12` · VM3 `10.0.12.13` · pfSense 
`10.0.12.14`
- **VLAN 20 — Admin / MGMT** : `10.0.20.0/29`
  - Admin `10.0.20.1` · Switch `10.0.20.2` · pfSense `10.0.20.6`
- **VLAN 30 — Stockage Ceph** : `10.0.30.8/29`
  - pve1 `10.0.30.11` · pve2 `10.0.30.12` · pve3 `10.0.30.13` · pfSense 
`10.0.30.14`

---

## 2) Configuration switch (trunks + VLAN)

### 2.1 VLAN
Créer les VLAN et les nommer :
- VLAN 10 : ProxmoxManagement
- VLAN 11 : ProxmoxCluster
- VLAN 12 : VM
- VLAN 20 : MGMT
- VLAN 30 : Storage

### 2.2 Ports (logique)
- **Port vers pfSense** : trunk avec **10,11,12,20,30**
- **Ports vers pve1/pve2/pve3** : trunk avec **10,11,12,30**
- **Port vers PC Admin** : access VLAN **20**

> Remarque : les numéros de ports dépendent du matériel. La logique 
reste identique : pfSense doit voir tous les VLAN, les nœuds Proxmox 
doivent voir VLAN 10/11/12/30, le poste admin uniquement VLAN 20.

---

## 3) pfSense : création des interfaces VLAN

### 3.1 Pré-requis
- pfSense dispose d’une interface physique connectée au trunk (vers 
switch).
- Cette interface servira de parent pour les VLAN.

### 3.2 Créer les VLAN
Dans pfSense :
- **Interfaces → Assignments → VLANs → Add**
Créer :
- VLAN 10 sur l’interface trunk
- VLAN 11 sur l’interface trunk
- VLAN 12 sur l’interface trunk
- VLAN 20 sur l’interface trunk
- VLAN 30 sur l’interface trunk

### 3.3 Assigner les interfaces
Dans **Interfaces → Assignments** :
- Ajouter chaque VLAN comme interface (OPT…)
- Renommer proprement :
  - `VLAN10_MGMT_PROXMOX`
  - `VLAN11_CLUSTER`
  - `VLAN12_VM_SERVICES`
  - `VLAN20_ADMIN`
  - `VLAN30_STORAGE`

### 3.4 IP statiques (pfSense)
Pour chaque interface VLAN :
- activer l’interface
- IPv4 Configuration Type : **Static IPv4**
- mettre les IP suivantes :

- VLAN10 : `10.0.10.14/29`
- VLAN11 : `10.0.11.14/29`
- VLAN12 : `10.0.12.14/29`
- VLAN20 : `10.0.20.6/29`
- VLAN30 : `10.0.30.14/29`

---

## 4) Règles firewall (minimales et propres)

Principe : **tout bloquer par défaut** et ouvrir uniquement ce qui est 
nécessaire.

### 4.1 VLAN20 (Admin) → administration
Autoriser :
- Admin → pfSense (HTTPS/SSH si utilisé)
- Admin → Proxmox (HTTPS 8006 + SSH)
- Admin → VMs (SSH/HTTP/HTTPS selon besoin)

Règles typiques :
- `VLAN20 net` → `VLAN10 net` : TCP 8006,22
- `VLAN20 net` → `VLAN12 net` : TCP 22,80,443

### 4.2 VLAN12 (VM/Services) → Internet (si nécessaire)
Utile pour :
- installation de paquets
- pull d’images
- accès dépôts

Règle :
- `VLAN12 net` → `WAN` : TCP 80,443 + DNS (53)

### 4.3 VLAN12 (VM/Services) → VLAN30 (Ceph)
Utile si les VMs consomment du stockage Ceph (RBD/CephFS via Proxmox).  
Sinon, garder fermé.

Règle (si besoin) :
- `VLAN12 net` → `VLAN30 net` : autoriser uniquement les ports 
nécessaires au mode retenu.

### 4.4 VLAN11 (Cluster) et VLAN30 (Storage)
- Ces VLAN sont **internes**.
- Éviter l’accès depuis d’autres VLAN sauf Admin si besoin de debug.
- Garder **strict** : l’objectif est de limiter la surface d’attaque.

---

## 5) Passerelles / routes

### 5.1 Passerelle par défaut des hôtes
Chaque machine/VM doit utiliser comme gateway :
- VLAN10 : `10.0.10.14`
- VLAN11 : `10.0.11.14` (si nécessaire)
- VLAN12 : `10.0.12.14`
- VLAN20 : `10.0.20.6`
- VLAN30 : `10.0.30.14` (si nécessaire)

En pratique :
- Proxmox : gateway sur le VLAN management (VLAN10).
- VMs : gateway sur VLAN12.
- PC Admin : gateway sur VLAN20.

---

## 6) Vérifications (à faire à chaque étape)

### 6.1 Depuis le PC Admin (VLAN20)
- ping pfSense (VLAN20) : `10.0.20.6`
- ping switch (VLAN20) : `10.0.20.2`
- ping Proxmox (VLAN10) : `10.0.10.11/12/13`

### 6.2 Depuis une VM (VLAN12)
- ping pfSense VLAN12 : `10.0.12.14`
- ping autres VMs : `10.0.12.12/13`
- test DNS (si autorisé) : `nslookup github.com`
- test HTTP (si autorisé) : `curl -I https://google.com`

### 6.3 Résultat attendu
- Chaque VLAN est isolé.
- Les accès d’administration passent uniquement par VLAN20.
- Les VMs ont la connectivité nécessaire (interne + Internet si ouvert).
- Le trafic cluster/stockage reste interne et contrôlé.

---

## 7) Erreurs fréquentes (et causes)
- **Pas d’accès aux VLAN** : trunk non configuré ou VLAN non autorisés 
sur le port.
- **Ping inter-VLAN KO** : règles firewall manquantes sur pfSense (par 
défaut tout bloque).
- **Proxmox accessible puis plus** : gateway mal placée (mettre la 
gateway sur le VLAN management).
- **VM sans Internet** : DNS bloqué ou NAT/WAN non configuré côté 
pfSense.
