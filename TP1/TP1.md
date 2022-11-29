# TP1 - Catherine Vanden Hende

## Kubeconfig

Les informations présentes dans ce fichier sont :

- La version de l'API
- Le cluster avec son certificat d'authenticité
- Le contexte avec les réfèrences au cluster, le namespace, l'uitlisateur
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

Les propriétés principales de mon pod sont :

- La version de l'API
- Son nom, son label
- Le namespace
- Le type de yml (ici Pod)
- Un container avec son image et ses propriétés
- Son status avec les différentes conditions
- Le status de son container

### Replicaset

Le `spec:template` contient les metadata des pods qui vont être créés. Le `selector:matchLabels` sert à préciser quels pods sont à traiter, spécifiés par leur label.

Il y a 3 pods dans mon namespace.

Quand je supprime un pod, un autre est recréé. Quand je supprime le replicaset, tous les pods créés par celui-ci disparraissent.

### Deployment

Par rapport au ReplicaSet, le Deployment spécifie un port pour les pods. Ce n'est pas le même type de yaml (kind)

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

Après la mise à l'échelle, il y a 5 pods.

### Mettre en standBy

Mise en pause puis modification : Il ne se passe rien sur les replicaSets.
Resume : un nouveau replicaSet est créé et les pods sont mis à jour

## Etape 2

### Mise en situation

Lors du lancement du hello-deployment, les pods n'arrivent pas à récuperer l'image car ils n'y ont pas accès.

La WebApp répond un texte "Je suis le pod situé sur le noeud" sur un fond coloré qui change de couleur quand on clique sur F5 (sûrement un changement de pod).

Après avoir ajouter les propriétés du pod et du node, elles s'affichent sur l'écran.
