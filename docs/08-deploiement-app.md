# 08 — Déploiement de l’application sur K3s (Web + BDD) + exposition via 
Traefik

Objectif : déployer l’application en mode **multi-services** sur K3s :
- un service **Base de données** (persistant)
- un service **Web**
- exposition externe via **Traefik** (Ingress = reverse proxy)

Hypothèses projet (à adapter à votre app) :
- les images sont stockées dans le registry local : `10.0.12.11:5000`
- l’app Web écoute sur le port `80` dans le conteneur
- la base de données est de type MariaDB/MySQL (exemple standard)

> Ce guide reste volontairement **générique et reproductible** : il 
suffit d’ajuster les variables (nom image, variables d’environnement).

---

## 1) Préparer un namespace dédié

Sur `k3s-server` :

```bash
sudo kubectl create namespace app
sudo kubectl get ns
```

---

## 2) Déployer la base de données (avec stockage persistant)

### 2.1 Créer un Secret (mot de passe DB)
⚠️ Ne jamais hardcoder un mot de passe dans un manifeste public.  
Dans un repo public, on met une valeur d’exemple et on indique de la 
remplacer.

Créer un secret (exemple) :

```bash
sudo kubectl -n app create secret generic db-secret   
--from-literal=MYSQL_ROOT_PASSWORD='ChangeMeRoot!'   
--from-literal=MYSQL_DATABASE='appdb'   
--from-literal=MYSQL_USER='appuser'   
--from-literal=MYSQL_PASSWORD='ChangeMeUser!'
```

Vérifier :

```bash
sudo kubectl -n app get secret db-secret
```

### 2.2 Stockage persistant
K3s fournit souvent un storageclass par défaut via `local-path`.  
Vérifier :

```bash
sudo kubectl get storageclass
```

Si une storageclass `local-path` est marquée `(default)`, vous pouvez 
créer un PVC simple.

Créer un PVC :

```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
  namespace: app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
EOF
```

Vérifier :

```bash
sudo kubectl -n app get pvc
```

### 2.3 Déploiement MariaDB (Stateful simple)
```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
        - name: mariadb
          image: mariadb:11
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_PASSWORD
          volumeMounts:
            - name: db-data
              mountPath: /var/lib/mysql
      volumes:
        - name: db-data
          persistentVolumeClaim:
            claimName: db-pvc
EOF
```

Créer un Service interne DB :

```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: mariadb
  namespace: app
spec:
  selector:
    app: mariadb
  ports:
    - port: 3306
      targetPort: 3306
EOF
```

Vérifier :

```bash
sudo kubectl -n app get deploy,svc,pods
```

---

## 3) Déployer le service Web (image depuis registry local)

### 3.1 Déployer Web
Remplacer l’image si besoin : `10.0.12.11:5000/web:latest`

```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: 10.0.12.11:5000/web:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          env:
            - name: DB_HOST
              value: "mariadb.app.svc.cluster.local"
            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_DATABASE
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: MYSQL_PASSWORD
EOF
```

### 3.2 Service Web interne
```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: app
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
EOF
```

Vérifier :

```bash
sudo kubectl -n app get deploy,svc,pods
```

---

## 4) Exposer l’application via Traefik (Ingress = reverse proxy)

### 4.1 Principe
Traefik reçoit les requêtes HTTP/HTTPS et les redirige vers 
`service/web` dans le namespace `app`.

### 4.2 Ingress minimal (HTTP)
Choisir un hostname (exemple) : `app.footstore.local`  
> Il doit être résolu (DNS local, /etc/hosts, ou autre).

Créer l’ingress :

```bash
cat <<'EOF' | sudo kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: app
spec:
  rules:
    - host: app.footstore.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF
```

Vérifier :

```bash
sudo kubectl -n app get ingress
```

---

## 5) Tests de validation (fonctionnel)

### 5.1 Vérifier que les pods tournent
```bash
sudo kubectl -n app get pods -o wide
```

### 5.2 Logs web
```bash
sudo kubectl -n app logs deploy/web --tail=100
```

### 5.3 Test HTTP depuis le cluster
```bash
sudo kubectl -n app run curltest --image=curlimages/curl:8.5.0 -it --rm 
--   curl -I http://web
```

### 5.4 Test externe (poste admin)
Sur le poste admin :
- ajouter une entrée `hosts` ou DNS pour `app.footstore.local`
- ouvrir : `http://app.footstore.local`

---

## 6) Nettoyage (si besoin)
Supprimer le namespace (supprime tout dedans) :

```bash
sudo kubectl delete namespace app
```

---

## 7) Checklist de fin de chapitre
- [ ] namespace `app` créé
- [ ] DB déployée + PVC bound
- [ ] Service DB interne OK
- [ ] Web déployé depuis registry local
- [ ] Service Web interne OK
- [ ] Ingress Traefik en place
- [ ] Accès web OK depuis poste admin

---

## Étape suivante
Chapitre **09 — Sécurité** : durcissement (pfSense règles, SSH, secrets, 
sauvegardes, journalisation) et alignement avec recommandations de 
bonnes pratiques.

