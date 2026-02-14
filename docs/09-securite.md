# 09 — Sécurité : durcissement, accès, segmentation VLAN, sauvegardes

Objectif : documenter les mesures de sécurité mises en place dans le 
projet, de façon **compréhensible**, **argumentée**, et alignée avec des 
bonnes pratiques (références type ANSSI), sans prétendre à une 
certification.

Périmètre sécurité couvert :
- segmentation réseau par VLAN + filtrage (pfSense)
- accès admin (SSH, comptes, mots de passe, clés)
- hygiène système (MAJ, services minimaux)
- sécurité cluster (Proxmox, K3s)
- sauvegardes / restauration
- journalisation et contrôle

> Important : ce guide est public (GitHub).  
> **Aucun secret réel** (mdp, clés privées) ne doit être publié.

---

## 1) Segmentation réseau (VLAN) : réduction de la surface d’attaque

### 1.1 Objectif
Isoler les flux afin de :
- limiter les mouvements latéraux en cas de compromission
- réduire l’exposition des services
- rendre les règles de filtrage explicites

### 1.2 VLANs utilisés (rappel)
- **VLAN 10** : Management Proxmox
- **VLAN 11** : Cluster Proxmox (Corosync)
- **VLAN 12** : Réseau VMs / Services (K3s, Web, BDD, Traefik)
- **VLAN 20** : Admin/Management (poste admin)
- **VLAN 30** : Storage Ceph

---

## 2) Filtrage pfSense : politique “deny by default”

### 2.1 Principe
- **Bloquer par défaut**
- n’autoriser que les flux strictement nécessaires au fonctionnement

### 2.2 Règles essentielles (logique)
Exemples de flux autorisés (à adapter à votre pfSense) :
- VLAN20 (admin) → VLAN10 (Proxmox UI/SSH) : TCP 8006, 22
- VLAN10 ↔ VLAN11 : trafic cluster Proxmox (Corosync) si nécessaire
- VLAN12 → VLAN30 : accès Ceph (si VMs/containers doivent écrire/ lire)
- VLAN12 → Internet : MAJ OS, pulls registry (si autorisé)
- Entrées externes → Traefik (VLAN12) : HTTP/HTTPS (80/443) selon 
besoin

> À documenter dans pfSense : interface, règle, source, destination, 
ports, justification.

### 2.3 NAT et exposition
- exposition via Traefik (Ingress) : un point d’entrée unique
- pas d’exposition directe de la BDD
- limitation des ports ouverts au strict nécessaire

---

## 3) Accès administrateur : contrôle des identités

### 3.1 Comptes et mots de passe
- mots de passe forts (longueur, complexité, non réutilisation)
- comptes nominatifs si possible (pas “admin” partagé)
- rotation si fuite suspectée

### 3.2 SSH : durcissement minimal (recommandé)
Sur les serveurs/VMs Linux :
- désactiver login root SSH (si possible)
- privilégier authentification par clé
- limiter les IPs autorisées (firewall)

Exemple (principe, sans exposer de config complète) :
- `PasswordAuthentication no`
- `PermitRootLogin no`
- `AllowUsers <user>`

### 3.3 Clés SSH
- clé privée jamais partagée/publiée
- clé publique ajoutée sur les machines gérées
- suppression des clés inutilisées

---

## 4) Hygiène système : mises à jour et réduction des services

### 4.1 Mises à jour
- Proxmox : mises à jour régulières via dépôts officiels
- VMs : `apt update && apt upgrade` (planifié)
- K3s : version stable, mise à jour contrôlée

### 4.2 Services minimaux
- supprimer/désactiver services inutiles
- limiter l’exposition réseau (ports)

---

## 5) Sécurisation Proxmox (hôte)

Mesures recommandées et appliquées :
- accès Proxmox uniquement depuis VLAN admin (VLAN20) vers VLAN10
- interface web Proxmox non exposée
- utilisation de VLAN dédié pour le cluster (VLAN11)
- séparation stockage (VLAN30) pour Ceph

---

## 6) Sécurisation K3s (cluster)

### 6.1 Séparation des rôles
- `k3s-server` : plan de contrôle
- workers : exécution des workloads
- trafic interne sur VLAN12

### 6.2 Secrets Kubernetes
- mots de passe DB stockés en **Secret**
- éviter de publier des secrets dans le repo
- utiliser des valeurs d’exemple et injecter en local

### 6.3 Reverse proxy Traefik
- Traefik centralise l’exposition des services
- permet d’appliquer des règles d’accès (Ingress, middlewares)
- évite d’ouvrir des ports arbitraires sur les pods

---

## 7) Sauvegardes et restauration (continuité)

### 7.1 Objectif
- être capable de restaurer l’application et ses données
- éviter la perte de données (BDD)

### 7.2 Sauvegardes
- snapshots/backup VM Proxmox (si utilisé)
- sauvegarde logique BDD (dump) planifiée
- copie vers un stockage séparé (si disponible)

Exemple de stratégie :
- dump DB quotidien (simple)
- rotation 7 jours
- sauvegarde hors machine (minimum : autre nœud / autre disque)

### 7.3 Restauration (test)
- un plan de restauration doit être documenté
- au moins 1 restauration testée (preuve de faisabilité)

---

## 8) Journalisation et contrôle

### 8.1 Logs systèmes
- `journalctl` sur VMs
- logs Proxmox (événements)
- logs K3s / Kubernetes (`kubectl logs`)

### 8.2 Centralisation (optionnel)
Non mis en place dans le périmètre minimal, mais possible :
- syslog central
- stack de monitoring/logging (ELK, Loki, etc.)

---

## 9) Limites et axes d’amélioration (sécurité)

Le projet est un lab : certaines améliorations sont identifiées :
- activer un IDS/IPS dédié (détection proactive)
- tests d’intrusion plus réguliers (scénarios réalistes)
- TLS complet sur Traefik (HTTPS, certificats)
- RBAC renforcé sur Kubernetes (droits minimaux)

---

## Checklist de fin de chapitre
- [ ] VLANs actifs + flux justifiés
- [ ] pfSense “deny by default” + exceptions documentées
- [ ] accès Proxmox non exposé
- [ ] secrets non publiés
- [ ] sauvegardes définies + restauration documentée
- [ ] logs utilisables en diagnostic

---

## Étape suivante
Chapitre **10 — Tests & Validation** : prouver la HA, la stabilité et 
les performances (Ceph fio, bascule, checks).

