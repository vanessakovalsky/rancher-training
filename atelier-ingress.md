# Atelier Rancher : Créer un Ingress multi-services avec TLS

**Durée :** 60 minutes  
**Niveau :** Intermédiaire  
**Prérequis :** Cluster Kubernetes opérationnel dans Rancher, accès admin

## Objectifs pédagogiques

À la fin de cet atelier, les participants sauront :
- Déployer plusieurs services dans Kubernetes via Rancher
- Configurer un Ingress Controller
- Créer un Ingress avec routage multi-services
- Implémenter la terminaison TLS avec certificats
- Diagnostiquer les problèmes courants d'Ingress

---

## Phase 1 : Préparation de l'environnement (10 minutes)

### 1.1 Vérification du cluster
1. Connectez-vous à l'interface Rancher
2. Sélectionnez votre cluster de travail
3. Vérifiez que l'Ingress Controller est installé :
   ```bash
   kubectl get pods -n ingress-nginx
   ```
   Si absent, installez-le via **Cluster > Apps > nginx-ingress**

### 1.2 Création du namespace
```bash
kubectl create namespace workshop-ingress
```

Ou via l'interface Rancher :
- **Cluster > Projects/Namespaces > Create Namespace**
- Nom : `workshop-ingress`

---

## Phase 2 : Déploiement des services de démonstration (15 minutes)

### 2.1 Service Application Web (Frontend)
Créez le fichier `frontend-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: workshop-ingress
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-content
          mountPath: /usr/share/nginx/html
        resources:
          limits:
            cpu: 1m
            memory: 128Mi
          requests:
            cpu: 1m
            memory: 128Mi
      volumes:
      - name: html-content
        configMap:
          name: frontend-content
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-content
  namespace: workshop-ingress
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Frontend App</title></head>
    <body>
      <h1>Application Frontend</h1>
      <p>Service accessible via Ingress</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: workshop-ingress
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### 2.2 Service API (Backend)
Créez le fichier `api-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: workshop-ingress
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=API Response from $(HOSTNAME)"
          ports:
            - containerPort: 5678
          env:
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          resources:
            limits:
              cpu: 1m
              memory: 128Mi
            requests:
              cpu: 1m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: workshop-ingress
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 5678
  type: ClusterIP

```

### 2.3 Déploiement
```bash
kubectl apply -f frontend-deployment.yaml
kubectl apply -f api-deployment.yaml
```

**Vérification :**
```bash
kubectl get pods,svc -n workshop-ingress
```

---

## Phase 3 : Configuration des certificats TLS (10 minutes)

### 3.1 Génération d'un certificat auto-signé
```bash
# Générer la clé privée
openssl genrsa -out tls.key 2048

# Générer le certificat
openssl req -new -x509 -key tls.key -out tls.crt -days 365 -subj "/CN=workshop.local"
```

### 3.2 Création du Secret TLS
```bash
kubectl create secret tls workshop-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n workshop-ingress
```

Ou via l'interface Rancher :
- **Storage > Secrets > Create**
- Type : **TLS Certificate**
- Namespace : `workshop-ingress`
- Name : `workshop-tls-secret`

### 3.3 Alternative : Certificat Let's Encrypt (bonus)
Pour un environnement de production, utilisez cert-manager :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@exemple.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

## Phase 4 : Création de l'Ingress multi-services (15 minutes)

### 4.1 Configuration de base
Créez le fichier `multi-service-ingress.yaml` :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: workshop-ingress
  namespace: workshop-ingress
  annotations:
    # Configuration NGINX Ingress Controller
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Taille maximale du body (pour uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Configuration CORS (si nécessaire)
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - workshop.local
    secretName: workshop-tls-secret
  rules:
  - host: workshop.local
    http:
      paths:
      # Route pour le frontend (racine)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      # Route pour l'API
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

### 4.2 Déploiement de l'Ingress
```bash
kubectl apply -f multi-service-ingress.yaml
```

### 4.3 Vérification de la configuration
```bash
# Statut de l'Ingress
kubectl get ingress -n workshop-ingress

# Détails de l'Ingress
kubectl describe ingress workshop-ingress -n workshop-ingress

# Logs du contrôleur Ingress
kubectl logs -n kube-system $(kubectl get pods -n kube-system | grep rke2-ingress-nginx-controller | awk '{print $1}')
```

---

## Phase 5 : Test et validation (8 minutes)

### 5.1 Configuration DNS locale
Ajoutez dans `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows) :
```
<IP_DU_LOAD_BALANCER> workshop.local
```

Pour trouver l'IP :
```bash
kubectl get svc -n ingress-nginx
```

### 5.2 Tests de connectivité
```bash
# Test HTTPS du frontend
curl -k https://workshop.local/

# Test HTTPS de l'API
curl -k https://workshop.local/api

# Vérification du certificat
openssl s_client -connect workshop.local:443 -servername workshop.local
```

### 5.3 Tests via navigateur
1. Ouvrez `https://workshop.local` → Frontend
2. Ouvrez `https://workshop.local/api` → API Response

---

## Phase 6 : Configuration avancée et dépannage (7 minutes)

### 6.1 Ajout d'un troisième service (Dashboard Admin)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-dashboard
  namespace: workshop-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: admin
  template:
    metadata:
      labels:
        app: admin
    spec:
      containers:
      - name: admin
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
  namespace: workshop-ingress
spec:
  selector:
    app: admin
  ports:
  - port: 80
    targetPort: 80
```

### 6.2 Mise à jour de l'Ingress avec authentification
```yaml
# Ajout dans les annotations de l'Ingress
nginx.ingress.kubernetes.io/auth-type: basic
nginx.ingress.kubernetes.io/auth-secret: basic-auth
nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - Admin Area'

# Nouvelle route dans spec.rules[0].http.paths
- path: /admin
  pathType: Prefix
  backend:
    service:
      name: admin-service
      port:
        number: 80
```

### 6.3 Création du secret d'authentification
```bash
htpasswd -c auth admin
kubectl create secret generic basic-auth --from-file=auth -n workshop-ingress
```

### 6.4 Dépannage courant
**Commandes de diagnostic :**
```bash
# État des pods Ingress Controller
kubectl get pods -n ingress-nginx

# Configuration NGINX générée
kubectl exec -n ingress-nginx <ingress-pod-name> -- cat /etc/nginx/nginx.conf

# Événements du cluster
kubectl get events -n workshop-ingress --sort-by=.metadata.creationTimestamp
```

**Problèmes fréquents :**
- **503 Service Unavailable** → Vérifier que les services et pods backend sont opérationnels
- **404 Not Found** → Vérifier la configuration des paths dans l'Ingress
- **Certificate errors** → Vérifier que le secret TLS est correctement créé et référencé

---

## Récapitulatif et bonnes pratiques

### Points clés abordés
1. ✅ Déploiement de services multiples
2. ✅ Configuration Ingress avec routage par path
3. ✅ Implémentation TLS avec certificats
4. ✅ Tests et validation
5. ✅ Dépannage et monitoring

### Bonnes pratiques de production
- Utilisez cert-manager pour les certificats automatiques
- Implémentez la surveillance avec Prometheus/Grafana
- Configurez des limits et requests sur vos déploiements
- Utilisez des namespaces séparés par environnement
- Documentez vos annotations Ingress
- Testez régulièrement vos certificats et leur renouvellement

### Ressources pour aller plus loin
- [Documentation officielle NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/)
- [Guide cert-manager pour Let's Encrypt](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/)
- [Monitoring Ingress avec Prometheus](https://docs.nginx.com/nginx-ingress-controller/logging-and-monitoring/prometheus/)
- [Configuration avancée : rate limiting, circuit breakers](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/)

