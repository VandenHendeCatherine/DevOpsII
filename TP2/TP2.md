# TP2 - Catherine Vanden Hende

## Etape 1

### Deployer l'api

Création de l'api : deployement, service, ingress.
Les pods ne peuvent pas se lancer correctement car ils leur manque une connexion à postgres :

```bash
***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to bind properties under 'spring.datasource.hikari.username' to java.lang.String:

    Property: spring.datasource.hikari.username
    Value: ${POSTGRES_USER}
    Origin: class path resource [application.yml]:9:17
    Reason: Could not resolve placeholder 'POSTGRES_USER' in value "${POSTGRES_USER}"

```

## Etape 2

### Base de données

Utilisation de variables d'environnements :

```yaml
env:
  - name: POSTGRES_DB
    valueFrom:
      configMapKeyRef:
        name: pg-config # Nom du configmap
        key: db_name # nom de la clef dans le config map contenant le nom de la DB
```

Consultation de la bd : `psql -d database -U user`

## Etape 3

### Lier l'API et la BDD

Le nom du service de la base de données est pg-service (mon-service.mon-namespace)

## Etape 4

### Front

Le computer créé n'a pas été enregistré en base et comme on a changé de pod, il a été perdu.

## Etape 5

### Persistance

Le type d'accès pour le Storage Class EBS est ReadWriteOnce d'après le tableau. Cela convient bien à la persistance de norte bdd car elle ne peut etre lue ou modifiée que par 1 pod à la fois donc notre volume restera cohérent.

Le nom du PV est :
pvc-b6345c6d-7998-4e35-b678-1a6d7fe56b24

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-db
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 3Gi
```

dans le deployement.yaml

```yaml
volumes:
  - name: pg-data
    persistentVolumeClaim:
      claimName: pg-db
```

```yaml
volumeMounts:
  - mountPath: /var/lib/postgresql/data
    name: pg-data
```

### Admin Bdd

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgadmin
  labels:
    app: pgadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pgadmin
  template:
    metadata:
      labels:
        app: pgadmin
    spec:
      containers:
        - name: pgadmin
          image: "dpage/pgadmin4"
          imagePullPolicy: IfNotPresent
          envFrom:
            - configMapRef:
                name: pgadmin
          env:
            - name: PGADMIN_DEFAULT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pgadmin
                  key: pgadmin-password
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: pgadmin
data:
  PGADMIN_DEFAULT_EMAIL: admin@takima.io
---
apiVersion: v1
kind: Secret
metadata:
  name: pgadmin
  labels:
    app: pgadmin
type: Opaque
data:
  pgadmin-password: "YWRtaW4xMjMq" #base64 of admin123*
---
apiVersion: v1
kind: Service
metadata:
  name: pgadmin
  labels:
    app: pgadmin
spec:
  type: NodePort
  ports:
    - port: 80
  selector:
    app: pgadmin
```
