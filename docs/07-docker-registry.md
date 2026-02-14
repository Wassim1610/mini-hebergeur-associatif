# 07 — Docker Registry : build + push des images (sans CI/CD)

Objectif : disposer d’un **registry local** pour stocker les images 
Docker de l’application (Web/BDD si nécessaire), afin d’éviter de 
dépendre d’un registre public et de garantir la reproductibilité du 
déploiement.

> Ce chapitre couvre une approche simple, adaptée au projet :  
> - un registry privé sur le réseau **VLAN 12** (VM/Services)  
> - push d’images depuis un poste/VM de build  
> - pull par K3s lors du déploiement applicatif

---

## 1) Choix de l’emplacement du registry

### 1.1 Option retenue (simple et efficace)
Héberger le registry sur une machine accessible depuis le cluster K3s, 
par exemple :
- soit directement sur `k3s-server` (VM1 : `10.0.12.11`)
- soit sur une VM dédiée (si vous voulez isoler)

Dans ce guide : **registry sur `k3s-server` (10.0.12.11)** pour limiter 
la complexité.

---

## 2) Installation Docker (sur la machine qui héberge le registry)

> Ici : sur `k3s-server` (10.0.12.11)

### 2.1 Installation Docker (méthode officielle Debian/Ubuntu)
Sur `k3s-server` :

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo 
"$ID")/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo   "deb [arch=$(dpkg --print-architecture) 
signed-by=/etc/apt/keyrings/docker.gpg] 
https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")   $(. 
/etc/os-release; echo "$VERSION_CODENAME") stable" |   sudo tee 
/etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io 
docker-buildx-plugin docker-compose-plugin
```

Activer Docker :

```bash
sudo systemctl enable --now docker
docker --version
```

Optionnel : ajouter l’utilisateur courant au groupe docker (évite sudo) 
:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## 3) Déployer un registry privé (conteneur Docker)

### 3.1 Lancer le registry
Sur `k3s-server` :

```bash
sudo docker run -d --restart=always --name registry   -p 5000:5000   
registry:2
```

### 3.2 Vérifier que le registry répond
```bash
curl -I http://10.0.12.11:5000/v2/
```

Résultat attendu :
- réponse HTTP `200` ou `401` (selon config) indiquant que `/v2/` 
existe.

---

## 4) Autoriser le pull depuis K3s (registry en HTTP)

Par défaut, Docker/Kubernetes préfèrent HTTPS.  
Dans un réseau local de lab (projet), on peut utiliser un registry HTTP 
avec configuration “insecure”.

### 4.1 Configurer K3s pour accepter le registry local
Sur `k3s-server` :

```bash
sudo mkdir -p /etc/rancher/k3s
cat <<'EOF' | sudo tee /etc/rancher/k3s/registries.yaml
mirrors:
  "10.0.12.11:5000":
    endpoint:
      - "http://10.0.12.11:5000"
EOF
```

Redémarrer K3s :

```bash
sudo systemctl restart k3s
```

Sur chaque worker (`k3s-worker-1` et `k3s-worker-2`), faire la même 
chose puis :

```bash
sudo systemctl restart k3s-agent
```

---

## 5) Build + tag + push des images

### 5.1 Convention de nommage
On tag les images pour le registry local :
- `10.0.12.11:5000/web:latest`
- `10.0.12.11:5000/db:latest` (uniquement si vous construisez une image 
DB custom)

### 5.2 Exemple : build image Web
Depuis un poste/VM qui contient le code et un `Dockerfile` :

```bash
docker build -t 10.0.12.11:5000/web:latest .
docker push 10.0.12.11:5000/web:latest
```

### 5.3 Vérifier que l’image est bien push
Depuis le host du registry :

```bash
curl http://10.0.12.11:5000/v2/_catalog
```

Vous devez voir `web` (et `db` si pushée).

---

## 6) Test de pull depuis K3s (sanity check)

Sur `k3s-server`, tester un pod simple qui tire l’image :

```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: registry-test
spec:
  containers:
    - name: web
      image: 10.0.12.11:5000/web:latest
      imagePullPolicy: Always
  restartPolicy: Never
EOF
```

Vérifier :

```bash
sudo kubectl get pod registry-test
sudo kubectl describe pod registry-test
```

Puis supprimer :

```bash
sudo kubectl delete pod registry-test
```

---

## 7) Points de contrôle (checklist)
- [ ] Docker installé et service actif sur le host registry
- [ ] Registry `registry:2` lancé sur `10.0.12.11:5000`
- [ ] `registries.yaml` appliqué sur server + workers
- [ ] Images taggées `10.0.12.11:5000/...` et push OK
- [ ] K3s arrive à pull une image depuis le registry local

---

## Étape suivante
Chapitre **08 — Déploiement applicatif** : déployer Web + BDD sur K3s, 
stockage persistant si nécessaire, et exposition via **Traefik** 
(Ingress/reverse proxy).

