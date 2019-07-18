# Stash with Azure Kubernetes Service (AKS)

This guide will show you how to use Stash to backup and restore Kubernetes resource volume in [Azure Kubernetes Service](https://azure.microsoft.com/en-us/services/kubernetes-service/). Here, we are going to backup a Deployment's volume into [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/). Then, we are going to restore this data into another Deploymnet's volume.

## Before You Begin

- At first, you need to have a AKS cluster. If you don't already have a cluster, create one from [here](https://azure.microsoft.com/en-us/services/kubernetes-service/).

- Install `Stash` in your cluster following the steps [here](https://appscode.com/products/stash/0.8.3/setup/install/).

- You should be familiar with the following `Stash` concepts:
  - [BackupConfiguration](/docs/concepts/crds/backupconfiguration.md/)
  - [BackupSession](/docs/concepts/crds/backupsession.md/)
  - [RestoreSession](/docs/concepts/crds/restoresession.md/)
  - [Repository](/docs/concepts/crds/repository.md/)
- You will need a [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/) to store the backup snapshots.

To keep everything isolated, we are going to use a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

>**Note:** YAML files used in this tutorial are stored in  [docs/examples/guides/latest/platforms](/docs/examples/guides/latest/platforms/aks) directory of [stashed/stash](https://github.com/stashed/stash) repository.

## Backup Deployment's Data

Here, we are going to deploy a Deployment with a PVC and generate some sample data in it. Then, we will backup this sample data using Stash.

### Deploy workload

At first, we will create a PVC then we will create a Deployment that will use this PVC.

**Create PVC:**

Below is the YAML of the sample PVC that we are going to create,

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: stash-sample-data
  namespace: demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Let's create the PVC we have shown above,

```console
$ kubectl apply -f ./docs/examples/guides/latest/platforms/aks/pvc.yaml
persistentvolumeclaim/stash-sample-data created
```

**Deploy Deployment:**

Now, we will deploy a Deployment that uses the above PVC. This Deployment will automatically generate sample data (`data.txt` file) in `/source/data` directory where we have mounted the PVC.

Below is the YAML of the Deployment that we are going to create,

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: stash-demo
  name: stash-demo
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stash-demo
  template:
    metadata:
      labels:
        app: stash-demo
      name: busybox
    spec:
      containers:
      - args: ["echo sample_data > /source/data/data.txt && sleep 3000"]
        command: ["/bin/sh", "-c"]
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        volumeMounts:
        - mountPath: /source/data
          name: source-data
      restartPolicy: Always
      volumes:
      - name: source-data
        persistentVolumeClaim:
          claimName: stash-sample-data
```

Let's create the Deployment we have shown above.

```console
$ kubectl apply -f ./docs/examples/guides/latest/platforms/aks/deployment.yaml
deployment.apps/stash-demo created
```

Now, wait for pod of the Deployment to go into the `Running` state.

```console
kubectl get pod -n demo
NAME                          READY   STATUS    RESTARTS   AGE
stash-demo-8685fb5478-4psw8   1/1     Running   0          4m47s
stash-demo-8685fb5478-89flr   1/1     Running   0          4m47s
stash-demo-8685fb5478-fjggh   1/1     Running   0          4m47s
```

Verify that the sample data has been created in `/source/data` directory using the following command,

```console
$ kubectl exec -n demo stash-demo-8685fb5478-4psw8 -- cat /source/data/data.txt
sample_data
```

### Prepare Backend

Now, we are ready to store our backed up data into a [Azure Blob Container](https://azure.microsoft.com/en-us/services/storage/blobs/). We have to create a Secret with necessary credentials and a Repository crd to use this backend.

**Create Secret:**

To configure the [Azure Blob Container](https://azure.microsoft.com/en-us/services/storage/blobs/), the following secret keys are needed:

| Key                     | Description                                                |
|-------------------------|------------------------------------------------------------|
| `RESTIC_PASSWORD`       | `Required`. Password used to encrypt snapshots by `restic` |
| `AZURE_ACCOUNT_NAME`    | `Required`. Azure Storage account name                     |
| `AZURE_ACCOUNT_KEY`     | `Required`. Azure Storage account key                      |

Let's create a secret called `azure-secret` with access credentials to our desired [Azure Blob Container](https://azure.microsoft.com/en-us/services/storage/blobs/),

```console
$ echo -n 'changeit' >RESTIC_PASSWORD
$ echo -n '<your-azure-storage-account-name>' > AZURE_ACCOUNT_NAME
$ echo -n '<your-azure-storage-account-key>' > AZURE_ACCOUNT_KEY
$ kubectl create secret generic -n demo azure-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./AZURE_ACCOUNT_NAME \
    --from-file=./AZURE_ACCOUNT_KEY
secret/azure-secret created
```

Verify that the secret has been created successfully,

```console
kubectl get secret -n demo azure-secret -o yaml
```

```yaml
$ kubectl get secret -n demo azure-secret -o yaml
apiVersion: v1
data:
  AZURE_ACCOUNT_KEY: <base64 encoded AZURE_ACCOUNT_KEY>
  AZURE_ACCOUNT_NAME: <base64 encoded AZURE_ACCOUNT_NAME>
  RESTIC_PASSWORD: Y2hhbmdlaXQ=
kind: Secret
metadata:
  creationTimestamp: "2019-07-18T04:22:58Z"
  name: azure-secret
  namespace: demo
  resourceVersion: "68336"
  selfLink: /api/v1/namespaces/demo/secrets/azure-secret
  uid: b8c0685d-a913-11e9-9330-ea541341590e
type: Opaque
```

**Create Repository:**

Now, create a `Respository` using this secret. Below is the YAML of `Repository` crd we are going to create,

```yaml
apiVersion: stash.appscode.com/v1alpha1
kind: Repository
metadata:
  name: azure-repo
  namespace: demo
spec:
  backend:
    azure:
      container: stashqa
      prefix: /source/data
    storageSecretName: azure-secret
```

Let's create the Repository we have shown above,

```console
$ kubectl apply -f ./docs/examples/guides/latest/platforms/aks/repository.yaml
repository.stash.appscode.com/azure-repo created
```

Now, we are ready to backup our volumes to our backend.

### Backup

We have to create a `BackupConfiguration` crd targeting the `stash-demo` Deployment that we have deployed earlier. Then, Stash will inject a sidecar container into the target. It will also create a `CronJob` to take periodic backup of `/source/data` directory of the target.

**Create BackupConfiguration:**

Below is the YAML of the `BackupConfiguration` crd that we are going to create,

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: BackupConfiguration
metadata:
  name: deployment-backup
  namespace: demo
spec:
  repository:
    name: azure-repo
  schedule: "*/1 * * * *"
  target:
    ref:
      apiVersion: apps/v1
      kind: Deployment
      name: stash-demo
    volumeMounts:
    - name: source-data
      mountPath: /source/data
    directories:
    - /source/data
  retentionPolicy:
    name: 'keep-last-5'
    keepLast: 5
    prune: true
```

Here,

- `spec.repository` refers to the `Repository` object `azure-repo` that holds backend [Azure Blob Container](https://azure.microsoft.com/en-us/services/storage/blobs/) information.

Let's create the `BackupConfiguration` crd we have shown above,

```console
$ kubectl apply -f ./docs/examples/guides/latest/platforms/aks/backupconfiguration.yaml
backupconfiguration.stash.appscode.com/deployment-backup created
```

**Verify Sidecar:**

If everything goes well, Stash will inject a sidecar container into the `stash-demo` Deployment to take backup of `/source/data` directory. Let’s check that the sidecar has been injected successfully,

```console
$ kubectl get pod -n demo 
NAME                          READY   STATUS        RESTARTS   AGE
stash-demo-5bdc545845-45smg   2/2     Running       0          45s
stash-demo-5bdc545845-dw4dn   2/2     Running       0          54s
stash-demo-5bdc545845-ncrhw   2/2     Running       0          61s
```

Look at the pod. It now has 2 containers. If you view the resource definition of this pod, you will see that there is a container named `stash` which is running `run-backup` command.

```yaml
$ kubectl get pod -n demo stash-demo-5bdc545845-45smg -o yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: stash-demo
    pod-template-hash: 5bdc545845
  name: stash-demo-5bdc545845-45smg
  namespace: demo
...
spec:
  containers:
  - args:
    - echo sample_data > /source/data/data.txt && sleep 3000
    command:
    - /bin/sh
    - -c
    image: busybox
    imagePullPolicy: IfNotPresent
    name: busybox
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /source/data
      name: source-data
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-h6sqd
      readOnly: true
  - args:
    - run-backup
    - --backup-configuration=deployment-backup
    - --secret-dir=/etc/stash/repository/secret
    - --enable-cache=true
    - --max-connections=0
    - --metrics-enabled=true
    - --pushgateway-url=http://stash-operator.kube-system.svc:56789
    - --enable-status-subresource=true
    - --use-kubeapiserver-fqdn-for-aks=true
    - --enable-analytics=true
    - --logtostderr=true
    - --alsologtostderr=false
    - --v=3
    - --stderrthreshold=0
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: spec.nodeName
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: suaas21/stash:volumeTemp_linux_amd64
    imagePullPolicy: IfNotPresent
    name: stash
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /etc/stash
      name: stash-podinfo
    - mountPath: /etc/stash/repository/secret
      name: stash-secret-volume
    - mountPath: /tmp
      name: tmp-dir
    - mountPath: /source/data
      name: source-data
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-h6sqd
      readOnly: true
  dnsPolicy: ClusterFirst
  nodeName: aks-agentpool-72468344-0
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  ...
...
```

**Verify CronJob:**

It will also create a `CronJob` with the schedule specified in `spec.schedule` field of `BackupConfiguration` crd.

Verify that the `CronJob` has been created using the following command,

```console
$ kubectl get cronjob -n demo
NAME                SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
deployment-backup   */1 * * * *   False     0        35s             64s
```

**Wait for BackupSession:**

The `deployment-backup` CronJob will trigger a backup on each schedule by creating a `BackpSession` crd. The sidecar container will watches for the `BackupSession` crd. Once it found one, it will take backup immediately.

Wait for a schedule to appear. Run the following command to watch `BackupSession` crd,

```console
$ watch -n 2 kubectl get backupsession -n demo 
Every 1.0s: kubectl get backupsession -n demo     suaas-appscode: Mon Jun 24 10:23:08 2019

NAME                           BACKUPCONFIGURATION   PHASE       AGE
deployment-backup-1561350125   deployment-backup     Running     30s
deployment-backup-1561350125   deployment-backup     Succeeded   63s
```

We can see from the above output that the backup session has succeeded. Now, we will verify that the backed up data has been stored in the backend.

**Verify Backup:**

Once a backup is complete, Stash will update the respective `Repository` crd to reflect the backup. Check that the repository `azure-repo` has been updated by the following command,

```console
$ kubectl get repository -n demo 
NAME         INTEGRITY   SIZE   SNAPSHOT-COUNT   LAST-SUCCESSFUL-BACKUP   AGE
azure-repo   true        48 B   4                26s                      12m
```

Now, if we navigate to the Azure blob container, we will see backed up data has been stored in `<storage account name>/source/data` directory as specified by `spec.backend.azure.prefix` field of `Repository` crd.

<figure align="center">
  <img alt="Backup data in GCS Bucket" src="/docs/images/latest/platforms/aks.png">
  <figcaption align="center">Fig: Backup data in Azure Blob Container</figcaption>
</figure>

>**Note:** Stash keeps all the backed up data encrypted. So, data in the backend will not make any sense until they are decrypted.

## Restore Deployment's Data

This section will show you how to restore the backed up data from [Azure Blob Storage](https://azure.microsoft.com/en-us/services/storage/blobs/) we have taken in earlier section.

**Deploy Deployment:**

We are going to create a new Deployment named `stash-recovered` and restore the backed up data inside it.

Below is the YAML of the Deployment that we are going to create,

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
  namespace: demo
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: stash-demo
  name: stash-recovered
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stash-demo
  template:
    metadata:
      labels:
        app: stash-demo
      name: busybox
    spec:
      containers:
      - args:
        - sleep
        - "3600"
        image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        volumeMounts:
        - mountPath: /restore/data
          name: restore-data
      restartPolicy: Always
      volumes:
      - name: restore-data
        persistentVolumeClaim:
          claimName: restore-pvc
```

Let's create the Deployment we have shown above.

```console
$ kubectl apply -f ./docs/examples/guides/latest/platforms/aks/recovered_deployment.yaml
persistentvolumeclaim/restore-pvc created
deployment.apps/stash-recovered created
```

**Create RestoreSession:**

Now, we need to create a `RestoreSession` crd targeting the `stash-recovered` Deployment to restore the backed up data inside it.

Below is the YAML of the `RestoreSesion` crd that we are going to create,

```yaml
apiVersion: stash.appscode.com/v1beta1
kind: RestoreSession
metadata:
  name: deployment-restore
  namespace: demo
spec:
  repository:
    name: azure-repo
  rules:
  - paths:
    - /source/data/
  target: # target indicates where the recovered data will be stored
    ref:
      apiVersion: apps/v1
      kind: Deployment
      name: stash-recovered
    volumeMounts:
    - name: restore-data
      mountPath: /source/data
```

Here,

- `spec.repository.name` specifies the `Repository` crd that holds the backend information where our backed up data has been stored.

- `spec.target.ref` refers to the target workload where the recovered data will be stored.

Let's create the `RestoreSession` crd we have shown above,

```console
$ kubectl apply -f ./docs/examples/plateorms/aks/restoresession.yaml
restoresession.stash.appscode.com/deployment-restore created
```

Once, you have created the `RestoreSession` crd, Stash will inject `init-container` to `stash-recovered` Deployment. The Deployment will restart and the `init-container` will recovered on start-up.

**Verify Init-Container:**

Wait until the `init-container` has been injected to the `stash-recovered` Deployment. Let’s describe the Deployment to verify that `init-container` has been injected successfully.

```yaml
$ kubectl describe deployment -n demo stash-recovered
Name:                   stash-recovered
Namespace:              demo
CreationTimestamp:      Thu, 18 Jul 2019 11:59:41 +0600
Labels:                 app=stash-demo
Selector:               app=stash-demo
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
Pod Template:
  Labels:       app=stash-demo
  Annotations:  stash.appscode.com/last-applied-restoresession-hash: 15483804576325149444
  Init Containers:
   stash-init:
    Image:      suaas21/stash:volumeTemp_linux_amd64
    Port:       <none>
    Host Port:  <none>
    Args:
      restore
      --restore-session=deployment-restore
      --secret-dir=/etc/stash/repository/secret
      --enable-cache=true
      --max-connections=0
      --metrics-enabled=true
      --pushgateway-url=http://stash-operator.kube-system.svc:56789
      --enable-status-subresource=true
      --use-kubeapiserver-fqdn-for-aks=true
      --enable-analytics=true
      --logtostderr=true
      --alsologtostderr=false
      --v=3
      --stderrthreshold=0
    Environment:
      NODE_NAME:   (v1:spec.nodeName)
      POD_NAME:    (v1:metadata.name)
    Mounts:
      /etc/stash/repository/secret from stash-secret-volume (rw)
      /source/data from restore-data (rw)
      /tmp from tmp-dir (rw)
  Containers:
   busybox:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Args:
      sleep
      3600
    Environment:  <none>
    Mounts:
      /restore/data from restore-data (rw)
  Volumes:
   restore-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  restore-pvc
    ReadOnly:   false
   tmp-dir:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
   stash-podinfo:
    Type:  DownwardAPI (a volume populated by information about the pod)
    Items:
      metadata.labels -> labels
   stash-secret-volume:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  azure-secret
    Optional:    false
  ...
```

Notice the `Init-Containers` section. We can see that init-container `stash-init` has been injected which is running `restore` command.

**Wait for RestoreSession to Succeeded:**

Run the following command to watch RestoreSession phase,

```console
$ watch -n 2 kubectl get restoresession -n demo
Every 3.0s: kubectl get restore --all-namespaces                 suaas-appscode: Thu Jul 18 12:02:10 2019

NAMESPACE   NAME                 REPOSITORY-NAME   PHASE       AGE
demo        deployment-restore   azure-repo        Running     5s
demo        deployment-restore   azure-repo        Succeeded   1m
```

So, we can see from the output of the above command that the restore process succeeded.

**Verify Restored Data:**

In this section, we will verify that the desired data has been restored successfully.

At first, check if the `stash-recovered` Deployment's pod has gone into `Running` state by the following command,

```console
$ kubectl get pod -n demo
NAME                               READY   STATUS        RESTARTS   AGE
stash-recovered-6669c8bcfd-7pz9m   1/1     Running       0          76s
stash-recovered-6669c8bcfd-dfppw   1/1     Running       0          85s
stash-recovered-6669c8bcfd-qkllx   1/1     Running       0          51s
```

Verify that the sample data has been restored in `/restore/data` directory of the `stash-recovered` Deployment's pod using the following command,

```console
$ kubectl exec -n demo stash-recovered-6669c8bcfd-7pz9m -- cat /restore/data/data.txt
sample_data
```

# Cleaning Up

To clean up the Kubernetes resources created by this tutorial, run:

```console
kubectl delete -n demo deployment stash-demo
kubectl delete -n demo deployment stash-recovered
kubectl delete -n demo backupconfiguration deployment-backup
kubectl delete -n demo restoresession deployment-restore
kubectl delete -n demo repository azure-repo
kubectl delete -n demo secret azure-secret
kubectl delete -n demo pvc --all
```