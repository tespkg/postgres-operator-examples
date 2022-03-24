mc config host add s3bak https://s3.amazonaws.com  xxxx xxxx
mc ls s3bak/tespkg-k8s-pgo-bucket/pgbackrest/postgres-operator/hippo-multi-repo/repo2

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
kubectl annotate -n postgres-operator postgrescluster hippo \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date)"
```

If you intend to take one-off backups with similar settings in the future, you can leave those in the spec; just update the annotation to a different value the next time you are taking a backup.

To re-run the command above, you will need to add the --overwrite flag so the annotation’s value can be updated, i.e.

```
kubectl annotate -n postgres-operator postgrescluster hippo --overwrite \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date)"
```
