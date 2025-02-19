# ForgeRock Directory Service Operator - ds-operator

The ds-operator deploys
the [ForgeRock Directory Server](https://www.forgerock.com/platform/directory-services)
 in a Kubernetes cluster. This
is an implementation of the [Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) pattern.

Basic features of the operator include:

* Creation of StatefulSets, Services and Persistent volume claims for the directory
* Configures replication by adding new directory pods to the replication topology
* Change service account passwords in the directory using a Kubernetes secret.
* Take Volume Snapshots of the directory disk, and restore a directory based on a snapshot
* Backup and Restore directory data to LDIF format.

Please see the annotated [hack/ds-kustomize](hack/ds-kustomize) for the annotated reference of the DirectoryService custom resource specification.

Developers please refer to the [developers guide](DEVELOPMENT.md).


## Install the Operator

**Important: Only one instance of the operator is required per cluster.**

ForgeRock developers: This is already installed on the `eng-shared` cluster.

The [install.sh](install.sh) script will install the latest release of the operator. You can also curl this script:

```bash
curl  -L  "https://github.com/ForgeRock/ds-operator/releases/latest/download/install.sh" -o /tmp/install.sh
chmod +x /tmp/install.sh
/tmp/install.sh install
```

The operator runs in the `fr-system` namespace. To see log output use kubectl:

```bash
kubectl -n fr-system get pods
kubectl logs -n fr-system  -l  control-plane=ds-operator -f
# or if you have stern installed...
stern -n fr-system ds-
```

## Deploy a Directory Instance

Once the ds-operator has been installed, you can deploy an instance of the directory service using the sample in [hack/ds-kustomize].

Below is a sample deployment session

```bash
kubectl apply -k hack/ds-kustomize.yaml

# pw.sh script will retrieve the uid=admin password:
./hack/pw.sh

# View the pods, statefulset, etc
kubectl get pod

# Scale the deployment by adding another replica
kubectl scale directoryservice/ds-idrepo --replicas=2

# You can edit the resource, or edit the ds.yaml and kubectl apply changes
# Things you can change at runtime include the number of replicas
kubectl edit directoryservice/ds-idrepo

# Delete the directory instance.
kubectl delete -k hack/ds-kustomize

# If you want to delete the PVC claims...
kubectl delete pvc data-ds-0
```

The directory service deployment creates a statefulset to run the directory service. The usual
`kubectl` commands (get, describe) can be used to diagnose the statefulsets, pods, and services.

## Directory Docker Image

The deployed spec.image must work in concert with the operator. There are sample Dockerfiles in the ForgeOps project. You must use the
most recent "dynamic" ds image in https://github.com/ForgeRock/forgeops/tree/master/docker/ds/ds.

Evaluation images have been built for you on gcr.io/forgeops-public/ds. The [ds.yaml](hack/ds-kustomize/ds.yaml) Custom Resource references this image.

The entrypoint and behavior of the docker image is important. If you want to make changes please consult the README for the ds image in forgeops.

## Secrets

The operator supports creating (some) secrets, or a bring-your-own secrets model. 

The operator can generate random secrets for the `uid=admin` account, `cn=monitor` and application service accounts (for
example - the AM CTS account). Refer to the annotated sample.

Kubernetes cert-manager is the recommended way to generate PEM certificates for the directory deployment.
The included sample in [hack/ds-kustomize/cert.yaml](hack/ds-kustomize/cert.yaml) creates certificates for the master and SSL keypairs. 

However, cert-manager is not a requirement. Any valid PEM certificates can be used for the directory deployment.
For example, you could use `openssl` commands to generate the PEM key pairs. All certificates must be of secret type `kubernetes.io/tls`.

WARNING: It is _extremely_ important to backup the  master key pair. Directory data is encrypted using the master keypair, and if it is lost your directory data will be unrecoverable.

## Scheduling

The operator provides the following control over scheduling:

* The pods will tolerate nodes that have been tainted with the following:
  * `kubectl taint nodes node1 key=directory:NoSchedule`.
* Tainting
 such nodes will help them "repel" non directory workloads, which can be helpful for performance.  If nodes are not tainted,
 this toleration has no effect. This should be thought of as an optional feature that most users will not require.
* Topology Constraints: The DS nodes will _prefer_ to be scheduled on nodes and zones that do not have other directory pods on them. This is
  "soft" anti-affinity.  DS pods will still be scheduled on the same node if the scheduler is not able to fulfill the request. 

## Volume Snapshots

Beginning in Kubernetes 1.20, [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) are generally
available. Snapshots enable the creation of a rapid point-in-time snapshot of a disk image. 

The ds-operator enables the following auto-snapshot features:

* The ability to initialize directory server data from a previous volume snapshot. This process
is much faster than recovering from backup media. The time to clone the disk will depend on the provider,
but in general will happen at block I/O speeds.
* The ability to schedule snapshots at regular intervals, and to automatically delete older snapshots.

Use cases that are enabled by snapshots include:

* Rapid rollback and recovery from the last snapshot point.
* For testing, initializing the directory with a large amount of sample data saved in a previous snapshot.


### Snapshot Prerequisites

* The cloud provider must support Kubernetes Volume Snapshots. This has been tested on GKE version 1.18 an above.
* Snapshots require the `csi` volume driver. Consult your providers documentation. On GKE enable
  the `GcePersistentDiskCsiDriver` addon when creating or updating the cluster. The forgeops `cluster-up.sh` script
  for GKE has been updated to include this addon.
* Create a `VolumeSnapshotClass`. The default expected by the ds-operator is `ds-snapshot-class`. The `cluster-up.sh` script also creates
  this class using the following definition
```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: ds-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
EOF
```

* The StorageClass in the ds-operator deployment yaml must also use the CSI driver. When enabling the `GcePersistentDiskCsiDriver` addon, GKE will automatically
  create two new storage classes: `standard-rwo` (balanced PD Disk) and `premium-rwo` (SSD PD disk). The example `hack/ds.yaml`  has been updated.

### Initializing the Directory Server from a Previous Volume Snapshot

Edit the Custom Resource Spec (CR) and update the `dataSource` field to point to the snapshot.
See [Create a PersistentVolumeClaim from a Volume Snapshot](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#create-persistent-volume-claim-from-volume-snapshot).

The snapshot name can be any valid DS snapshot in the same namespace. The name `$(latest)` is special, and will
be replaced by the latest auto-snapshot taken by the operator.


PVC claims that are already provisioned are not overwritten by Kubernetes. This setting only applies to the
initialization of *new* PVC claims.


Changing this setting after deployment of a directory instance does not have any effect. This
setting works by updating the volume claim template for the Kubernetes StatefulSet. Kubernetes
does not allow the template to be updated after deployment.

A "rollback" procedure is as follows:

* Validate that you have a good snapshot to rollback to. `kubectl get volumesnapshots`.
* Delete the directory deployment *and* the PVC claims.  For example: `kubectl delete directoryservice/ds-idrepo-0 && kubectl delete pvc ds-idrepo-0` (repeat for all PVCs).
* Set the `dataSource` either to "$(latest)" (assuming the operator is taking snapshots) or the name of a specific volume snapshot.
* Redeploy the directory service.  The new PVCs will be cloned from the snapshot.
* *All* directory instances are initialized from the same snapshot volume. For example, in a two way replication topology,
  ds-idrepo-0 and ds-idrepo-1 will both contain the same starting data and changelog.

*WARNING*: This procedure destroys any updates made since the last snapshot. Use with caution.

### Enabling Automatic VolumeSnapshots

The ds-operator can be configured to automatically take snapshots. Update the CR spec:

```yaml
snapshots:
  enabled: true
  # Take a snapshot every 20 minutes
  periodMinutes: 20
  # Keep this many snapshots. Older snapshots will be deleted
  snapshotsRetained: 3
```

Snapshot settings can be dynamically changed while the directory is running.

Notes:

* Only the first pvc (data-ds-idrepo-0 for example) is used for the automatic snapshot. When initializing from a snapshot,
  all directory replicas start with the same data (see above)
* Snapshots can be expensive. Do not snapshot overly frequently, and retain only the number of
  snapshots that you need for availability. Many providers rate limit the number of snapshots that can be created. GCP
  limits the number of snapshots to 6 per hour per PVC.
* Snapshots are not a replacement for offline backups. If the data on disk is corrupt, the snapshot will also
  be corrupt.
* The very first snapshot will not happen until after the first `periodMinutes` (20 minutes in the example above).
  This is to give the directory time to start up before taking the first snapshot.

### Advanced Snapshot Scenarios

You can manually take snapshots at any time assuming you are using the CSI driver. For example

```
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: my-great-snapshot-1
spec:
  volumeSnapshotClassName: ds-snapshot-class
  source:
    persistentVolumeClaimName: data-ds-idrepo-0
EOF
```

You can pre-create PVC claims, causing Kubernetes to ignore the PVC claim template created by the operator.
For example, if you wanted to initialize a 3rd directory instance with a specific snapshot, apply
the following *before* the PVC is created by the operator:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.beta.kubernetes.io/gid: "0"
  labels:
  name: data-ds-idrepo-2
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard-rwo
  dataSource:
    apiGroup: snapshot.storage.k8s.io
    kind: VolumeSnapshot
    name: my-cool-snap-1
```

## Multi-cluster (Preview)

DS can be configured across multiple clusters located in the same or different geographical regions for high availability or DR purposes.

By overriding and passing the DS_BOOTSTRAP_SERVERS and DS_GROUP_ID environment variables, you can configure
replication across multiple clusters. The requirements:

* The DS pods must be directly reachable across all clusters.
* Pods must have a resolvable DNS name across all clusters.

Multi-cluster is an advanced use case. You must thoroughly understand how to configure cross cluster networking and DNS resolution
before attempting to setup DS multi-cluster.

We describe using [CloudDNS for GKE doc](https://github.com/ForgeRock/forgeops/blob/master/etc/multi-cluster/clouddns/README.md) in the forgeops repo.

To enable multi-cluster, configure the following environment variables in the custom resource

* DS_GROUP_ID must contain the cluster identifier as described in the CloudDNS docs(e.g. "eu" for EU cluster). This variable
 is appended to the pod hostname in each cluster to form a name that is unique across all ds instances in the topology. For example,
the pod `ds-idrepo-0` has the same name in the both the `us` and `eu` clusters. Appending the group id (say `eu`) makes the DS server id unique. 
Note that that group name is NOT used to form a DNS resolvable name. It exists for DS to disambiguate instances. 
* DS_BOOTSTRAP_REPLICATION_SERVERS enumerates the servers to bootstrap the cluster. This should include at least one server in each cluster.
These dns names must resolve in all clusters.


Example env configuration for ds-idrepo on EU cluster:

CloudDNS:
```yaml
  env:
    - name: DS_GROUP_ID
      value: "EU"
    - name: DS_BOOTSTRAP_REPLICATION_SERVERS
      value: "ds-idrepo-0.ds-idrepo.prod.svc.eu:8989,ds-idrepo-0.ds-idrepo.prod.svc.us:8989"
```

## Backup and Restore (Preview)

The operator supports two new Custom Resources:

* [DirectoryBackup])(hack/ds-backup.yaml)
* [DirectoryRestore](hack/ds-restore.yaml)


These CRs are used to create LDIF exports and restore them again.

Taking a DirectoryBackup does the following:

* Snapshots and then clones a PVC with the contents of the directory data.
* Runs a directory server binary pod, mounting that PVC
* Exports the data in (tar, LDIF or dsbackup) format to another pvc

When the process concludes, the pvc with the data can be further processed. For example
you can mount that pvc on a pod that will export the data to GCS or S3. This design
provides maximum flexibility to BYOJ (bring your own Job) for final archival.

Deleting the `DirectoryBackup` CR will the delete the volumesnapshot but it retains the backup pvc.

Taking a DirectoryRestore does the following:

* Creates a new 'data' pvc to hold the restored directory data
* Runs a job that mounts an existing backup PVC (likely the one created by a DirectoryBackup above) and
 performs an data import into the directory data pvc.
* Creates a volume snapshot of the directory data pvc

On conclusion of a restore, the volume snapshot can be used to initialize a new directory instance (see above).

For day to day backup of directory data, prefer to use tools such as [velero.io](https://velero.io). Ldif export and import
is useful to validate the integrity of the database, and for long term archival storage where a text based
format is preferred.