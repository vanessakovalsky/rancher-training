**Objectif :** Déployer une application web avec stratégie RollingUpdate

**Configuration avancée :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-frontend
  namespace: webapp-dev
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: webapp-frontend
  template:
    metadata:
      labels:
        app: webapp-frontend
        version: v1.0
    spec:
      imagePullSecrets:
      - name: docker-hub-secret
      containers:
      - name: frontend
        image: nginx:1.21-alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```
