# TP1 - Catherine Vanden Hende

## Kubeconfig

Les informations présentes dans la config de kube sont :

- La version de l'API
- Le cluster avec son certificat d'authenticité
- Le contexte avec les réfèrences au cluster, le namespace, l'utilisateur
- Le type de yml (ici Config)
- L'utilisateur avec son nom et son token

## Premières commandes

La différence entre `kubectl get pods -n votre_namespace` et `kubectl get pods -n default` est le droit d'accès. D'un côté, c'est "mon" namespace donc j'ai accès à tous ses pods et default n'est pas mon namespace et je n'y ai pas accès.

```bash
c.vanden-hende@tpg24:~/DevOpsII$ kubectl get pods -n catherine-vanden-hende
No resources found in catherine-vanden-hende namespace.
c.vanden-hende@tpg24:~/DevOpsII$ kubectl get pods -n default
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:tech:catherine-vanden-hende" cannot list resource "pods" in API group "" in the namespace "default"
```

## Etape 1

### Premières ressources

Création d'un pod à partir d'une image Docker : `kubectl run mynginx --image public.ecr.aws/l3x6e3t5/takima-training/nginx`
(https://kubernetes.io/docs/concepts/workloads/pods/)

Les propriétés principales de mon pod sont :

- La version de l'API
- Son nom, son label
- Le namespace
- Le type de yml (ici Pod)
- Un container avec son image et ses propriétés
- Son status avec les différentes conditions
- Le status de son container

(visible avec `kubectl get pods mynginx -o yaml`)

Suppression d'un pod : `kubectl delete pods nom_du_pod`

Créer un pod à partir d'un fichier yaml : `kubectl apply -f monfichier.yaml`

`--dry-run=client` : pas de création dans kubernates mais dump du fichier créé

Logs : `kubectl logs nom_du_pod`

### Replicaset

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: unicorn-front-replicaset
  labels:
    app: unicorn-front
spec:
  template:
    metadata:
      name: unicorn-front-pod
      labels:
        app: unicorn-front
    spec:
      containers:
        - name: unicorn-front
          image: public.ecr.aws/l3x6e3t5/takima-training/nginx
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front
```

Le `spec:template` contient les metadata des pods qui vont être créés. Le `selector:matchLabels` sert à préciser quels pods sont à traiter, spécifiés par leur label.

Il y a 3 pods dans mon namespace quand je deploie ce replicaset.

Quand je supprime un pod, un autre est recréé. Quand je supprime le replicaset, tous les pods créés par celui-ci disparraissent.

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unicorn-front-deployment
  labels:
    app: unicorn-front
spec:
  replicas: 3
  selector:
    matchLabels:
      app: unicorn-front
  template:
    metadata:
      labels:
        app: unicorn-front
    spec:
      containers:
        - name: unicorn-front
          image: public.ecr.aws/l3x6e3t5/takima-training/nginx:1.7.9
          ports:
            - containerPort: 80
```

Par rapport au ReplicaSet, le Deployment spécifie un port pour les pods. Ce n'est pas le même type de yaml (kind). Il permet de créer des replicaSets

Il y a 1 replicaset et 3 pods.

Une fois modifié et appliqué, le deployement a 2 replicasets mais seulement 3 pods.

`kubectl describe deployments` :

```bash
c.vanden-hende@tpg24:~/DevOpsII$ kubectl describe deployments
Name:                   unicorn-front-deployment
Namespace:              catherine-vanden-hende
CreationTimestamp:      Mon, 28 Nov 2022 13:41:39 +0100
Labels:                 app=unicorn-front
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=unicorn-front
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=unicorn-front
  Containers:
   unicorn-front:
    Image:        public.ecr.aws/l3x6e3t5/takima-training/nginx:1.7.9
    Port:         81/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   unicorn-front-deployment-596dbb7bb8 (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  12m    deployment-controller  Scaled up replica set unicorn-front-deployment-57fcc67585 to 3
  Normal  ScalingReplicaSet  3m28s  deployment-controller  Scaled up replica set unicorn-front-deployment-596dbb7bb8 to 1
  Normal  ScalingReplicaSet  3m26s  deployment-controller  Scaled down replica set unicorn-front-deployment-57fcc67585 to 2
  Normal  ScalingReplicaSet  3m26s  deployment-controller  Scaled up replica set unicorn-front-deployment-596dbb7bb8 to 2
  Normal  ScalingReplicaSet  3m23s  deployment-controller  Scaled down replica set unicorn-front-deployment-57fcc67585 to 1
  Normal  ScalingReplicaSet  3m23s  deployment-controller  Scaled up replica set unicorn-front-deployment-596dbb7bb8 to 3
  Normal  ScalingReplicaSet  3m21s  deployment-controller  Scaled down replica set unicorn-front-deployment-57fcc67585 to 0
```

On observe le nombre de révision, tous les evenements subis par le namespace avec les up et down ainsi que le nombre de replicas et de pods.

Editer la ressource : `kubectl edit deployment.v1.apps/unicorn-front-deployment`

### Rollback

`kubectl set image deployment.v1.apps/unicorn-front-deployment unicorn-front=public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage --record=true` :

```bash
Failed to pull image "public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage": rpc error: code = Unknown desc = Error response from daemon: manifest for public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage not found: manifest unknown: Requested image not found
```

Un ReplicaSet a été créé et un pod est en cours de création. Comme l'image n'est pas trouvable, le pod reste dans un état d'erreur et les pods existants ne sont pas supprimés.

`kubectl rollout history deployment.v1.apps/unicorn-front-deployment` :

```bash
deployment.apps/unicorn-front-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment.v1.apps/unicorn-front-deployment nginx=public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage --record=true
3         kubectl set image deployment.v1.apps/unicorn-front-deployment unicorn-front=public.ecr.aws/l3x6e3t5/takima-training/nginx:1.91-falseimage --record=true
```

Change-cause : commande changeant le deployement.

### Mettre à l'échelle

`kubectl scale deployment.v1.apps/unicorn-front-deployment --replicas=5`
Après la mise à l'échelle, il y a 5 pods.

### Mettre en standBy

`kubectl rollout pause deployment.v1.apps/unicorn-front-deployment`

Mise en pause puis modification : Il ne se passe rien sur les replicaSets.
Resume : un nouveau replicaSet est créé et les pods sont mis à jour

## Etape 2

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: unicorn-front-service
spec:
  selector:
    app: unicorn-front
  ports:
    - protocol: TCP
      port: 80 # port d'écoute du service
      targetPort: 80 # port exposé par le container du pod
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: unicorn-front-ingress
spec:
  rules:
    - host: replace-with-your-url
      http:
        paths:
          - backend:
              service:
                name: unicorn-front-service
                port:
                  number: 80
            path: /
            pathType: Prefix
  tls:
  - hosts:
     - replace-with-your-url
     secretName: unicorn-front-tls
```

### Mise en situation

Lors du lancement du hello-deployment, les pods n'arrivent pas à récuperer l'image car ils n'y ont pas accès.

La WebApp répond un texte "Je suis le pod situé sur le noeud" sur un fond coloré qui change de couleur quand on clique sur F5 (sûrement un changement de pod).

Création d'un secret : `kubectl create secret docker-registry auth-master3-registry --docker-server=registry.master3.takima.io --docker-username=readregcred --docker-password=ZzeuobQWZbPLJRbSNmxt`

A ajouter au ingress.yaml :

```yaml
imagePullSecrets:
  - name: auth-master3-registry
```

Après avoir ajouter les propriétés du pod et du node, elles s'affichent sur l'écran.

## Etape 3

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app
data:
  # property-like keys; each key maps to a simple value
  color: "#200"
```

Dans le deployement.yaml:

```yaml
env:
  - name: CUSTOM_COLOR # Vrai key de la variable d'env. Peut être différent de la valeur dans le config map
    valueFrom:
      configMapKeyRef:
        name: web-app # Nom du configmap
        key: color # nom de la clef dans le config map
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hello-secret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

Dans le deployement.yaml:

```yaml
env:
  - name: CUSTOM_COLOR # Vrai key de la variable d'env. Peut être différent de la valeur dans le config map
    valueFrom:
      secretpKeyRef:
        name: hello-secret # Nom du configmap
        key: username # nom de la clef dans le config map
```

### Resources

```yaml
resources:
  limits:
    cpu: 500m
  requests:
    cpu: 200m
```
