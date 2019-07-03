---
title: Stand-alone Volume Backup Overview | Stash
description: An overview on how stand-alone volume backup works in Stash.
menu:
  product_stash_0.8.3:
    identifier: volume-backup-overview
    name: Backup & Restore a PVC?
    parent: volume-backup
    weight: 10
product_name: stash
menu_name: product_stash_0.8.3
section_menu_id: guides
---

# Backup Stand-alone PVC using Stash

## Before You Begin

## Prepare Volume

**Create PV:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    app: nfs-demo
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: "nfs-service.storage.svc.cluster.local"
    path: "/"
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/pv.yaml
persistentvolume/nfs-pv created
```

**Create PVC:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: demo
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      app: nfs-demo
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/pvc.yaml
persistentvolumeclaim/nfs-pvc created
```

```console
$ kubectl get pvc -n demo nfs-pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   1Gi        RWX                           32s
```

**Deploy Workload:**

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: demo-pod-1
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c","touch /shared/config/pod-1.conf && sleep 3000"]
    volumeMounts:
    - name: shared-config
      mountPath: /shared/config
  volumes:
  - name: shared-config
    persistentVolumeClaim:
      claimName: nfs-pvc
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/pod-1.yaml
pod/demo-pod-1 created
```

```console
$ kubectl exec -n demo demo-pod-1 ls /shared/config
index.html
pod-1.conf
```

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: demo-pod-2
  namespace: demo
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["/bin/sh", "-c","touch /shared/config/pod-2.conf && sleep 3000"]
    volumeMounts:
    - name: shared-config
      mountPath: /shared/config
  volumes:
  - name: shared-config
    persistentVolumeClaim:
      claimName: nfs-pvc
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/pod-2.yaml
pod/demo-pod-2 created
```

```console
$ kubectl exec -n demo demo-pod-2 ls /shared/config
index.html
pod-1.conf
pod-2.conf
```

## Backup

**Create Storage Secret:**

```console
$ echo -n 'changeit' > RESTIC_PASSWORD
$ echo -n '<your-project-id>' > GOOGLE_PROJECT_ID
$ cat /path/to/downloaded/sa/file.json > GOOGLE_SERVICE_ACCOUNT_JSON_KEY
$ kubectl create secret generic -n demo gcs-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./GOOGLE_PROJECT_ID \
    --from-file=./GOOGLE_SERVICE_ACCOUNT_JSON_KEY
secret/gcs-secret created
```

**Create Repository:**

```yaml
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: gcs-repo
  namespace: demo
spec:
  backend:
    gcs:
      bucket: appscode-qa
      prefix: stash-backup/volumes/nfs-pvc
    storageSecretName: gcs-secret
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/repository.yaml
repository.stash.appscode.com/gcs-repo created
```

**Create BackupConfiguration:**

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: nfs-pvc-backup
  namespace: demo
spec:
  task:
    name: pvc-backup
  repository:
    name: gcs-repo
  schedule: "*/5 * * * *"
  target:
    ref:
      apiVersion: v1
      kind: PersistentVolumeClaim
      name: nfs-pvc
    volumeMounts:
    - name: nfs-pvc
      mountPath: /shared/config
    directories:
    - /shared/config
  retentionPolicy:
    keepLast: 5
    prune: true
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/backupconfiguration.yaml
backupconfiguration.stash.appscode.com/nfs-pvc-backup created
```

If everything goes well, Stash will create a CronJob to trigger backup periodically.

**Verify CronJob:**

Verify that Stash has created a CronJob to trigger periodic backup of the targeted PVC by the following command,

```console
$ kubectl get cronjob -n demo
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
nfs-pvc-backup   */5 * * * *   False     0        <none>          28s
```

**Wait for BackupSession:**

Now, wait for a backup schedule to appear. You can watch for `BackupSession` crd using the following command,

```console
$ watch -n 1 kubectl get backupsession -n demo -l=stash.appscode.com/backup-configuration=nfs-pvc-backup

Every 1.0s: kubectl get backupsession -n demo -l=stash.appscode.com/backup-...  workstation: Wed Jul  3 19:53:13 2019

NAME                        BACKUPCONFIGURATION   PHASE       AGE
nfs-pvc-backup-1562161802   nfs-pvc-backup        Succeeded   3m11s
```

>Respective CronJob creates `BackupSession` crd with the following label `stash.appscode.com/backup-configuration=\<BackupConfiguration crd name>`. We can use this label to watch only the `BackupSession` of our desired `BackupConfiguration`.

**Verify Backup:**

When backup session is completed, Stash will update the respective `Repository` to reflect the latest state of backed up data.

Run the following command to check if the snapshots are stored in the backend,

```console
$ kubectl get repository -n demo gcs-repo
NAME       INTEGRITY   SIZE   SNAPSHOT-COUNT   LAST-SUCCESSFUL-BACKUP   AGE
gcs-repo   true        80 B   1                25s                      49m
```

If we navigate to `stash-backup/volumes/nvs-pvc` directory of our GCS bucket, we will see that the snapshot has been stored there.

<figure align="center">
  <img alt="Backed up data of a stand-alone PVC in GCS backend" src="/docs/images/guides/latest/volumes/pvc_repo.png">
  <figcaption align="center">Fig: Backed up data of a stand-alone PVC in GCS backend</figcaption>
</figure>

>Stash keeps all backup data encrypted. So, snapshot files in the bucket will not contain any meaningful data until they are decrypted.

## Restore

**Simulate Disaster:**

```console
$ kubectl exec -n demo demo-pod-2 -- sh -c "rm /shared/config/*"
```

```console
$ kubectl exec -n demo demo-pod-2 ls /shared/config/
```

**Create RestoreSession:**

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: nfs-pvc-restore
  namespace: demo
spec:
  task:
    name: pvc-restore
  repository:
    name: gcs-repo
  target:
    ref:
      apiVersion: v1
      kind: PersistentVolumeClaim
      name: nfs-pvc
    volumeMounts:
    - name:  nfs-pvc
      mountPath:  /shared/config
  rules:
  - paths:
    - /shared/config
```

```console
$ kubectl apply -f ./docs/examples/guides/latest/volumes/restoresession.yaml
restoresession.stash.appscode.com/nfs-pvc-restore created
```

**Wait for RestoreSession to Succeed:**

```console
$ watch -n 1 kubectl get restoresession -n demo nfs-pvc-restore


Every 1.0s: kubectl get restoresession -n demo nfs-pvc-restore                  workstation: Wed Jul  3 20:10:52 2019

NAME              REPOSITORY-NAME   PHASE       AGE
nfs-pvc-restore   gcs-repo          Succeeded   32s
```

**Verify Restored Data:**

```console
$ kubectl exec -n demo demo-pod-2 ls /shared/config/
index.html
pod-1.conf
pod-2.conf
```

## Cleanup

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl delete backupconfiguration -n demo nfs-pvc-backup
kubectl delete restoresession -n demo nfs-pvc-restore

kubectl delete secret -n demo gcs-secret
kubectl delete repository -n demo gcs-repo

kubectl delete pod -n demo demo-pod-1
kubectl delete pod -n demo demo-pod-2

kubectl delete pvc -n demo nfs-pvc
kubectl delete pv -n demo nfs-pv
```

If you would like to uninstall Stash operator, please follow the steps [here](/docs/setup/uninstall.md).
