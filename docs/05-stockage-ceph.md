# 05 — Stockage distribué : Ceph (Proxmox)

Objectif : déployer **Ceph** sur les 3 nœuds Proxmox afin de fournir un 
**stockage distribué** (SDS) tolérant aux pannes, utilisé pour stocker 
les disques des VMs (RBD) et valider la solution par des tests (état 
cluster + `fio`).

> Nœuds : `pve1`, `pve2`, `pve3`  
> Réseau Ceph : **VLAN 30** (dédié stockage) : `10.0.30.8/29`  
> IPs : pve1 `10.0.30.11` · pve2 `10.0.30.12` · pve3 `10.0.30.13`

---

## 1) Pré-requis (à vérifier avant d’installer Ceph)

### 1.1 Disques OSD
- Chaque nœud doit disposer d’au moins **1 disque dédié** (non utilisé 
par Proxmox) pour Ceph.
- Ce disque sera **effacé** (wipe) lors de la création de l’OSD.

> Exemple : sur chaque nœud, un disque `/dev/sdb` dédié Ceph.

### 1.2 Connectivité VLAN 30
Sur chaque nœud (pve1, pve2, pve3) :

```bash
ping -c 2 10.0.30.11
ping -c 2 10.0.30.12
ping -c 2 10.0.30.13
```

Résultat attendu : ping OK entre nœuds sur VLAN 30.

---

## 2) Installation Ceph (sur chaque nœud)

Sur **pve1**, **pve2** et **pve3** :

```bash
pveceph install
```

> Si votre Proxmox demande une version Ceph : choisir la version stable 
proposée (souvent **Reef** sur Proxmox 8).

Vérification rapide :

```bash
ceph -v
```

---

## 3) Initialisation du cluster Ceph (une seule fois)

### 3.1 Init (sur pve1 uniquement)
Sur `pve1` :

```bash
pveceph init --network 10.0.30.8/29
```

> On force Ceph à utiliser le réseau Storage (VLAN 30) comme **public 
network**.  
> Si vous avez une séparation “public/cluster network”, elle n’est pas 
utilisée ici (projet matériel modeste).

Vérifier que la config est prise en compte :

```bash
cat /etc/pve/ceph.conf
```

Vous devez voir une ligne proche de :
- `public_network = 10.0.30.8/29`

---

## 4) Déploiement des services Ceph (MON + MGR)

Principe :
- **MON** : cœur du cluster (quorum / état)
- **MGR** : modules de gestion / dashboard / métriques

### 4.1 Créer MON (sur chaque nœud)
Sur `pve1` :

```bash
pveceph mon create
```

Sur `pve2` :

```bash
pveceph mon create
```

Sur `pve3` :

```bash
pveceph mon create
```

### 4.2 Créer MGR (sur chaque nœud)
Sur `pve1` :

```bash
pveceph mgr create
```

Sur `pve2` :

```bash
pveceph mgr create
```

Sur `pve3` :

```bash
pveceph mgr create
```

---

## 5) Création des OSD (stockage réel)

### 5.1 Identifier le disque dédié (IMPORTANT)
Sur chaque nœud, lister les disques :

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

Repérer le disque **vide** et dédié Ceph (ex : `/dev/sdb`).

⚠️ Ne jamais utiliser le disque système Proxmox.

### 5.2 Créer l’OSD (sur chaque nœud)
Exemple si le disque dédié est `/dev/sdb` :

Sur `pve1` :

```bash
pveceph osd create /dev/sdb
```

Sur `pve2` :

```bash
pveceph osd create /dev/sdb
```

Sur `pve3` :

```bash
pveceph osd create /dev/sdb
```

> Proxmox/Ceph configure BlueStore automatiquement (cas standard).

---

## 6) Vérifications de santé (indispensable)

### 6.1 État global
Sur n’importe quel nœud :

```bash
ceph -s
```

Résultat attendu :
- `HEALTH_OK` (ou `HEALTH_WARN` temporaire le temps d’initialisation)
- 3 MON en quorum
- 3 OSD up/in

### 6.2 Topologie OSD
```bash
ceph osd tree
```

### 6.3 Capacité
```bash
ceph df
```

---

## 7) Créer un pool RBD pour les VMs

Objectif : créer un pool qui servira à stocker des **images RBD** 
(disques VM).

### 7.1 Créer le pool
Sur un nœud :

```bash
ceph osd pool create vm-pool
```

Activer l’autoscaling des PG (recommandé sur versions récentes) :

```bash
ceph osd pool set vm-pool pg_autoscale_mode on
```

### 7.2 Initialiser RBD
```bash
rbd pool init vm-pool
```

### 7.3 Vérifier
```bash
ceph osd pool ls
```

---

## 8) Ajouter Ceph (RBD) comme stockage dans Proxmox

### 8.1 Méthode UI (recommandée pour éviter les erreurs)
Dans Proxmox :
- **Datacenter → Storage → Add → RBD**
- ID : `ceph-vm`
- Pool : `vm-pool`
- Monitors : `10.0.30.11 10.0.30.12 10.0.30.13`
- Content : `Disk image`, (optionnel : `Container`)
- Valider

### 8.2 Vérification côté Proxmox
Vous devez voir le stockage apparaître “OK”, et pouvoir créer une VM 
avec disque sur `ceph-vm`.

---

## 9) Validation performance (fio) — benchmark simple et propre

But : exécuter un benchmark reproductible et obtenir des valeurs 
comparables.

### 9.1 Préparer l’outil fio
Sur un nœud (ex : pve1) :

```bash
apt update
apt install -y fio
```

### 9.2 Créer une image RBD de test
```bash
rbd create vm-pool/fio-test --size 10G
```

Mapper l’image (crée un device /dev/rbdX) :

```bash
rbd map vm-pool/fio-test
rbd showmapped
```

Repérer le device (souvent `/dev/rbd0`).

### 9.3 Formater + monter
```bash
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/fio
mount /dev/rbd0 /mnt/fio
```

### 9.4 Lancer fio (60s)
```bash
fio --name=randrw   --ioengine=libaio --direct=1   --rw=randrw 
--rwmixread=70   --bs=4k --iodepth=32 --numjobs=4   --size=2G 
--runtime=60 --time_based   --group_reporting   
--filename=/mnt/fio/testfile
```

### 9.5 Nettoyage
```bash
umount /mnt/fio
rbd unmap /dev/rbd0
rbd rm vm-pool/fio-test
```

> Conserver la sortie fio dans vos preuves (si besoin) : rediriger vers 
un fichier.

---

## 10) Validation résilience (comportement en panne)

### 10.1 Panne simulée (simple)
Exemples :
- arrêter un OSD (sur 1 nœud)  
- ou éteindre 1 nœud Proxmox

Ce que vous observez :
- `ceph -s` passe en `HEALTH_WARN` (normal)
- les données restent accessibles (réplication)

### 10.2 Retour à la normale
Après redémarrage :
- `ceph -s` revient à `HEALTH_OK`
- les OSD repassent `up/in`

---

## 11) Erreurs fréquentes (et causes)

### 11.1 `HEALTH_WARN` persistant
- OSD down (disque absent / mal créé)
- horloge désynchronisée (NTP)
- réseau storage instable (VLAN 30 mal trunké)

### 11.2 Impossible d’ajouter le stockage RBD dans Proxmox
- IP moniteurs incorrectes
- Ceph installé sur un seul nœud (MON insuffisants)
- pool non créé / non initialisé (`rbd pool init` manquant)

### 11.3 `pveceph osd create` efface le mauvais disque
- vous n’avez pas vérifié `lsblk` → erreur classique, à éviter 
systématiquement

---

## 12) Checklist de validation (fin du chapitre)
- [ ] `ceph -s` = `HEALTH_OK`
- [ ] 3 MON en quorum + 3 MGR (ou au moins 1 actif + 2 standby)
- [ ] 3 OSD `up/in`
- [ ] pool `vm-pool` existant + `rbd pool init` fait
- [ ] stockage Proxmox (RBD) ajouté et utilisable
- [ ] benchmark fio exécuté et nettoyé

---

## Étape suivante
Chapitre **06 — K3s** : création des VMs (VLAN 12), installation K3s 
(server + agents), puis déploiement Traefik (Ingress) et services 
applicatifs (Web + BDD).

