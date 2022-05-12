
# Migrating data to Capella


## Prerequisites

### 1) Whitelist Source cluster NAT Gateway Elastic IPs

For Capella cluster to be able to communicate with your managed cluster, you either need to while Whitelist NAT Gateways or VPC peer the hosted cluster with the Capella cluster. Here is how you can find out the IP address of the hosted cluster:

SSH to one of the node of the source cluster and run this command. Make sure `curl` package is installed.

```shell
$ curl https://canhazip.com/

35.XX.56.XXX
```

The IP returned should be used to whitelist on the Capella cluster by clicking `Connect > Manage Allowed IP > Add Allowed IP`

Once you are done white-listing the IP you can verify the connectivity from source cluster by running `nslookup` from any node on the source cluster. Make sure, however, your instance has `nslookup` installed. Here is the command you would use to verify the connectivity from source cluster to Capella cluster SRV:

```SHELL
$nslookup -type=SRV _couchbases._tcp.cb.alpha.cloud.couchbase.com
Server:         10.100.0.10
Address:        10.100.0.10#53

Non-authoritative answer:
_couchbases._tcp.cb.alias name="#statement".cloud.couchbase.com        service = 0 0 11207 a.abc.cloud.couchbase.com.
_couchbases._tcp.cb.b.cloud.couchbase.com        service = 0 0 11207 b.xvz.cloud.couchbase.com.
_couchbases._tcp.cb.c.cloud.couchbase.com        service = 0 0 11207 c.efg.cloud.couchbase.com.

Authoritative answers can be found from:
```

***Note***: In the SRV name above `cb.alpha.cloud.couchbase.com` is the cluster's WAN name that can be found from `Connect > Connection` tab and `_couchbases._tcp.` is the prefix which you have to add every-time.

### 2) Create Buckets in Capella

We have to manually create the same buckets on Capella cluster that exists on the source cluster to be able to apply backup restores.

{% hint style="working" %}
**Note:** Bucket conflict resolution should also match between source and target clusters otherwise restore would fail.
{% endhint %}

### 3) Create Database user in Capella

We also going to need Capella Database user credential at the time of restore that has Read/Write access to all the buckets under the cluster.

From Capella dashboard, select the Cluster first and then hit `Connect > Manage Credentials > Create Database Credential`.

In this example we are going to create a credential with username as `xdcr` and password as `passwd`. Select `All Buckets`, `All Scopes` and `Read/Write` access and finally hit `Create` button.

## Backup and Restore (for data migration)

### 1. Perform Full Backup on source Cluster

Here we are making an assumption that you have backup configured already with AWS S3. S3 would help us access backup snapshots easily during restore.




#### a) Manual backup via cbbackupmgr

In case you don't have cluster deployed and managed via K8s, then you can still use `cbbackupmgr` cli to trigger full backup. In our example we will be using a docker image (separate from CB Cluster) to be source of executing BR commands.

Start a docker container with same Couchbase image as used in Couchbase Cluster. This will make sure that same binaries are used by the backup/restore commands.

Once the container is up SSH into that container and set these environment variables.

```shell
export AWS_REGION=us-west-2
export CB_AWS_ENABLE_EC2_METADATA=true
export AWS_ACCESS_KEY_ID=XXXXX
export AWS_SECRET_ACCESS_KEY=YYYY/ZZZ
```
`Note`: Due to Couchbase image mismatch (between Couchbase Cluster and the binaries on the docker container) there is a possibility that backup and/or restore job may fail so make sure they are same

We are now ready to run the `cbbackupmgr config` command to create a new repository. Here is how you can do it:

```shell
$ cbbackupmgr config -a s3://cbdbdemo-backup/blue/archive \
-r capella-migration-0516 --obj-staging-dir /tmp/staging


Backup repository `capella-migration-0516` created successfully in
archive `s3://cbdbdemo-backup/blue/archive`
```
Run `info` command to list the snapshots under the repository. There is not going to be anything because we didn't take any backup but it still a good test.

```shell
$ cbbackupmgr info -a s3://cbdbdemo-backup/blue/archive  \
-r capella-migration-0516 --obj-staging-dir /tmp/staging

Name              | Size | # Backups |
capella-migration | 0B   | 0         |

```
At this point we are ready to take a full-backup under the newly configured repository `capella-migration-0516`.

```shell

$ cbbackupmgr backup -c couchbases://blue.cb.svc.cluster.local:18091 \
-a s3://cbdbdemo-backup/blue/archive -r capella-migration-0516 \
--obj-staging-dir /tmp/staging  --full-backup  --threads 4  \
--no-ssl-verify -u Administrator -p <passwd>
```

`Note`: You can also take a incremental-backup within an existing archieve but we would like restore to go fastest by taking a fresh full-backup.

Run `info` command again to list the snapshots under the repository. This time there is going to be at least one snapshot that we just took:

```shell
$ cbbackupmgr info -a s3://cbdbdemo-backup/blue/archive  \
-r capella-migration-0516 --obj-staging-dir /tmp/staging

Name                   | Size      | # Backups |
capella-migration-0426 | 215.97MiB | 1         |

+  Backup                         | Size      | Type | Source                                       | Cluster UUID                     | Range | Events | Aliases | Complete |
+  2022-04-27T19_41_39.916765854Z | 215.97MiB | FULL | couchbases://cluster:18091 | xvz | N/A   | 0      | 0       | true     |

```

#### b)  Backup via Cronjob

If you already have Couchbase Autonomous Operator managed backup cronjobs setup then taking a full backup is as simple as triggering the cronjob.

### 2. Restore Backup to Capella

First find out the endpoint of one of node of the Capella Cluster. In my example I will be using `8bf.cloud.couchbase.com` as the endpoint. Replace the password as `-p` and repository name as `-a`  where the backup snapshots are saved earlier and then execute below shell command:

```shell


$ cbbackupmgr restore -c https://3thfesc0aszntesi.fcokdfqzloopic-u.cloud.couchbase.com:18091 \
-a s3://cbdbdemo-backup/blue/archive  -r capella-migration-0516 \
--obj-staging-dir /tmp/staging/  --threads 4 \
--no-ssl-verify -u xdcr -p <passwd>

```

### 3. Run pillowfight to ingest data after restore

In the demo we would like to demostrate that after restoration of backup snapshot is done, any new mutations can be replicated to Capella via XDCR. In order to generate constant workload we are going to run `cbc pillowfight` tool to generate more mutations to the source bucket. Here is the command we used after performing `SSH` to one of the couchbase pod where we already have `Couchbase Home Directory` available and in the path:

```SHELL
$ cbc pillowfight -U couchbase://localhost/travel-sample -u Administrator \
--num-items 10000 --json --subdoc -m 1000 -M 5000 -P <passwd>

```
## XDCR for continuous replication



Once you have restored the data to Capella cluster, indexes for each bucket would be created as well. At this point if you would like to keep the source cluster (self-managed) active and data to be replicated to Capella cluster, XDCR can be used to initiate from source cluster to Capella Cluster.

Here are the steps you would need to perform to complete the uni-directional replication.




## Auxiliary




### 1. Backup/Restore Pod yaml

Adjust the `resources` section of the YAML to match `cpu` and `memory` available on the K8s nodes. You would also want to make sure that this pod doesn't host itself on the node running Couchbase pod, so make sure the minimum resources required to host `backup-restore` pod guarantees no co-existance with CB pods.

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: backup-restore
  namespace: cb
spec:  # specification of the pod's contents
  containers:
    - name: backup-restore-tools
      image: couchbase/operator-backup:1.1.1
      # Just spin & wait forever
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 30; done;" ]
      resources:
        requests:
          cpu: "1"
          memory: 1Gi
  restartPolicy: Never
```

### 2. Couchbase cluster-values.yaml

In the demo we used below `couchbase-deployment.yaml` file but the it is out-of scope to cover `how to deploy externalDNS` to fully deploy Couchbase cluster with loadbalancer.  

```YAML
# couchbaseOperator is the controller for couchbase cluster
couchbaseOperator:
  # additional command arguments will be translated to `--key=value`
  commandArgs:
    # pod creation timeout
    pod-create-timeout: 15m
    zap-log-level: info

# Default values for couchbase-cluster
cluster:
  platform: aws
  # name of the cluster. defaults to name of chart release
  name: blue
  # image is the base couchbase image and version of the couchbase cluster
  image: "couchbase/server:6.6.3"
  # guarantees that the pods in the same cluster are unable to be scheduled on the same node
  antiAffinity: true
  upgradeStrategy: RollingUpgrade
  hibernate: false
  hibernationStrategy: Immediate
  recoveryPolicy: PrioritizeDataIntegrity
  security:
    adminSecret: cb-auth-secret
    rbac:
      managed: false
  # networking options
  networking:
    networkPlatform: Istio
    # Option to expose admin console
    exposeAdminConsole: true
    # Option to expose admin console
    adminConsoleServices:
      - data
    # Specific services to use when exposing ui
    exposedFeatures:
      - client
      - xdcr
    # The Couchbase cluster tls configuration (auto-generated)
    tls:
      static:
        serverSecret: blue-server-tls
        operatorSecret: blue-operator-tls
    # The dynamic DNS configuration to use when exposing services
    dns:
      domain: xyz.com
    # Custom map of annotations to be added to console and per-pod (exposed feature) services
    exposedFeatureServiceTemplate:
      metadata:
        annotations:
          # service.beta.kubernetes.io/aws-load-balancer-internal: "true"
          service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
          service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "Environment=dev"
      spec:
        type: LoadBalancer
    adminConsoleServiceTemplate:
      metadata:
        annotations:
          # service.beta.kubernetes.io/aws-load-balancer-internal: "true"
          service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "3600"
          service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "Environment=dev"
      spec:
        type: LoadBalancer
  logging:
    # The retention period that log volumes are kept for after their associated pods have been deleted.
    logRetentionTime: 604800s
    # The maximum number of log volumes that can be kept after their associated pods have been deleted.
    logRetentionCount: 20
  # backup defines values for automated backup.
  backup:
    # managed determines whether Automated Backup is enabled
    managed: true
    # image used by the Operator to perform backup or restore
    # image: couchbase/operator-backup:6.6.0-102
    image: couchbase/operator-backup:1.1.0
    s3Secret: s3-cb-backup-secret-managed
    # optional service account to use when performing backups
    # service account will be created if it does not exist
    serviceAccountName: backup-blue

  # defines integration with third party monitoring sofware
  monitoring:
    prometheus:
      # defines whether Prometheus metric collection is enabled
      enabled: true
      # image used by the Operator to perform metric collection
      # (injected as a "sidecar" in each Couchbase Server Pod)
      image: couchbase/exporter:1.0.5
      resources:
        requests:
          cpu: 100m
          memory: 100Mi
        limits:
          memory: 500Mi
          cpu: 500m
  # Cluster wide settings for nodes and services
  cluster:
    # The amount of memory that should be allocated to the data service
    dataServiceMemoryQuota: 512Mi
    # The amount of memory that should be allocated to the index service
    indexServiceMemoryQuota: 512Mi
    # The amount of memory that should be allocated to the search service
    searchServiceMemoryQuota: 512Mi
    # The amount of memory that should be allocated to the eventing service
    eventingServiceMemoryQuota: 256Mi
    # The amount of memory that should be allocated to the analytics service
    analyticsServiceMemoryQuota: 1Gi
    # The index storage mode to use for secondary indexing
    indexStorageSetting: plasma
    indexer:
      storageMode: plasma
    # Timeout that expires to trigger the auto failover.
    autoFailoverTimeout: 10s
    # The number of failover events we can tolerate
    autoFailoverMaxCount: 3
    # Whether to auto failover if disk issues are detected
    autoFailoverOnDataDiskIssues: true
    # Whether the cluster will automatically failover an entire server group
    autoFailoverServerGroup: true
    # How long to wait for transient errors before failing over a faulty disk
    autoFailoverOnDataDiskIssuesTimePeriod: 60s
    # configuration of global Couchbase auto-compaction settings.
    autoCompaction:
      # amount of fragmentation allowed in persistent database [2-100]
      databaseFragmentationThreshold:
        percent: 20
        size: 2Gi
      # amount of fragmentation allowed in persistent view files [2-100]
      viewFragmentationThreshold:
        percent: 20
        size: 2Gi
      # whether auto-compaction should be performed in parallel
      parallelCompaction: false
      timeWindow:
        start: 09:00
        end: 13:00
        abortCompactionOutsideWindow: true
      tombstonePurgeInterval: 72h
  # kubernetes security context applied to pods
  securityContext:
    # fsGroup of persistent volume mount
    fsGroup: 1000
    runAsUser: 1000
    runAsNonRoot: true
  # cluster buckets
  buckets:
    # Managed defines whether buckets are managed by operator or the clients.
    managed: false
  enablePreviewScaling: false
  servers:
    # Name for the server configuration. It must be unique.
    default: null
  # Data service for demonstration purpose only. Use MDS in production.
    data-index-query:
      # Size of the couchbase cluster.
      size: 3
      # The services to run on nodes
      services:
        - data
        - index
        - query
        - eventing
      # Defines whether Autoscale is permitted for this specific server configuration.
      # Only `query` service is allowed to be defined unless `enablePreviewScaling` is set.
      autoscaleEnabled: false
      # volume claims to use for persistent storage
      volumeMounts:
        default: pvc-default 	# /opt/couchbase/var/lib/couchbase
        data: pvc-data	      # /mnt/data
        index: pvc-index	      # /mnt/index
      # ServerGroups define the set of availability zones we want to distribute pods over.
      serverGroups:
        - us-west-2a
        - us-west-2b
        - us-west-2c
      resources:
        requests:
          cpu: 1000m
          memory: 1Gi
      pod:
        spec:
          nodeSelector:
            beta.kubernetes.io/instance-type: t3.small

  # VolumeClaimTemplates define the desired characteristics of a volume
  # that can be requested and claimed by a pod.
  enableOnlineVolumeExpansion: true
  volumeClaimTemplates:
    - metadata:
        name: pvc-default
      spec:
        storageClassName: nas
        resources:
          requests:
            storage: 6Gi
    - metadata:
        name: pvc-analytics
      spec:
        storageClassName: nas
        resources:
          requests:
            storage: 20Gi
    - metadata:
        name: pvc-data
      spec:
        storageClassName: nas
        resources:
          requests:
            storage: 21Gi
    - metadata:
        name: pvc-index
      spec:
        storageClassName: nas
        resources:
          requests:
            storage: 13Gi
backups:
  default-backup:
    name: backup
    strategy: full_incremental
    full:
      schedule: "00 1 * * *"
    incremental:
      schedule: "30 * * * *"
    successfulJobsHistoryLimit: 1
    failedJobsHistoryLimit: 1
    backupRetention: 168h   # 7 days on local disk
    logRetention: 168h      # 7 days on local disk
    size: 50Gi
    s3bucket: s3://cbdbdemo-backup/blue

```
