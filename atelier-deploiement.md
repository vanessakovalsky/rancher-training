# Atelier Rancher : Déployer une application web avec stratégie RollingUpdate
**Durée : 60 minutes**

## Objectifs pédagogiques
À la fin de cet atelier, les participants sauront :
- Comprendre le principe de la stratégie RollingUpdate
- Déployer une application web dans Rancher avec RollingUpdate
- Effectuer une mise à jour progressive
- Gérer un rollback simple

## Prérequis
- Accès à une instance Rancher configurée
- Namespace `workshop` pré-créé
- Connaissances de base de Kubernetes

---

## Étape 1 : Théorie express RollingUpdate (5 min)

### Principe
RollingUpdate = mise à jour progressive sans interruption de service

### Paramètres essentiels :
- **maxSurge** : Pods supplémentaires autorisés (défaut: 25%)
- **maxUnavailable** : Pods indisponibles maximum (défaut: 25%)

---

## Étape 2 : Déploiement initial (10 min)

### Création rapide du Deployment

1. **Navigation** : Workloads > Deployments > Create
2. **Configuration** :
   - **Name** : `webapp-demo`
   - **Namespace** : `workshop`
   - **Replicas** : `3`
   - **Container Image** : `nginx:1.20`
   - **Container Port** : `80`

3. **Stratégie RollingUpdate** (onglet Upgrading) :
   - **Update Strategy** : `RollingUpdate`
   - **Max Surge** : `1`
   - **Max Unavailable** : `1`

4. **Labels** :
   ```yaml
   app: webapp-demo
   version: v1.0
   ```

5. **Cliquez** sur **Create**

### Création du Service
1. **Service Discovery > Services > Create**
2. **Configuration** :
   - **Name** : `webapp-service`
   - **Namespace** : `workshop`
   - **Type** : `ClusterIP`
   - **Selector** : `app: webapp-demo`
   - **Port** : `80 → 80`

---

## Étape 3 : Validation du déploiement (5 min)

- Vérifiez 3 pods `Running` dans **Workloads > Pods**
- Notez les noms des pods actuels

---

## Étape 4 : Mise à jour avec RollingUpdate en action (25 min)

### 4.1 Préparation de la mise à jour
**Objectif** : Observer le processus RollingUpdate nginx:1.20 → nginx:1.21

### 4.2 Lancement de la mise à jour
1. **Workloads > Deployments** → Sélectionner `webapp-demo`
2. **Edit Config**
3. **Modifications** :
   - Image : `nginx:1.20` → `nginx:1.21`
   - Label version : `v1.0` → `v2.0`
4. **Save**

### 4.3 Observation en temps réel (activité clé 🎯)
**Instructions** : Ouvrez 2 onglets Rancher

**Onglet 1** - Surveillance pods :
- **Workloads > Pods** (namespace workshop)
- **Actualiser régulièrement** (F5)

**Onglet 2** - Événements :
- **Cluster > Events** (filtrer namespace workshop)

### 4.4 Séquence observée
Notez la progression :
1. ✅ Création du 1er nouveau pod (nginx:1.21)
2. ✅ Attente Ready du nouveau pod
3. ✅ Suppression du 1er ancien pod
4. ✅ Création du 2ème nouveau pod
5. ✅ Répétition jusqu'à completion

### 4.5 Points d'attention
- **Aucune interruption** : Service toujours accessible
- **Rollout progressif** : 1 pod à la fois (maxSurge=1, maxUnavailable=1)
- **Auto-healing** : Pods défaillants recréés automatiquement

---

## Étape 5 : Test de rollback rapide (10 min)

### 5.1 Simulation d'erreur
1. **Edit** le deployment webapp-demo
2. **Image défaillante** : `nginx:version-inexistante`
3. **Save** et observer l'échec

### 5.2 Rollback via Rancher
1. Dans le deployment → **Revision History**
2. Sélectionner la **révision précédente** (nginx:1.21)
3. **Rollback to this revision**
4. Vérifier le retour à l'état stable

---

## Étape 6 : Test des stratégies (5 min)

### Expérimentation rapide
Modifiez les paramètres RollingUpdate et relancez une mise à jour :

**Conservative** :
- maxSurge: 0, maxUnavailable: 1
- Observation : Mise à jour plus lente, un seul pod à la fois

**Agressive** :
- maxSurge: 2, maxUnavailable: 0  
- Observation : Mise à jour plus rapide, aucune interruption

---

## Étape 7 : Nettoyage et conclusion (3 min)

### Nettoyage
- Supprimer le deployment `webapp-demo`
- Supprimer le service `webapp-service`

### Points clés retenus ✅
- RollingUpdate = **zéro interruption** de service
- Paramètres **maxSurge/maxUnavailable** contrôlent la vitesse
- **Rollback facile** via l'historique des révisions
- **Surveillance temps réel** essentielle en production

---

## Questions express (2 min)

1. **Que se passe-t-il si maxSurge=0 et maxUnavailable=0 ?**
2. **Comment accélérer un rollout sans risque ?**
3. **Quand utiliser un rollback automatique ?**

---

## Ressources pour aller plus loin

- [Kubernetes Rolling Updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
- [Rancher Workload Management](https://rancher.com/docs/)

---

## Durée totale : 60 minutes

**Répartition** : 5min théorie + 45min pratique + 10min Q&R
