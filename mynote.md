mc config host add pgs3 https://s3.amazonaws.com  xxxx xxxx
mc ls pgs3/tespkg-k8s-pgo-bucket/pgbackrest/postgres-operator/hippo-multi-repo/repo2

## install PGO
kubectl apply -k kustomize/install


## create postgres cluster


kubectl apply -k kustomize/multi-backup-repo-shaojun

kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/cluster=hippo-multi-repo 


## create db
k get po --show-labels |grep master
k exec -it hippo-multi-repo-00-hgjh-0 -- bash
psql

create database mydb1;
\l


## configure the custom resource to take a one-off full backup
first, enable full backup
```
spec:
  backups:
    pgbackrest:
      manual:
        repoName: repo1
        options:
         - --type=full
```
note, this does not the trigger on-off full backup,
you have to do that by adding the postgres-operator.crunchydata.com/pgbackrest-backup annotation to your custom resource. The best way to set this annotation is with a timestamp, so you know when you initialized the backup.
```
kubectl annotate -n postgres-operator postgrescluster hippo-multi-repo \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date)"
```

If you intend to take one-off backups with similar settings in the future, you can leave those in the spec; just update the annotation to a different value the next time you are taking a backup.

To re-run the command above, you will need to add the --overwrite flag so the annotation’s value can be updated, i.e.

```
kubectl annotate -n postgres-operator postgrescluster hippo-multi-repo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date)"

kubectl -n postgres-operator get postgrescluster hippo-multi-repo -o jsonpath='{.metadata.annotations."postgres-operator.crunchydata.com/pgbackrest-backup"}'
```
## shutdown cluster, and then, create a new cluster and restore data from s3

shutdown
```
kubectl patch postgrescluster/hippo-multi-repo -n postgres-operator --type merge \
  --patch '{"spec":{"shutdown": true}}'
```

restore to new cluster

```
kubectl -n postgres-operator apply -f restore-elephant-f-s3.yaml
```

## path elephant cluster backup policy
copy restore-elephant-f-s3.yaml to set-elephant-backup-repo1.yaml
edit set-elephant-backup-repo1.yaml
delete spec.dataSource
add below content
```
  backups:
    pgbackrest:
      global:
        repo1-retention-full: "3"
        repo1-retention-full-type: time
      repos:
      - name: repo1
        schedules:
          full: "*/10 * * * *"
          incremental: "*/5 * * * *"
```
apply set-elephant-backup-repo1.yaml

```
kubectl -n  postgres-operator apply -f set-elephant-backup-repo1.yaml
```
## test backup pitr

```sh
PRIMARY_POD=$(kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/role=master \
  -o jsonpath='{.items[*].metadata.labels.postgres-operator\.crunchydata\.com/instance}') && echo $PRIMARY_POD
kubectl exec -it $PRIMARY_POD-0 -- bash

echo 'select current_timestamp'| psql
```

```
       current_timestamp
-------------------------------
 2022-04-12 02:24:14.73141+00
(1 row)

```

list db
```
echo '\l' |psql
```

```
                                                                            List of databases
       Name       |  Owner   | Encoding |   Collate   |    Ctype    |        Access privileges        |  Size   | Tablespace |                Description
------------------+----------+----------+-------------+-------------+---------------------------------+---------+------------+--------------------------------------------
 elephant         | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres                   +| 8577 kB | pg_default |
                  |          |          |             |             | postgres=CTc/postgres          +|         |            |
                  |          |          |             |             | elephant=CTc/postgres           |         |            |
 hippo-multi-repo | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres                   +| 8577 kB | pg_default |
                  |          |          |             |             | postgres=CTc/postgres          +|         |            |
                  |          |          |             |             | "hippo-multi-repo"=CTc/postgres |         |            |
 postgres         | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |                                 | 8649 kB | pg_default | default administrative connection database
 t1               | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |                                 | 8577 kB | pg_default |
 t1_elephant      | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |                                 | 8577 kB | pg_default |
 template0        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres                    +| 8401 kB | pg_default | unmodifiable empty database
                  |          |          |             |             | postgres=CTc/postgres           |         |            |
 template1        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres                    +| 8577 kB | pg_default | default template for new databases
                  |          |          |             |             | postgres=CTc/postgres           |         |            |
(7 rows)
```

```
dropdb t1_elephant
echo 'select current_timestamp'| psql
```

```
       current_timestamp
-------------------------------
 2022-04-12 02:25:07.112981+00
(1 row)
```

list db
```

echo '\l' |psql
```

```
                                          List of databases
       Name       |  Owner   | Encoding |   Collate   |    Ctype    |        Access privileges
------------------+----------+----------+-------------+-------------+---------------------------------
 elephant         | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres                   +
                  |          |          |             |             | postgres=CTc/postgres          +
                  |          |          |             |             | elephant=CTc/postgres
 hippo-multi-repo | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =Tc/postgres                   +
                  |          |          |             |             | postgres=CTc/postgres          +
                  |          |          |             |             | "hippo-multi-repo"=CTc/postgres
 postgres         | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
 t1               | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 |
 template0        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres                    +
                  |          |          |             |             | postgres=CTc/postgres
 template1        | postgres | UTF8     | en_US.utf-8 | en_US.utf-8 | =c/postgres                    +
                  |          |          |             |             | postgres=CTc/postgres
(6 rows)
```


restore

update restore-elephant-pitr.yaml as below
```
spec:
  dataSource:
    postgresCluster:
      clusterNamespace: postgres-operator
      clusterName: elephant
      repoName: repo1
      options: 
      - --type=time 
      - --target="2022-04-12 02:25:07+00" 
```
```
kubectl apply -f restore-elephant-pitr.yaml
```