# 04 — Proxmox : installation + réseau VLAN + cluster

Objectif : installer Proxmox VE sur 3 nœuds, appliquer l’adressage VLAN 
final, puis créer un **cluster Proxmox** reproductible.

**Nœuds** : `pve1`, `pve2`, `pve3`  
**pfSense** : routeur/pare-feu inter-VLAN  
**VLAN Management Proxmox (VLAN 10)** : administration Proxmox (UI/SSH)  
**VLAN Cluster (VLAN 11)** : trafic Corosync/cluster Proxmox  
**VLAN Storage (VLAN 30)** : trafic Ceph (préparé ici, utilisé au 
chapitre 05)  
**VLAN VM/Services (VLAN 12)** : VMs K3s/services (utilisé via tag VLAN 
sur les interfaces VM)

---

## 1) Installation Proxmox VE (sur pve1 / pve2 / pve3)

### 1.1 Installation
1. Installer **Proxmox VE** via l’ISO sur **chaque** machine.
2. Pendant l’installation :
   - définir le **hostname** :
     - `pve1` / `pve2` / `pve3`
   - définir un mot de passe root (fort) et un email (si demandé).
   - configurer une IP temporaire si nécessaire (DHCP ou statique).  
     > L’adressage final VLAN sera appliqué ensuite via 
`/etc/network/interfaces`.

### 1.2 Mise à jour système + dépôt no-subscription
Sur **chaque nœud** après la première installation :

```bash
apt update
apt install -y curl gnupg lsb-release
```

Désactiver le dépôt entreprise (s’il existe) et activer no-subscription 
:

```bash
sed -i 's|^deb https://enterprise.proxmox.com|# deb 
https://enterprise.proxmox.com|g' 
/etc/apt/sources.list.d/pve-enterprise.list 2>/dev/null || true

cat > /etc/apt/sources.list.d/pve-no-subscription.list << 'EOF'
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
EOF

apt update
apt full-upgrade -y
reboot
```

---

## 2) Adressage réseau final (VLAN 10 / 11 / 30)

### 2.1 Adressage retenu (celui du projet)
- **VLAN 10 — Proxmox Management** : `10.0.10.8/29`
  - pve1 `10.0.10.11` · pve2 `10.0.10.12` · pve3 `10.0.10.13` · pfSense 
`10.0.10.14`
- **VLAN 11 — Cluster Proxmox** : `10.0.11.8/29`
  - pve1 `10.0.11.11` · pve2 `10.0.11.12` · pve3 `10.0.11.13` · pfSense 
`10.0.11.14`
- **VLAN 30 — Stockage Ceph** : `10.0.30.8/29`
  - pve1 `10.0.30.11` · pve2 `10.0.30.12` · pve3 `10.0.30.13` · pfSense 
`10.0.30.14`

**Gateway Proxmox (important)** : mettre la passerelle sur le VLAN 
Management (VLAN 10) → `10.0.10.14`  
> Cela garantit un accès stable à l’interface Proxmox (admin) et évite 
des routes incohérentes.

---

## 3) Configuration réseau Proxmox : bridge trunk + VLAN-aware

### 3.1 Principe
Chaque nœud Proxmox est connecté à un **port trunk** (switch ou trunk 
virtuel).  
Le trunk transporte : **10, 11, 12, 20, 30**.

On crée :
- un bridge **`vmbr0`** (VLAN-aware) relié à la carte physique
- des sous-interfaces VLAN côté Proxmox pour le 
management/cluster/storage :
  - `vmbr0.10` (MGMT Proxmox)
  - `vmbr0.11` (Cluster)
  - `vmbr0.30` (Storage Ceph)

### 3.2 Identifier l’interface réseau physique
Sur chaque nœud :

```bash
ip link
```

Repérer l’interface reliée au trunk (ex : `eno1` ou `enp1s0`).  
Dans la suite, on l’appelle : **`IFACE_TRUNK`**.

---

## 4) Configuration `/etc/network/interfaces` (pve1, pve2, pve3)

### 4.1 pve1
Éditer le fichier :

```bash
nano /etc/network/interfaces
```

Coller (remplacer `IFACE_TRUNK` par l’interface réelle) :

```ini
auto lo
iface lo inet loopback

iface IFACE_TRUNK inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports IFACE_TRUNK
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 11 12 20 30

auto vmbr0.10
iface vmbr0.10 inet static
    address 10.0.10.11/29
    gateway 10.0.10.14

auto vmbr0.11
iface vmbr0.11 inet static
    address 10.0.11.11/29

auto vmbr0.30
iface vmbr0.30 inet static
    address 10.0.30.11/29
```

Appliquer sans reboot :

```bash
ifreload -a
```

### 4.2 pve2
Même logique sur `pve2` (seules les IP changent) :

```ini
auto lo
iface lo inet loopback

iface IFACE_TRUNK inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports IFACE_TRUNK
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 11 12 20 30

auto vmbr0.10
iface vmbr0.10 inet static
    address 10.0.10.12/29
    gateway 10.0.10.14

auto vmbr0.11
iface vmbr0.11 inet static
    address 10.0.11.12/29

auto vmbr0.30
iface vmbr0.30 inet static
    address 10.0.30.12/29
```

Appliquer :

```bash
ifreload -a
```

### 4.3 pve3
Même logique sur `pve3` :

```ini
auto lo
iface lo inet loopback

iface IFACE_TRUNK inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports IFACE_TRUNK
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10 11 12 20 30

auto vmbr0.10
iface vmbr0.10 inet static
    address 10.0.10.13/29
    gateway 10.0.10.14

auto vmbr0.11
iface vmbr0.11 inet static
    address 10.0.11.13/29

auto vmbr0.30
iface vmbr0.30 inet static
    address 10.0.30.13/29
```

Appliquer :

```bash
ifreload -a
```

---

## 5) Vérifications réseau (immédiates)

### 5.1 Vérifier que les interfaces VLAN existent
Sur chaque nœud :

```bash
ip a | grep -E "vmbr0\.10|vmbr0\.11|vmbr0\.30" -n
```

### 5.2 Vérifier la passerelle et l’accès pfSense
Sur chaque nœud :

```bash
ip route | head
ping -c 2 10.0.10.14
```

### 5.3 Vérifier l’accès à l’interface web Proxmox
Depuis le PC Admin (VLAN 20), ouvrir :
- `https://10.0.10.11:8006`
- `https://10.0.10.12:8006`
- `https://10.0.10.13:8006`

Résultat attendu :
- Les 3 nœuds sont joignables et l’UI Proxmox s’ouvre.

---

## 6) Résolution DNS locale (requis pour le cluster)

Pour éviter des erreurs de join (nom non résolu / mismatch), on fixe les 
noms via `/etc/hosts`.

Sur **chaque nœud** :

```bash
nano /etc/hosts
```

Ajouter :

```ini
10.0.10.11 pve1
10.0.10.12 pve2
10.0.10.13 pve3
```

Tester :

```bash
ping -c 1 pve2
ping -c 1 pve3
```

---

## 7) Création du cluster Proxmox (Corosync sur VLAN 11)

Principe :
- **UI / administration** : VLAN 10
- **trafic cluster Corosync** : VLAN 11 (réseau dédié, plus propre et 
plus stable)

### 7.1 Créer le cluster sur pve1
Sur `pve1` :

```bash
pvecm create mini-hebergeur --link0 10.0.11.11
```

Vérifier :

```bash
pvecm status
```

Résultat attendu : cluster initialisé, quorum OK (1 nœud).

### 7.2 Ajouter pve2
Sur `pve2` :

```bash
pvecm add 10.0.10.11 --link0 10.0.11.12
```

### 7.3 Ajouter pve3
Sur `pve3` :

```bash
pvecm add 10.0.10.11 --link0 10.0.11.13
```

### 7.4 Vérification finale du cluster
Sur `pve1` :

```bash
pvecm nodes
pvecm status
```

Résultat attendu :
- 3 nœuds visibles
- quorum OK (3 votes)

---

## 8) Bonnes pratiques minimales (stabilité)

### 8.1 Synchronisation horaire (recommandé)
Sur les 3 nœuds :

```bash
timedatectl set-ntp true
timedatectl status
```

### 8.2 Choix firewall Proxmox
Pour éviter les incohérences pendant la mise en place :
- soit firewall Proxmox désactivé le temps du déploiement puis activé 
ensuite,
- soit activé dès le départ avec règles strictes.

> La sécurité détaillée est traitée dans les chapitres dédiés, afin de 
garder cette section focalisée sur la reproductibilité du cluster.

---

## 9) Checklist de validation (fin du chapitre)
- [ ] Les 3 UI Proxmox sont accessibles : 
`https://10.0.10.11/12/13:8006`
- [ ] `pvecm status` indique quorum OK
- [ ] Les interfaces VLAN existent :
  - `vmbr0.10` (MGMT) + gateway `10.0.10.14`
  - `vmbr0.11` (Cluster)
  - `vmbr0.30` (Storage)
- [ ] `pve1/pve2/pve3` résolvent correctement (hosts)

---

## 10) Étape suivante
Le prochain chapitre est **05 — Stockage Ceph** :
- création du cluster Ceph,
- choix des disques OSD,
- réseau Ceph sur VLAN 30,
- état `HEALTH_OK`,
- validation perf et comportement via tests (ex : fio).

