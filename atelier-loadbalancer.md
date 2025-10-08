# Atelier Pratique : Configuration LoadBalancer 

## 🎯 Objectifs (5 min)
- Configurer et tester un LoadBalancer 
- Résoudre un problème courant

## 📋 Prérequis
- Cluster K3s/RKE2 géré par Rancher
- kubectl configuré
- Plage IP disponible (ex: 192.168.1.100-110)

---

## 📝 Étape 1 : Installation du LoadBalancer

* Installer MetalLB en tant que load balancer :
```
 kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```
* Attendre que les deux pods soit au statut running : kubectl get pods -n metallb-system
* Configurer la plage d'adresse IP disponible pour votre load balancer (adapter les plages d'adresse IP avec vos adresses IP):
```
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.26.90.100-10.26.90.110  
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
EOF
```
* Vous n'avez plus qu'à tester en passant à l'étape 2

---

## 📝 Étape 2 : Déploiement application test (10 min)

### Application Nginx complète
```yaml
# nginx-lb-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

### Déployer et vérifier
```bash
kubectl apply -f nginx-lb-demo.yaml

# Surveiller l'attribution d'IP (attendre quelques secondes)
kubectl get svc nginx-loadbalancer -w
```

**✅ Validation** : Service obtient une EXTERNAL-IP

---

## 📝 Étape 3 : Test et validation (15 min)

### Test de connectivité
```bash
# Récupérer l'IP externe
EXTERNAL_IP=$(kubectl get svc nginx-loadbalancer -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "External IP: $EXTERNAL_IP"

# Tester l'accès
curl http://$EXTERNAL_IP
```

### Test de répartition de charge
```bash
# Test rapide de load balancing
for i in {1..5}; do curl -s http://$EXTERNAL_IP | grep -o "Welcome to nginx" && echo " - Test $i OK"; done
```

**✅ Validation** : Application accessible via IP externe

---

## 📝 Étape 4 : Configuration avancée (10 min)

### Service avec IP fixe
```yaml
# nginx-fixed-ip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-fixed-ip
  annotations:
    # Spécifier une IP spécifique (adapter selon votre plage)
    klipper.cattle.io/external-ip: "192.168.1.105"
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 8080
    targetPort: 80
  type: LoadBalancer
```

### Appliquer et tester
```bash
kubectl apply -f nginx-fixed-ip.yaml

# Vérifier l'IP assignée
kubectl get svc nginx-fixed-ip

# Tester sur le nouveau port
curl http://192.168.1.105:8080
```

**✅ Validation** : Service utilise l'IP spécifiée

---

## 📝 Étape 5 : Résolution de problème (10 min)

### Simuler un problème courant
```bash
# Créer un service avec une IP non disponible
kubectl patch svc nginx-fixed-ip -p '{"metadata":{"annotations":{"klipper.cattle.io/external-ip":"192.168.1.999"}}}'
```

### Diagnostic rapide
```bash
# Vérifier le statut du service
kubectl describe svc nginx-fixed-ip

# Vérifier les événements
kubectl get events | grep nginx-fixed-ip | tail -5

```

### Résolution
```bash
# Corriger avec une IP valide
kubectl patch svc nginx-fixed-ip -p '{"metadata":{"annotations":{"klipper.cattle.io/external-ip":"192.168.1.106"}}}'

# Vérifier la correction
kubectl get svc nginx-fixed-ip
curl http://192.168.1.106:8080
```

**✅ Validation** : Problème identifié et résolu

---

## 📝 Étape 6 : Nettoyage (5 min)

```bash
# Supprimer les ressources
kubectl delete svc nginx-loadbalancer nginx-fixed-ip
kubectl delete deployment nginx-demo

# Vérifier le nettoyage
kubectl get svc | grep LoadBalancer
```

**✅ Validation** : Environnement nettoyé

---

## 🎓 Synthèse et points clés (5 min)

### Ce que nous avons appris :
1. **Klipper LB** fonctionne automatiquement avec K3s/RKE2
2. **Attribution automatique** d'IP depuis la plage disponible
3. **Pods svclb** créés automatiquement sur chaque nœud
4. **Diagnostic** via describe et events

### Commandes essentielles :
```bash
# Statut des LoadBalancers
kubectl get svc --all-namespaces | grep LoadBalancer

# Pods Klipper actifs  
kubectl get pods -n kube-system | grep svclb

# Test rapide de connectivité
curl http://$(kubectl get svc SERVICE_NAME -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### Cas d'usage typiques :
- **Développement** : Exposition rapide d'applications
- **Edge computing** : LoadBalancing simple sur site
- **Démonstrations** : Setup rapide sans cloud provider

### Limitations à connaître :
- Limité au réseau local des nœuds
- Pas de features avancées (SSL termination, etc.)
- Une IP par service

---

## ✅ Validation finale

**L'atelier est réussi si :**
- [ ] Service LoadBalancer déployé et accessible
- [ ] IP externe attribuée automatiquement
- [ ] Test de connectivité réussi
- [ ] Problème diagnostiqué et résolu
- [ ] Nettoyage effectué

**Durée réelle :** 45-60 minutes selon le niveau des participants

---

## 📖 Pour aller plus loin

### Documentation utile :
- [K3s ServiceLB](https://docs.k3s.io/networking#servicelb)
- [Rancher LoadBalancer](https://rancher.com/docs/rancher/v2.6/en/k8s-in-rancher/load-balancers-and-ingress/)

### Alternatives à explorer :
- **MetalLB** : Plus de fonctionnalités avancées
- **Ingress Controllers** : Pour HTTP/HTTPS avec domaines
- **Cloud LoadBalancers** : Pour les environnements cloud
