# 10 — Tests & Validation : HA, stockage, fonctionnement applicatif

Objectif : documenter les **preuves** que l’infrastructure répond aux 
exigences :
- fonctionnement applicatif (Web + BDD + Traefik)
- stockage distribué Ceph : état + performance (fio)
- résilience / disponibilité : comportement en cas de panne (nœud/VM)

Ce chapitre doit permettre au lecteur de comprendre **quoi tester**, 
**comment**, et **quel résultat** est attendu.

---

## 1) Tests de fonctionnement applicatif (multi-services)

### 1.1 Vérifier l’état des pods (K3s)
Sur `k3s-server` :

```bash
sudo kubectl -n app get pods -o wide
sudo kubectl -n app get deploy,svc,ingress
```

Résultat attendu :
- pods `web` et `mariadb` en `Running`
- services `web` et `mariadb` présents
- ingress `web-ingress` présent

### 1.2 Test HTTP interne (dans le cluster)
```bash
sudo kubectl -n app run curltest --image=curlimages/curl:8.5.0 -it --rm 
--   curl -I http://web
```

Résultat attendu :
- réponse HTTP `200/301/302` selon l’app (une réponse valide suffit)

### 1.3 Test via Traefik (Ingress)
Depuis le poste admin (ou une VM sur le réseau) :
- résolution du nom (DNS/hosts) → `app.footstore.local`
- accès : `http://app.footstore.local`

Résultat attendu :
- page web chargée via Traefik (reverse proxy)

---

## 2) Validation du stockage Ceph

### 2.1 Santé Ceph
Sur un nœud Proxmox :

```bash
ceph -s
ceph osd tree
ceph df
```

Résultat attendu :
- `HEALTH_OK` (ou warnings justifiés)
- 3 MON en quorum
- OSD `up/in`

### 2.2 Test de performance (fio) — méthode reproductible
Le but : obtenir des valeurs comparables.

Sur un nœud Proxmox (ex : pve1) :

1) Installer fio :
```bash
apt update
apt install -y fio
```

2) Créer une image RBD de test (10G) :
```bash
rbd create vm-pool/fio-test --size 10G
rbd map vm-pool/fio-test
rbd showmapped
```

3) Formater + monter :
```bash
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/fio
mount /dev/rbd0 /mnt/fio
```

4) Lancer fio (60s) :
```bash
fio --name=randrw   --ioengine=libaio --direct=1   --rw=randrw 
--rwmixread=70   --bs=4k --iodepth=32 --numjobs=4   --size=2G 
--runtime=60 --time_based   --group_reporting   
--filename=/mnt/fio/testfile
```

5) Nettoyage :
```bash
umount /mnt/fio
rbd unmap /dev/rbd0
rbd rm vm-pool/fio-test
```

Résultat attendu :
- fio s’exécute sans erreur
- sortie contenant IOPS/latency/bandwidth (à conserver si besoin)

---

## 3) Validation HA / Résilience (Proxmox)

### 3.1 Objectif
Mesurer et prouver :
- continuité de service en cas de panne d’un nœud
- capacité de redémarrage et retour à l’état nominal

> Dans un projet étudiant, on valide surtout le comportement (bascule / 
reprise) et la stabilité.

### 3.2 Test 1 — Panne d’un nœud Proxmox (simulation)
Préparer :
- s’assurer que l’app est accessible (`app.footstore.local`)
- ouvrir un ping / un navigateur pour observer

Action :
- éteindre brutalement un nœud (ex : `pve2`) :
  - via Proxmox : **Shutdown**
  - ou bouton power (si simulation “hard”)

Observation :
- cluster Proxmox : un nœud down
- Ceph : possible `HEALTH_WARN` temporaire
- application : doit rester accessible si les VMs/services ne sont pas 
sur le nœud arrêté ou si redémarrage automatique configuré

Résultat attendu :
- infrastructure continue de fonctionner (ou reprise documentée si 
redémarrage nécessaire)

### 3.3 Test 2 — Redémarrage du nœud
Action :
- rallumer le nœud
- attendre resync Ceph si besoin

Vérifier :
```bash
pvecm status
ceph -s
```

Résultat attendu :
- quorum OK
- Ceph revient en `HEALTH_OK`

---

## 4) Validation réseau (VLAN / flux)

### 4.1 Vérifier la segmentation
Depuis le poste admin (VLAN20) :
- accès Proxmox (VLAN10) OK
- accès direct BDD depuis l’extérieur : **non** (si filtrage correct)
- accès web via Traefik : OK

### 4.2 Vérifier les flux autorisés uniquement
Dans pfSense :
- logs firewall cohérents
- règles “deny by default” + exceptions justifiées

---

## 5) Synthèse des preuves attendues (liste courte)

- [ ] App accessible via Traefik (Ingress)
- [ ] Pods `web` et `mariadb` en `Running`
- [ ] Ceph : `HEALTH_OK` + OSD `up/in`
- [ ] fio exécuté (sortie exploitable)
- [ ] comportement cohérent en panne/reprise (cluster + stockage)

---

## Étape suivante
Chapitre **11 — Troubleshooting** : problèmes fréquents (VLAN, Ceph, 
K3s, Traefik) et méthodes de diagnostic rapides.

