# TP3 - Catherine Vanden Hende

## ElasticSearch

```yaml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: formation-elk
spec:
  version: 7.16.0
  nodeSets:
  - name: default
    count: 3
    config:
      node.store.allow_mmap: false
------
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: formation-kb
spec:
  version: 7.16.0
  count: 1
  elasticsearchRef:
    name: "formation-elk"
  podTemplate:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        resources:
          limits:
            memory: 1Gi
            cpu: 1
-------
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
  name: kibana-ingress
spec:
  rules:
  - host: kibana.catherine-vanden-hende.takima.cloud
    http:
      paths:
      - backend:
          service:
            name: formation-kb-kb-http
            port:
              number: 5601
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - kibana.catherine-vanden-hende.takima.cloud
    secretName: app-wildcard-student
```

On remarque que 3 pods elk ont été créés (visibles dans Kibana). (`GET _cat/health?v GET _cat/allocation?v`)

```bash
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1669796902 08:28:22  formation-elk green           3         3     24  12    0    0        0             0                  -                100.0%
```

| shards | disk.indices | disk.used | disk.avail | disk.total | disk.percent | host       | ip         | node                       |
| :----- | :----------- | :-------- | :--------- | :--------- | :----------- | :--------- | :--------- | :------------------------- |
| 7      | 63.4mb       | 80.4mb    | 893mb      | 973.4mb    | 8            | 10.0.7.146 | 10.0.7.146 | formation-elk-es-default-1 |
| 7      | 65.8mb       | 83mb      | 890.3mb    | 973.4mb    | 8            | 10.0.2.48  | 10.0.2.48  | formation-elk-es-default-2 |
| 8      | 4.8mb        | 23.1mb    | 950.2mb    | 973.4mb    | 2            | 10.0.3.49  | 10.0.3.49  | formation-elk-es-default-0 |

Vérification du bon fonctionnement d'elasticSearch :

```bash
# PUT tp/_doc/1
{
  "_index" : "tp",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}

# GET tp/_doc/1
{
  "_index" : "tp",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "body" : "hello"
  }
}

# GET _cat/indices?v
health status index                           uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .geoip_databases                seN1eQFZTc2a6N__BquQCw   1   1         41           30      126mb           63mb
green  open   .security-7                     FdTu1ue_Tpuk4HH0xXdsdQ   1   1         53           53    730.7kb        365.3kb
green  open   .apm-custom-link                lbmHl5VgSdu9cbdbiYr1Dw   1   1          0            0       452b           226b
green  open   .kibana_task_manager_7.16.0_001 CoFSG6AAT9a2iXSEOAAeCQ   1   1         18         2327      2.4mb            2mb
green  open   .apm-agent-configuration        OLEnRXk-R5umPCKS-4XJWQ   1   1          0            0       452b           226b
green  open   .kibana_7.16.0_001              ArPo8BavR-GzC5rZREMeGQ   1   1         29            4      9.4mb          4.7mb
green  open   tp                              P5TY1k3kT-WyrY4ebDpb8w   1   1          1            0       452b           226b
green  open   .tasks                          rwiDivnOS8G2tVX6V3nC6w   1   1          2            0     15.5kb          7.7kb


```

## Helm

values.yaml

```yaml
---
front:
  enabled: true
  image:
    repository: registry.master3.takima.io/guide/kubernetes-resources/front
    tag: latest

  serviceName: "front"

  servicePort: 80
```

```yaml
{{- if .Values.front.enabled }}
apiVersion: v1
kind: Service
metadata:

  name: {{ .Values.front.serviceName }}

spec:
  selector:
    app: front
  ports:
    - protocol: TCP

      port: {{ .Values.front.servicePort }}

      targetPort: 80
{{ end }}

```

Tester les ressources : `helm template --values ./values.yaml ./ --output-dir dist`

Deployer : `helm install cdb ./`

Modification : `helm upgrade cdb ./`

Suppression : `helm delete mon-app`

## ArgoCD

Lorsqu'on détruit manuellement un deployement, ArgoCD le detecte et en relance un automatiquement.
Lorsqu'on modifie le values.yaml (nombre de pods par exemple), ArgoCD le detecte et fait la mise à jour en conséquence.
