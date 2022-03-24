https://access.crunchydata.com/documentation/postgres-operator/5.0.5/quickstart/


git clone ...

## install PGO
kubectl apply -k kustomize/install



## demo
kubectl apply -k kustomize/postgres


PRIMARY_POD=$(kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/role=master \
  -o jsonpath='{.items[*].metadata.labels.postgres-operator\.crunchydata\.com/instance}') && echo $PRIMARY_POD


kubectl exec -it $PRIMARY_POD-0 bash




## user



k get secret hippo-pguser-rhino -o jsonpath="{.data.password}" | base64 --decode ; echo

create table t_zoo1 as select s, md5(random()::text) from generate_Series(1,5) s;


BEGIN;
create table t_random as select s, md5(random()::text) from generate_Series(1,5) s;
COMMIT;




# backup

kubectl annotate -n postgres-operator postgrescluster hippo-multi-repo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-backup="$( date '+%F_%H:%M:%S' )"






select current_timestamp;

mc rm --recursive --force test1/tespkg-k8s-pgo-bucket/pgbackrest/postgres-operator/hippo-multi-repo


# PITR


```sh
PRIMARY_POD=$(kubectl -n postgres-operator get pods \
  --selector=postgres-operator.crunchydata.com/role=master \
  -o jsonpath='{.items[*].metadata.labels.postgres-operator\.crunchydata\.com/instance}') && echo $PRIMARY_POD
kubectl exec -it $PRIMARY_POD-0 psql

```

```yaml
kubectl exec -it $PRIMARY_POD-0 psql
create database target1a;
\c target1a;

BEGIN;
create table t_random as select s, md5(random()::text) from generate_Series(1,5) s;
COMMIT;

create database target1b;

first_timestamp=`echo "select current_timestamp" | kubectl exec -i ${PRIMARY_POD}-0 psql | grep "+"`
echo ${first_timestamp}
echo ${first_timestamp:1}
## 2022-03-23 10:10:20.571184+00


postgres=# select pg_switch_wal();
 pg_switch_wal 
---------------
 0/302C080
(1 row)

kubectl annotate -n postgres-operator postgrescluster hippo-multi-repo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-backup="$( date '+%F_%H:%M:%S' )"


kubectl exec -it $PRIMARY_POD-0 psql
create database target2a;
create database target2b;

sec_timestamp=`echo "select current_timestamp" | kubectl exec -i ${PRIMARY_POD}-0 psql | grep "+"`
echo ${sec_timestamp}
echo ${sec_timestamp:1}
## 2022-03-23 07:39:21.553572+00
```


bash-4.4$ pwd
/pgdata/pg14_wal
bash-4.4$ cat 000000010000000000000006.00000028.backup
START WAL LOCATION: 0/6000028 (file 000000010000000000000006)
STOP WAL LOCATION: 0/6000138 (file 000000010000000000000006)
CHECKPOINT LOCATION: 0/6000060
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2022-03-23 10:53:24 UTC
LABEL: pgBackRest backup started at 2022-03-23 10:53:23.927192+00
START TIMELINE: 1
STOP TIME: 2022-03-23 10:53:33 UTC
STOP TIMELINE: 1
bash-4.4$



## debug 
```yaml
/pgbackrest/repo1

[postgres@hippo-multi-repo-repo-host-0 repo1]$ du -sh *
2.1M    archive
27M     backup
12K     log
16K     lost+found

[postgres@hippo-multi-repo-repo-host-0 0000000100000000]$ cat 000000010000000000000004.00000028.backup
START WAL LOCATION: 0/4000028 (file 000000010000000000000004)
STOP WAL LOCATION: 0/4000138 (file 000000010000000000000004)
CHECKPOINT LOCATION: 0/4000060
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2022-03-23 09:24:47 UTC
LABEL: pgBackRest backup started at 2022-03-23 09:24:47.079323+00
START TIMELINE: 1
STOP TIME: 2022-03-23 09:24:58 UTC
STOP TIMELINE: 1

```


10:35 create  target2a target2b

10:38 drop  target1b


10:40 drop target2a

10:59 create target3a target3b


kubectl annotate postgrescluster hippo-multi-repo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-restore=id1