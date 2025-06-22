# Atelier configuration de la connexion avec un registre privé

# Atelier Rancher - Configuration d'un registre privé Docker Hub

## Objectifs pédagogiques
- Créer un registre privé sur Docker Hub
- Configurer l'authentification dans Rancher
- Déployer une application utilisant le registre privé
- Gérer les secrets d'authentification

## Prérequis
- Compte Docker Hub
- Accès à une instance Rancher
- Cluster Kubernetes configuré dans Rancher
- Droits d'administration sur le projet/namespace

---

## Étape 1 : Création du registre privé sur Docker Hub

### 1.1 Connexion à Docker Hub
1. Rendez-vous sur [hub.docker.com](https://hub.docker.com)
2. Connectez-vous avec votre compte Docker Hub
3. Si vous n'avez pas de compte, créez-en un

### 1.2 Création d'un dépôt privé
1. Cliquez sur **"Create Repository"**
2. Remplissez les informations :
   - **Name** : `mon-app-privee` (exemple)
   - **Description** : `Application privée pour l'atelier Rancher`
   - **Visibility** : Sélectionnez **"Private"**
3. Cliquez sur **"Create"**

### 1.3 Préparation d'une image de test
```bash
# Créer un Dockerfile simple
cat > Dockerfile << EOF
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# Créer un fichier index.html
cat > index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Application Privée</title>
</head>
<body>
    <h1>Bienvenue dans l'application privée !</h1>
    <p>Cette image provient du registre privé Docker Hub</p>
</body>
</html>
EOF

# Build et push de l'image
docker build -t votre-username/mon-app-privee:v1.0 .
docker login
docker push votre-username/mon-app-privee:v1.0
```

---

## Étape 2 : Configuration du registre dans Rancher

### 2.1 Accès à l'interface Rancher
1. Connectez-vous à votre interface Rancher
2. Sélectionnez votre cluster
3. Naviguez vers le projet où vous souhaitez configurer le registre

### 2.2 Ajout du registre privé
1. Dans le menu de gauche, allez dans **"Storage"** > **"Secrets"**
2. Cliquez sur **"Add Secret"**
3. Sélectionnez **"Registry"** comme type de secret

### 2.3 Configuration du secret de registre
Remplissez les informations suivantes :
- **Name** : `docker-hub-private`
- **Registry Domain Name** : `index.docker.io`
- **Username** : Votre nom d'utilisateur Docker Hub
- **Password** : Votre mot de passe Docker Hub ou Access Token
- **Namespace** : Sélectionnez le namespace approprié

> **Note** : Il est recommandé d'utiliser un Access Token plutôt qu'un mot de passe pour des raisons de sécurité.

### 2.4 Création d'un Access Token Docker Hub (Recommandé)
1. Sur Docker Hub, allez dans **Account Settings** > **Security**
2. Cliquez sur **"New Access Token"**
3. Donnez un nom au token : `rancher-access`
4. Sélectionnez les permissions : **Read, Write, Delete**
5. Copiez le token généré
6. Utilisez ce token comme mot de passe dans Rancher

---

## Étape 3 : Test du déploiement avec le registre privé

### 3.1 Création d'un déploiement de test
1. Dans Rancher, allez dans **"Workloads"** > **"Deployments"**
2. Cliquez sur **"Deploy"**
3. Configurez le déploiement :

#### Configuration de base
- **Name** : `app-privee-test`
- **Docker Image** : `votre-username/mon-app-privee:v1.0`
- **Pull Policy** : `Always`

#### Configuration du secret
- Dans la section **"Secrets"**
- **Image Pull Secrets** : Sélectionnez `docker-hub-private`

#### Configuration réseau
- **Port Mapping** :
  - **Port Name** : `http`
  - **Expose Port** : `80`
  - **Protocol** : `TCP`
  - **As a** : `ClusterIP`

### 3.2 Création du service
1. Activez **"Add port or expose a port"**
2. Configurez :
   - **Service Type** : `ClusterIP` ou `NodePort` selon vos besoins
   - **Target Port** : `80`

---

## Étape 4 : Vérification et tests

### 4.1 Vérification du déploiement
1. Vérifiez que le pod se lance correctement
2. Consultez les logs pour s'assurer qu'il n'y a pas d'erreurs d'authentification
3. Vérifiez le statut du déploiement dans **"Workloads"**

### 4.2 Test d'accès à l'application
```bash
# Si vous avez configuré un NodePort
curl http://<node-ip>:<nodeport>

# Ou utilisez kubectl port-forward
kubectl port-forward deployment/app-privee-test 8080:80
curl http://localhost:8080
```

### 4.3 Vérification des secrets
```bash
# Vérifier que le secret est bien créé
kubectl get secrets -n <votre-namespace>

# Examiner le secret (base64 encodé)
kubectl describe secret docker-hub-private -n <votre-namespace>
```

---

## Étape 5 : Configuration avancée et bonnes pratiques

### 5.1 Utilisation dans un fichier YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-privee-yaml
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-privee
  template:
    metadata:
      labels:
        app: app-privee
    spec:
      imagePullSecrets:
      - name: docker-hub-private
      containers:
      - name: app-container
        image: votre-username/mon-app-privee:v1.0
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "128Mi"
            cpu: "100m"
          requests:
            memory: "64Mi"
            cpu: "50m"
```

### 5.2 Gestion des versions et tags
```bash
# Bonnes pratiques pour le versioning
docker tag votre-username/mon-app-privee:v1.0 votre-username/mon-app-privee:latest
docker push votre-username/mon-app-privee:latest

# Utilisation de tags sémantiques
docker tag votre-username/mon-app-privee:v1.0 votre-username/mon-app-privee:1.0.0
docker push votre-username/mon-app-privee:1.0.0
```

### 5.3 Sécurisation du registre
- **Utilisez des Access Tokens** au lieu de mots de passe
- **Limitez les permissions** des tokens
- **Rotation régulière** des credentials
- **Audit des accès** via les logs Docker Hub

---

## Étape 6 : Dépannage courant

### 6.1 Erreurs d'authentification
**Symptôme** : `ImagePullBackOff` ou `ErrImagePull`

**Solutions** :
```bash
# Vérifier les événements du pod
kubectl describe pod <pod-name>

# Tester l'authentification manuellement
docker login
docker pull votre-username/mon-app-privee:v1.0

# Vérifier le secret
kubectl get secret docker-hub-private -o yaml
```

### 6.2 Problèmes de configuration réseau
**Symptôme** : Pod démarré mais service inaccessible

**Solutions** :
```bash
# Vérifier les services
kubectl get svc

# Tester la connectivité interne
kubectl exec -it <pod-name> -- wget -O- http://localhost:80

# Vérifier les endpoints
kubectl get endpoints
```

### 6.3 Permissions insuffisantes
**Symptôme** : Impossible de créer le secret ou le déploiement

**Solutions** :
- Vérifier les RBAC dans Rancher
- S'assurer d'avoir les droits sur le namespace
- Contacter l'administrateur Rancher si nécessaire

---

## Questions de validation

### Questions théoriques
1. Quelle est la différence entre un registre public et privé ?
2. Pourquoi utiliser des Access Tokens plutôt que des mots de passe ?
3. Que se passe-t-il si on oublie de configurer `imagePullSecrets` ?

### Exercices pratiques
1. Créez un second déploiement avec une version différente de votre image
2. Configurez un registre privé avec un autre fournisseur (ex: AWS ECR)
3. Mettez en place une stratégie de rolling update

---

## Ressources complémentaires

### Documentation officielle
- [Rancher Documentation - Private Registries](https://rancher.com/docs/)
- [Kubernetes - Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Docker Hub - Access Tokens](https://docs.docker.com/docker-hub/access-tokens/)

### Commandes utiles
```bash
# Gérer les secrets via kubectl
kubectl create secret docker-registry <secret-name> \
  --docker-server=<server> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email>

# Examiner la configuration Docker
docker system info | grep -i registry

# Nettoyer les ressources de test
kubectl delete deployment app-privee-test
kubectl delete secret docker-hub-private
```


