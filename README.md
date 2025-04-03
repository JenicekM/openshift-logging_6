# openshift-logging_6
How to deploy openshift logging 6.0 with Loki, how to migrate Elasticsearch to Loki. 


## **Deploy Elasticsearch**

Deploy openshift-logging 5.8 with Elasticsearch, using LogForwarder to forward audit,infra,app logs

```
oc apply -f elasticsearch.yaml
```

Review the procedure [Here](https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/logging/logging-6-0#log6x-upgrading-to-6)

* Backup Elasticsearch and Kibana resources  

  ```
    oc -n openshift-logging get elasticsearch elasticsearch -o yaml > /tmp/cr-elasticsearch.yaml
  ```
  ```
    oc -n openshift-logging get kibana kibana -o yaml > /tmp/cr-kibana.yaml
  ```
  > **Important** To continue using an existing Red Hat managed Elasticsearch or Kibana deployment provided by the elasticsearch-operator, remove the owner references from the Elasticsearch resource named elasticsearch, and the Kibana resource named kibana in the openshift-logging namespace before removing the ClusterLogging resource named instance in the same namespace.

* Temporarily set ClusterLogging to state Unmanaged  
  ```
    oc -n openshift-logging patch clusterlogging/instance -p '{"spec":{"managementState": "Unmanaged"}}' --type=merge
  ```
* Remove ClusterLogging ownerReferences from the Elasticsearch resource 
  ```
    oc -n openshift-logging patch elasticsearch/elasticsearch -p '{"metadata":{"ownerReferences": []}}' --type=merge
  ```
* Remove ClusterLogging ownerReferences from the Kibana resource 
  ```
    oc -n openshift-logging patch kibana/kibana -p '{"metadata":{"ownerReferences": []}}' --type=merge
  ```
* Remove ClusterLogging ownerReferences from the Kibana resource 
  ```
    oc -n openshift-logging patch clusterlogging/instance -p '{"spec":{"managementState": "Managed"}}' --type=merge
  ```

## **Deploy Loki**
* Deploy Minio Operator, 
  ```
  git clone https://github.com/JenicekM/minio-operator.git;cd minio-operator
  oc apply -f minio-operator.yaml
  oc apply -f scc-minio-operator.yaml 
  ```
* Deploy Minio Operator tenant and create S3 bucket
  ```
  oc apply -f tenant-openshift.yaml
  oc apply -f route.yaml 
  oc get route -n minio-tenant # user: minio pass: minio123
  ```
* Create S3 bucket and api_key, secret_key
  ```
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc alias set myminio http://localhost:9000 minio minio123
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc mb myminio/mybucket
  ```
* Create logging-loki-s3 secret 
  ```
  oc create secret generic logging-loki-s3 \
  --from-literal=bucketnames="loki-bucket" \
  --from-literal=endpoint="http://myminio-hl.minio-tenant.svc:9000" \
  --from-literal=access_key_id="2ckK7ufBNy5r1e0JKcHJ" \
  --from-literal=access_key_secret="AdJt308wCKq6ABgAjSYrNLztxPQoMpIxGCwVo1Uh" \
  -n openshift-logging --dry-run=client -o yaml > secret.yaml

  oc apply -f secret.yaml
  ```

* Deploy operators namespaces and subs.
  ```
  oc apply -f loki_6-prepare.yaml
  ```
* Deploy operators namespaces and subs.
  ```
  oc apply -f  loki_6.yaml
  ```
  > **Note** If all collectors are not restarting, delete daemonset and restart cluster-logging operator. 