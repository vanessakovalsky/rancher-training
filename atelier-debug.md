# Atelier Rancher : Diagnostic et Résolution de Problèmes Courants
**Durée : 30 minutes | Niveau : Intermédiaire**

## Objectifs pédagogiques
À la fin de cet atelier, vous serez capable de :
- Identifier les sources de problèmes courants dans Rancher
- Utiliser les outils de diagnostic intégrés
- Appliquer une méthodologie structurée de résolution
- Résoudre les incidents les plus fréquents

## Prérequis
- Rancher Server déployé et accessible
- Accès administrateur à l'interface Rancher
- kubectl configuré (optionnel mais recommandé)
- Cluster Kubernetes opérationnel

---

## Phase 1 : Méthodologie de diagnostic (5 min)

### 1.1 Approche structurée STAR
**S**ymptômes - **T**ests - **A**nalyse - **R**ésolution

1. **Identifier les symptômes** : Que constate-t-on exactement ?
2. **Effectuer des tests** : Reproduire et isoler le problème
3. **Analyser les logs** : Examiner les journaux système
4. **Résoudre méthodiquement** : Appliquer la solution et vérifier

### 1.2 Sources d'information principales
- Interface Web Rancher
- Logs des composants Rancher
- Logs Kubernetes (kubelet, kube-proxy, etc.)
- Métriques et monitoring

---

## Phase 2 : Problèmes d'authentification et d'accès (8 min)

### Scénario 1 : Impossible de se connecter à Rancher

#### **Symptômes typiques**
- Page de connexion inaccessible
- Erreur "Connection refused"
- Certificats SSL invalides

#### **Diagnostic étape par étape**

**Étape 1 : Vérifier l'état du service Rancher**
```bash
# Vérifier les pods Rancher
kubectl get pods -n cattle-system

# Examiner les logs du pod Rancher
kubectl logs -n cattle-system deployment/rancher -f
```

**Étape 2 : Contrôler la connectivité réseau**
```bash
# Tester la connectivité
curl -k https://rancher.votre-domaine.com/ping

# Vérifier les services
kubectl get svc -n cattle-system
```

**Étape 3 : Examiner les certificats**
```bash
# Vérifier les secrets TLS
kubectl get secrets -n cattle-system | grep tls

# Détailler le certificat
kubectl describe secret tls-rancher-ingress -n cattle-system
```

#### **Solutions courantes**
1. **Redémarrer le pod Rancher** :
   ```bash
   kubectl rollout restart deployment/rancher -n cattle-system
   ```

2. **Regénérer les certificats** (si expirés) :
   ```bash
   kubectl delete secret tls-rancher-ingress -n cattle-system
   # Puis redéployer Rancher avec de nouveaux certificats
   ```

### Scénario 2 : Erreur "Unauthorized" après connexion

#### **Actions de diagnostic**
1. Vérifier dans Rancher UI : **Global > Security > Authentication**
2. Examiner les logs d'authentification :
   ```bash
   kubectl logs -n cattle-system deployment/rancher | grep -i auth
   ```

#### **Solution**
- Réinitialiser le mot de passe admin via bootstrap password
- Vérifier la configuration de l'Active Directory/LDAP

---

## Phase 3 : Problèmes de clusters (10 min)

### Scénario 3 : Cluster en état "Unavailable"

#### **Diagnostic**

**Étape 1 : Examiner l'état dans Rancher UI**
- Aller dans **Cluster Management**
- Cliquer sur le cluster problématique
- Observer les événements dans l'onglet "Events"

**Étape 2 : Vérifier les agents Rancher**
```bash
# Sur chaque nœud du cluster
docker ps | grep rancher

# Vérifier les logs de l'agent
docker logs <container-id-rancher-agent>
```

**Étape 3 : Contrôler la connectivité**
```bash
# Depuis les nœuds workers vers le server Rancher
curl -k https://rancher.votre-domaine.com/ping

# Vérifier les ports requis (80, 443, 6443)
telnet rancher.votre-domaine.com 443
```

#### **Solutions**
1. **Redémarrer l'agent Rancher** :
   ```bash
   docker restart <rancher-agent-container>
   ```

2. **Regénérer la commande d'inscription** :
   - Dans Rancher UI : Cluster > Registration
   - Copier et réexécuter la commande sur les nœuds

### Scénario 4 : Pods en état "Pending" ou "CrashLoopBackOff"

#### **Diagnostic**
```bash
# Examiner les pods problématiques
kubectl get pods --all-namespaces | grep -v Running

# Décrire un pod spécifique
kubectl describe pod <pod-name> -n <namespace>

# Consulter les logs
kubectl logs <pod-name> -n <namespace> --previous
```

#### **Causes et solutions courantes**

**Problème de ressources** :
```bash
# Vérifier les ressources des nœuds
kubectl describe nodes
kubectl top nodes
```
*Solution* : Augmenter les ressources ou ajuster les limits/requests

**Problèmes de réseau** :
```bash
# Tester la connectivité entre pods
kubectl run test-pod --image=busybox -it --rm -- /bin/sh
```

---

## Phase 4 : Problèmes de performance et monitoring (5 min)

### Scénario 5 : Rancher lent ou non-responsif

#### **Diagnostic de performance**

**Étape 1 : Vérifier les métriques système**
```bash
# Utilisation CPU/Mémoire des pods Rancher
kubectl top pods -n cattle-system

# Vérifier l'espace disque sur les nœuds
kubectl get nodes -o wide
```

**Étape 2 : Examiner les logs pour les erreurs**
```bash
kubectl logs -n cattle-system deployment/rancher | grep -i error
```

#### **Solutions d'optimisation**
1. **Augmenter les ressources Rancher** :
   ```bash
   kubectl patch deployment rancher -n cattle-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"rancher","resources":{"limits":{"memory":"4Gi","cpu":"2"}}}]}}}}'
   ```

2. **Nettoyer les anciens événements** :
   - Interface Rancher : Tools > Remove Orphaned Data

---

## Phase 5 : Exercice pratique guidé (2 min)

### Mise en situation
Un cluster apparaît comme "Disconnected" depuis 10 minutes. Les utilisateurs ne peuvent plus déployer d'applications.

### Votre mission
1. **Identifier** : Quels sont les symptômes visibles ?
2. **Diagnostiquer** : Quelle est la cause probable ?
3. **Résoudre** : Quelles actions entreprendre ?
4. **Vérifier** : Comment confirmer la résolution ?

### Méthodologie à appliquer
```
┌─ Symptômes observés
├─ Tests de connectivité  
├─ Analyse des logs
├─ Action corrective
└─ Validation du fix
```

---

## Checklist de résolution rapide

### 🔍 **Diagnostic initial (2 min)**
- [ ] État des clusters dans Rancher UI
- [ ] Logs récents dans l'interface
- [ ] Métriques de base (CPU, RAM, Disk)

### 🛠️ **Actions de première intervention (3 min)**
- [ ] Redémarrage des composants défaillants
- [ ] Vérification de la connectivité réseau
- [ ] Validation des certificats

### ✅ **Validation de la résolution (1 min)**
- [ ] Tous les clusters "Active"
- [ ] Déploiement test réussi
- [ ] Monitoring normal

---

## Ressources pour aller plus loin

### Commandes de diagnostic essentielles
```bash
# Vue d'ensemble du cluster
kubectl cluster-info

# États des nœuds
kubectl get nodes -o wide

# Événements récents
kubectl get events --sort-by='.lastTimestamp'

# Pods système critiques
kubectl get pods -n kube-system
kubectl get pods -n cattle-system
```

### Documentation officielle
- [Rancher Troubleshooting Guide](https://rancher.com/docs/rancher/v2.x/en/troubleshooting/)
- [Kubernetes Debugging Guide](https://kubernetes.io/docs/tasks/debug-application-cluster/)

### Outils recommandés
- **k9s** : Interface CLI interactive pour Kubernetes
- **Lens** : IDE Kubernetes avec monitoring intégré
- **Prometheus + Grafana** : Monitoring avancé

---

*Fin de l'atelier - Questions/Réponses et retours d'expérience*
