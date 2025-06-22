# Atelier Rancher : Créer différents types de services pour exposer une application

**Durée :** 30 minutes  
**Niveau :** Intermédiaire  
**Prérequis :** Accès à un cluster Rancher fonctionnel

## Objectifs pédagogiques

À la fin de cet atelier, vous saurez :
- Créer et configurer différents types de services Kubernetes via Rancher
- Comprendre les cas d'usage de chaque type de service
- Tester l'accessibilité des applications exposées
- Utiliser l'interface Rancher pour gérer les services

## Architecture de l'atelier

Nous déploierons une application web simple (nginx) et l'exposerons via 4 types de services différents :
1. **ClusterIP** - Accès interne uniquement
2. **NodePort** - Accès externe via port sur les nœuds
3. **LoadBalancer** - Accès externe via load balancer
4. **Ingress** - Accès externe via nom de domaine

---

## Étape 1 : Préparation de l'environnement (5 minutes)

### 1.1 Connexion à Rancher
1. Connectez-vous à votre interface Rancher
2. Sélectionnez votre cluster de travail
3. Naviguez vers **Workloads > Deployments**

### 1.2 Création du namespace
1. Cliquez sur **Create Namespace**
2. Nom : `workshop-services`
3. Cliquez sur **Create**

### 1.3 Déploiement de l'application de test
1. Dans **Workloads > Deployments**, cliquez sur **Create**
2. Configurez le déploiement :
   - **Name** : `nginx-demo`
   - **Namespace** : `workshop-services`
   - **Container Image** : `nginx:latest`
   - **Port** : `80`
   - **Replicas** : `3`
3. Cliquez sur **Create**

**Vérification :** Attendez que les 3 pods soient en état "Running"

---

## Étape 2 : Service ClusterIP (5 minutes)

Le service ClusterIP expose l'application uniquement à l'intérieur du cluster.

### 2.1 Création du service ClusterIP
1. Naviguez vers **Service Discovery > Services**
2. Cliquez sur **Create**
3. Configuration :
   - **Name** : `nginx-clusterip`
   - **Namespace** : `workshop-services`
   - **Service Type** : `ClusterIP`
   - **Target Workload** : Sélectionnez `nginx-demo`
   - **Port Mapping** :
     - Port : `80`
     - Target Port : `80`
     - Protocol : `TCP`
4. Cliquez sur **Create**

### 2.2 Test du service ClusterIP
1. Naviguez vers **Workloads > Pods**
2. Cliquez sur un pod nginx-demo, puis **Execute Shell**
3. Testez la connectivité interne :
```bash
curl nginx-clusterip.workshop-services.svc.cluster.local
```

**Résultat attendu :** Page d'accueil nginx

---

## Étape 3 : Service NodePort (5 minutes)

Le service NodePort expose l'application sur un port spécifique de chaque nœud du cluster.

### 3.1 Création du service NodePort
1. Dans **Services**, cliquez sur **Create**
2. Configuration :
   - **Name** : `nginx-nodeport`
   - **Namespace** : `workshop-services`
   - **Service Type** : `NodePort`
   - **Target Workload** : Sélectionnez `nginx-demo`
   - **Port Mapping** :
     - Port : `80`
     - Target Port : `80`
     - Node Port : `30080` (ou laissez vide pour auto-attribution)
     - Protocol : `TCP`
3. Cliquez sur **Create**

### 3.2 Test du service NodePort
1. Récupérez l'IP d'un nœud : **Cluster > Nodes**
2. Testez l'accès externe :
```bash
curl http://<NODE_IP>:30080
```

**Résultat attendu :** Page d'accueil nginx accessible depuis l'extérieur

---

## Étape 4 : Service LoadBalancer (5 minutes)

Le service LoadBalancer utilise un load balancer externe (cloud provider).

### 4.1 Création du service LoadBalancer
1. Dans **Services**, cliquez sur **Create**
2. Configuration :
   - **Name** : `nginx-loadbalancer`
   - **Namespace** : `workshop-services`
   - **Service Type** : `LoadBalancer`
   - **Target Workload** : Sélectionnez `nginx-demo`
   - **Port Mapping** :
     - Port : `80`
     - Target Port : `80`
     - Protocol : `TCP`
3. Cliquez sur **Create**

### 4.2 Vérification du LoadBalancer
1. Dans la liste des services, vérifiez l'attribution d'une IP externe
2. Si disponible, testez l'accès :
```bash
curl http://<EXTERNAL_IP>
```

**Note :** L'IP externe peut prendre quelques minutes à être attribuée selon votre environnement cloud.

---

## Étape 5 : Ingress Controller (8 minutes)

L'Ingress permet l'accès externe via des noms de domaine et offre des fonctionnalités avancées (SSL, routing).

### 5.1 Vérification de l'Ingress Controller
1. Naviguez vers **Service Discovery > Ingresses**
2. Vérifiez qu'un Ingress Controller est installé (nginx-ingress, traefik, etc.)

### 5.2 Création de l'Ingress
1. Cliquez sur **Create**
2. Configuration :
   - **Name** : `nginx-ingress`
   - **Namespace** : `workshop-services`
   - **Rules** :
     - **Host** : `nginx-demo.local` (ou votre domaine)
     - **Path** : `/`
     - **Target Service** : `nginx-clusterip`
     - **Port** : `80`
3. Dans **Certificates**, ajoutez si nécessaire un certificat SSL
4. Cliquez sur **Create**

### 5.3 Configuration DNS locale (pour test)
Ajoutez dans votre fichier `/etc/hosts` (Linux/Mac) ou `C:\Windows\System32\drivers\etc\hosts` (Windows) :
```
<INGRESS_IP> nginx-demo.local
```

### 5.4 Test de l'Ingress
```bash
curl http://nginx-demo.local
```

---

## Étape 6 : Comparaison et nettoyage (2 minutes)

### 6.1 Comparaison des services

| Type | Accessibilité | Use Case | IP/Port |
|------|---------------|----------|---------|
| ClusterIP | Interne uniquement | Communication inter-services | IP cluster interne |
| NodePort | Externe via nœuds | Test, développement | IP nœud + port spécifique |
| LoadBalancer | Externe via LB | Production cloud | IP externe attribuée |
| Ingress | Externe via domaine | Production, SSL, routing | Nom de domaine |

### 6.2 Vérification finale
1. Listez tous vos services : **Service Discovery > Services**
2. Vérifiez leur état et configuration
3. Testez l'accès selon le type de service

### 6.3 Nettoyage (optionnel)
Pour nettoyer l'environnement :
1. Supprimez les services créés
2. Supprimez le déploiement `nginx-demo`
3. Supprimez le namespace `workshop-services`

---

## Points clés à retenir

### Bonnes pratiques
- **ClusterIP** : Par défaut pour les services internes
- **NodePort** : Éviter en production, réserver au développement
- **LoadBalancer** : Idéal pour les services publics en cloud
- **Ingress** : Solution recommandée pour l'exposition HTTP/HTTPS

### Sécurité
- Utilisez des Network Policies pour contrôler le trafic
- Configurez SSL/TLS sur les Ingress
- Limitez l'exposition des services sensibles

### Monitoring
- Surveillez les métriques des services dans Rancher
- Configurez des alertes sur la disponibilité
- Utilisez les logs pour le troubleshooting

---

## Questions de validation

1. Quel type de service utiliseriez-vous pour une base de données accessible uniquement par votre application ?
2. Comment exposer une API REST avec SSL et un nom de domaine personnalisé ?
3. Quelle est la différence entre un service LoadBalancer et un Ingress ?
4. Comment tester la connectivité d'un service ClusterIP ?

---

## Ressources complémentaires

- [Documentation Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Guide Rancher Services](https://rancher.com/docs/rancher/v2.x/en/k8s-in-rancher/workloads-and-pods/load-balancing-and-ingress/)
- [Best Practices Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

**Félicitations ! Vous maîtrisez maintenant les différents types de services Kubernetes dans Rancher.**
