# EDB Kasten integration 

## Goals

The goal of this project is to show how Kasten can be used with the EDB external backup adapter to create fast and consistent backup for EDB cluster.

The [external backup adapter](https://www.enterprisedb.com/docs/postgres_for_kubernetes/latest/addons/) create through annotations or labels a way for third party tool as Kasten to discover the api that they have to invoke to take a safe storage backup. 

This approach take full advantage of the Kasten data management for backing up pvc (snapshot are fully incremental and portable) and using EDB API to take consistent backup of large clusters.

## How it works 

1. The EDB Backup adapter will put the annotations/labels on one of the replicas (not the master) that has the commands to switch on backup mode 
2. Kasten prebackup hook blueprint discover this replica and call the EDB pre-backup command on it, now the PVC of the elected replica is fully consistent for a backup
3. Kasten proceed the backup of the complete namespace as usual
4. Kasten postbackup hook blueprint call the EDB post-backup commands, the elected replica is back in a "normal" mode
5. When Kasten restore the namespace, the EDB operator discover the elected replica and use it as the master for the EDB cluster

[Workflow diagram](./images/edb-backup-adapter.drawio.png)

# Getting Started 
## Install the operator 

If you already have EDB operator installed on kubernetes you can skip this part  

```
kubectl apply -f https://get.enterprisedb.io/cnp/postgresql-operator-1.19.1.yaml
```

This will create the operator namespace where the controller will be running.

## Create an EDB cluster, a client and some data 

If you already have an EDB cluster with your client application you can skip this part

```
kubctl create ns edb
kubectl apply -f cluster-example.yaml -n edb
```

Wait for the cluster to be fully ready.

Install the cnp plugin if you haven't it yet 
```
curl -sSfL \
  https://github.com/EnterpriseDB/kubectl-cnp/raw/main/install.sh | \
  sudo sh -s -- -b /usr/local/bin
```

Create a client certificate to the database
```
kubectl cnp certificate cluster-app \
  --cnp-cluster cluster-example \
  --cnp-user app \
  -n edb 
```

Now you can create the client 
```
kubectl create -f client.yaml -n edb 
```

Create some data 
```
kubectl exec -it deploy/cert-test -- bash
psql 'sslkey=/etc/secrets/app/tls.key sslcert=/etc/secrets/app/tls.crt sslrootcert=/etc/secrets/ca/ca.crt host=cluster-example-rw dbname=app user=app sslmode=verify-full'
\c app
DROP TABLE IF EXISTS links;
CREATE TABLE links (
	id SERIAL PRIMARY KEY,
	url VARCHAR(255) NOT NULL,
	name VARCHAR(255) NOT NULL,
	description VARCHAR (255),
        last_update DATE
);
INSERT INTO links (url, name, description, last_update) VALUES('https://kasten.io','Kasten','Backup on kubernetes',NOW());
select * from links;
\q
exit
```

## Add the backup decorator annotations to the cluster 

If you create the cluter from the previous section the cluster-example already include the backup decorator therefore you can skip this part.

Add this annotations to the cluster, in [cluster-example.yaml ](./cluster-example.yaml) you have an example. 

```
    "k8s.enterprisedb.io/addons": '["external-backup-adapter-cluster"]'
    "k8s.enterprisedb.io/externalBackupAdapterClusterConfig": |-
      electedResourcesDecorators:
        - key: "kasten-enterprisedb.io/elected"
          metadataType: "label"
          value: "true"
      excludedResourcesDecorators:
        - key: "kasten-enterprisedb.io/excluded"
          metadataType: "label"
          value: "true"
        - key: "kasten-enterprisedb.io/excluded-reason"
          metadataType: "annotation"
          value: "Not necessary for backup"
      backupInstanceDecorators:
        - key: "kasten-enterprisedb.io/hasHooks"
          metadataType: "label"
          value: "true"
        - key: "kanister.kasten.io/blueprint"
          metadataType: "annotation"
          value: "edb-hooks"
      preBackupHookConfiguration:
        container:
          key: "kasten-enterprisedb.io/pre-backup-container"
        command:
          key: "kasten-enterprisedb.io/pre-backup-command"
        onError:
          key: "kasten-enterprisedb.io/pre-backup-on-error"
      postBackupHookConfiguration:
        container:
          key: "kasten-enterprisedb.io/post-backup-container"
        command:
          key: "kasten-enterprisedb.io/post-backup-command"
```

## Install the edb blueprint

```
kubectl create -f edb-hooks.yaml
```

## create a bakckup policy with the hooks 

Create a policy for the edb namespace: set up a location profile for export and kanister actions. 

Add the hooks :

![Policy hooks](./images/policy-hooks.png)


## Launch a backup 

Launch a backup, that will create 2 restorepoints a local and a remote.

Delete the namespace edb 

```
kubectl delete ns edb
```

## Restore 

On the remote restore point hit restore. You should find all your datas.



