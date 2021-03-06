# Playbooks
This document contains playbooks, or at least a checklist of what to look for, for alerts in the cortex-mixin.

# Alerts

## CortexIngesterRestarts
First, check if the alert is for a single ingester or multiple. Even if the alert is only for one ingester, it's best to follow up by checking `kubectl get pods --namespace=<prod/staging/etc.>` every few minutes, or looking at the query `rate(kube_pod_container_status_restarts_total{container="ingester"}[30m]) > 0` just until you're sure there isn't a larger issue causing multiple restarts.

Next, check `kubectl get events`, with and without the addition of the `--namespace` flag, to look for node restarts or other related issues. Grep or something similar to filter the output can be useful here. The most common cause of this alert is a single cloud providers node restarting and causing the ingester on that node to be rescheduled somewhere else.

In events you're looking for things like:
```
57m Normal NodeControllerEviction Pod Marking for deletion Pod ingester-01 from Node cloud-provider-node-01
37m Normal SuccessfulDelete ReplicaSet (combined from similar events): Deleted pod: ingester-01
32m         Normal    NodeNotReady              Node   Node cloud-provider-node-01 status is now: NodeNotReady 
28m         Normal    DeletingAllPods           Node   Node cloud-provider-node-01 event: Deleting all Pods from Node cloud-provider-node-01.
```

If nothing obvious from the above, check for increased load:
- If there is an increase in the number of active series and the memory provisioned is not enough, scale up the ingesters horizontally to have the same number of series as before per ingester.
- If we had an outage and once Cortex is back up, the incoming traffic increases. (or) The clients have their Prometheus remote-write lagging and starts to send samples at a higher rate (again, an increase in traffic but in terms of number of samples). Scale up the ingester horizontally in this case too.

## CortexRequest Latency
First establish if the alert is for read or write latency. The alert should say.

### Write Latency
Using the Cortex write dashboard, find the cluster which reported the high write latency and deduce where in the stack the latency is being introduced:

distributor: It is quite normal for the distributor P99 latency to be 50-100ms, and for the ingesters to be ~5ms. If the distributor latency is higher than this, you may need to scale up the distributors. If there is a high error rate being introduced at the distributors (400s or 500s) this has been know to induce latency.

ingesters: It is very unusual for ingester latency to be high, as they just write to memory. They probably needs scaling up, but it is worth investigating what is going on first.

### Read Latency
Query performance is an known problem. When you get this alert, you need to work out if: (a) this is a operation issue / configuration (b) this is because of algorithms and inherently limited (c) this is a bug

Using the Cortex read dashboard, find the cluster which reported the high read latency and deduce where in the stack the latency is being introduced.

query_frontend: If there is a significant P99 or avg latency difference between the frontend and the querier, you can't scale them up - we rely on their being two frontend. Is this latency coming from the cache? Scale that up. What the CPU usage of the query frontend service? Do we need to increase the CPU requests and have it scheduled to a less busy box? Note QPS on the querier will be higher than on the frontend as it splits queries into multiple smaller ones.

ingesters: Latency should be in the ~100ms - queries are in memory. If its more, check the CPU usage and consider scaling it up. NB scale ingesters slowly, 1-2 new replicas an hour.

If you think its provisioning / scaling is the problem, consult the scaling dashboard. These are just recommendations - make reasonable adjustments.

Right now most of the execution time will be spent in PromQL's innerEval. NB that the prepare (index and chunk fetch) are now interleaved with Eval, so you need to expand both to confirm if its flow execution of slow fetching.

## CortexTransferFailed
This alert goes off when an ingester fails to find another node to transfer its data to when it was shutting down. If there is both a pod stuck terminating and one stuck joining, look at the kubernetes events. This may be due to scheduling problems caused by some combination of anti affinity rules/resource utilization. Adding a new node can help in these circumstances. You can see recent events associated with a resource via kubectl describe, ex: `kubectl -n <namespace> describe pod <pod>`

## CortexIngesterUnhealthy
This alert goes off when an ingester is marked as unhealthy. Check the ring web page to see which is marked as unhealthy. You could then check the logs to see if there are any related to that ingester ex: `kubectl logs -f ingester-01 --namespace=prod`. A simple way to resolve this may be to click the "Forgot" button on the ring page, especially if the pod doesn't exist anymore. It might not exist anymore because it was on a node that got shut down, so you could check to see if there are any logs related to the node that pod is/was on, ex: `kubectl get events --namespace=prod | grep cloud-provider-node`.

## CortexFlushStuck
@todo

## CortexLoadBalancerErrors
@todo

## CortexTableSyncFailure
@todo

## CortexQuerierCapacityFull
@todo

## CortexFrontendQueriesStuck
@todo

## CortexProvisioningTooMuchMemory
@todo

## MemcachedDown
@todo

## CortexRulerFailedRingCheck

This alert occurs when a ruler is unable to validate whether or not it should claim ownership over the evaluation of a rule group. The most likely cause is that one of the rule ring entries is unhealthy. If this is the case proceed to the ring admin http page and forget the unhealth ruler. The other possible cause would be an error returned the ring client. If this is the case look into debugging the ring based on the in-use backend implementation.

## CortexIngesterHasNotShippedBlocks

This alert fires when a Cortex ingester is not uploading any block to the long-term storage. An ingester is expected to upload a block to the storage every block range period (defaults to 2h) and if a longer time elapse since the last successful upload it means something is not working correctly.

How to investigate:
- Ensure the ingester is receiving write-path traffic (samples to ingest)
- Look for any upload error in the ingester logs (ie. networking or authentication issues)

### Ingester hit the disk capacity

If the ingester hit the disk capacity, any attempt to append samples will fail. You should:

1. Increase the disk size and restart the ingester. If the ingester is running in Kubernetes with a Persistent Volume, please refers to [Resizing Persistent Volumes using Kubernetes](#resizing-persistent-volumes-using-kubernetes).
2. Investigate why the disk capacity has been hit
  - Was the disk just too small?
  - Was there an issue compacting TSDB head and the WAL is increasing indefinitely?

## CortexIngesterHasNotShippedBlocksSinceStart

Same as [`CortexIngesterHasNotShippedBlocks`](#CortexIngesterHasNotShippedBlocks).

## CortexIngesterTSDBHeadCompactionFailed

This alert fires when a Cortex ingester is failing to compact the TSDB head into a block.

A TSDB instance is opened for each tenant writing at least 1 series to the ingester and its head contains the in-memory series not flushed to a block yet. Once the TSDB head is compactable, the ingester will try to compact it every 1 minute. If the TSDB head compaction repeatedly fails, it means it's failing to compact a block from the in-memory series for at least 1 tenant, and it's a critical condition that should be immediately investigated.

How to investigate:
- Look for details in the ingester logs

## CortexQuerierHasNotScanTheBucket

This alert fires when a Cortex querier is not successfully scanning blocks in the storage (bucket). A querier is expected to periodically iterate the bucket to find new and deleted blocks (defaults to every 5m) and if it's not successfully synching the bucket since a long time, it may end up querying only a subset of blocks, thus leading to potentially partial results.

How to investigate:
- Look for any scan error in the querier logs (ie. networking or rate limiting issues)

## CortexQuerierHighRefetchRate

This alert fires when there's an high number of queries for which series have been refetched from a different store-gateway because of missing blocks. This could happen for a short time whenever a store-gateway ring resharding occurs (e.g. during/after an outage or while rolling out store-gateway) but store-gateways should reconcile in a short time. This alert fires if the issue persist for an unexpected long time and thus it should be investigated.

How to investigate:
- Ensure there are no errors related to blocks scan or sync in the queriers and store-gateways
- Check store-gateway logs to see if all store-gateway have successfully completed a blocks sync

## CortexStoreGatewayHasNotSyncTheBucket

This alert fires when a Cortex store-gateway is not successfully scanning blocks in the storage (bucket). A store-gateway is expected to periodically iterate the bucket to find new and deleted blocks (defaults to every 5m) and if it's not successfully synching the bucket for a long time, it may end up querying only a subset of blocks, thus leading to potentially partial results.

How to investigate:
- Look for any scan error in the store-gateway logs (ie. networking or rate limiting issues)

## CortexCompactorHasNotSuccessfullyCleanedUpBlocks

This alert fires when a Cortex compactor is not successfully deleting blocks marked for deletion for a long time.

How to investigate:
- Ensure the compactor is not crashing during compaction (ie. `OOMKilled`)
- Look for any error in the compactor logs (ie. bucket Delete API errors)

## CortexCompactorHasNotSuccessfullyCleanedUpBlocksSinceStart

Same as [`CortexCompactorHasNotSuccessfullyCleanedUpBlocks`](#CortexCompactorHasNotSuccessfullyCleanedUpBlocks).

## CortexCompactorHasNotUploadedBlocks

This alert fires when a Cortex compactor is not uploading any compacted blocks to the storage since a long time.

How to investigate:
- If the alert `CortexCompactorHasNotSuccessfullyRun` or `CortexCompactorHasNotSuccessfullyRunSinceStart` have fired as well, then investigate that issue first
- If the alert `CortexIngesterHasNotShippedBlocks` or `CortexIngesterHasNotShippedBlocksSinceStart` have fired as well, then investigate that issue first
- Ensure ingesters are successfully shipping blocks to the storage
- Look for any error in the compactor logs

### Compactor is failing because of `not healthy index found`

The compactor may fail to compact blocks due a corrupted block index found in one of the source blocks:

```
level=error ts=2020-07-12T17:35:05.516823471Z caller=compactor.go:339 component=compactor msg="failed to compact user blocks" user=REDACTED err="compaction: group 0@6672437747845546250: block with not healthy index found /data/compact/0@6672437747845546250/REDACTED; Compaction level 1; Labels: map[__org_id__:REDACTED]: 1/1183085 series have an average of 1.000 out-of-order chunks: 0.000 of these are exact duplicates (in terms of data and time range)"
```

When this happen you should:
1. Rename the block prefixing it with `corrupted-` so that it will be skipped by the compactor and queriers. Keep in mind that doing so the block will become invisible to the queriers too, so its series/samples will not be queried. It's safe to do it on a single block with compaction level 1 (because of the samples replication), but not on multiple overlapping level 1 blocks or any block with a compaction level > 1.
2. Ensure the compactor has recovered
3. Investigate offline the root cause (eg. download the corrupted block and debug it locally)

To rename a block stored on GCS you can use the `gsutil` CLI:

```
# Replace the placeholders:
# - BUCKET: bucket name
# - TENANT: tenant ID
# - BLOCK:  block ID

gsutil mv gs://BUCKET/TENANT/BLOCK gs://BUCKET/TENANT/corrupted-BLOCK
```

## CortexCompactorHasNotUploadedBlocksSinceStart

Same as [`CortexCompactorHasNotUploadedBlocks`](#CortexCompactorHasNotUploadedBlocks).

## Resizing Persistent Volumes using Kubernetes

This is the short version of an extensive documentation on [how to resize Kubernetes Persistent Volumes](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/).

**Pre-requisites**:

- Running Kubernetes v1.11 or above
- The PV storage class has `allowVolumeExpansion: true`
- The PV is backed by a supported block storage volume (eg. GCP-PD, AWS-EBS, ...)

**How to increase the volume**:

1. Edit the PVC (persistent volume claim) `spec` for the volume to resize and **increase** `resources` > `requests` > `storage`
2. Restart the pod attached to the PVC for which the storage request has been increased

## How to create clone volume (Google Cloud specific)

In some scenarios, it may be useful to preserve current volume status for inspection, but keep using the volume.
[Google Persistent Disk supports "Clone"](https://cloud.google.com/compute/docs/disks/add-persistent-disk#source-disk) operation that can be used to do that.
Newly cloned disk is independant from its original, and can be used for further investigation by attaching it to a new Machine / Pod.

When using Kubernetes, here is YAML file that creates PV (`clone-ingester-7-pv`) pointing to the new disk clone (`clone-pvc-80cc0efa-4996-11ea-ba79-42010a96008c` in this example),
PVC (`clone-ingester-7-pvc`) pointing to PV, and finally Pod (`clone-ingester-7-dataaccess`) using the PVC to access the disk.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: clone-ingester-7-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity: 
    storage: 150Gi
  gcePersistentDisk:
    fsType: ext4
    pdName: clone-pvc-80cc0efa-4996-11ea-ba79-42010a96008c
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast
  volumeMode: Filesystem
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: clone-ingester-7-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 150Gi
  storageClassName: fast
  volumeName: clone-ingester-7-pv
  volumeMode: Filesystem
---
apiVersion: v1
kind: Pod
metadata:
    name: clone-ingester-7-dataaccess
spec:
    containers:
    - name: alpine
      image: alpine:latest
      command: ['sleep', 'infinity']
      volumeMounts:
      - name: mypvc
        mountPath: /data
      resources:
        requests:
          cpu: 500m
          memory: 1024Mi
    volumes:
    - name: mypvc
      persistentVolumeClaim:
        claimName: clone-ingester-7-pvc
```

After this preparation, one can use `kubectl exec -t -i clone-ingester-7-dataaccess /bin/sh` to inspect the disk mounted under `/data`.
