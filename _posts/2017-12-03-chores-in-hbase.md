---
title: Chores in HBase
categories:
 - HBase
---

> If reasons fail, try code.

## Chore

Chores, `ScheduledChore` in code base, are scheduled and performed periodically with initial delay in hbase, responsible for various tasks. There are many chores in master, regionserver, thrift server. They are put in a queue owned by `ChoreService`. Because of periodic, each chore has its own start time (configurable) and may be different from others. And in `ChoreService`, there are workers polling chores from queue and executing tasks of chores, the number of workers dynamically changes, starting from one. You may notice it is possible that a chore may miss its start time, and this is the moment of increasing the number of workers, indicating the need of more workers to speed up executing tasks of chores, while decreasing the number of workers is not an usual case which happens only at stopping status.

### Chores in Master

`HFileCleaner`, for cleaning hfile archives under ${hbase.rootdir}/archive folder every minute by default.  
`LogCleaner`, for cleaning oldWALs under ${hbase.rootdir}/oldWALs folder every minute by default.  
`BalancerChore`, for balancing regions in cluster every 5 minutes by default.  
`CatalogJanitor`, for cleaning merged regions and splited regions in hbase:meta every 5 minutes by default.  
`ClusterStatusChore`, for feeding cluster status to balancer every minute by default.  
`ClusterStatusPublisher`, for publishing dead regionservers to client every 10 seconds by default.  
`ExpiredMobFileCleanerChore`, for cleaning expired (ttl set and min version is 0) mob file every day by default.  
`MobCompactionChore`, for running compaction to merge mob files every week by default.  
`QuotaObserverChore`, for receiving region filesystem-space usage reports and taking actions on those violating a defined quota every minute by default.  
`RegionNormalizerChore`, for normalizing regions (merge or split) every 5 minutes by default.  
`ReplicationMetaCleaner`, for cleaning useless data in hbase:meta used by serial replication every minute by default.  
`ReplicationZKNodeCleanerChore`, for cleaning replication queues belonging to the peer which does not exist every minute by default.  
`SnapshotQuotaObserverChore`, for computing the size of each snapshot created from a table with a space quota every 5 minutes by default.  
`TimeoutMonitor`, for checking split tasks and resubmiting those time out every second by default.

### Chores in RegionServer

`NonceCleaner`(inner class), for cleaning up old nonces every 5 minutes by default.  
`CompactionThroughputTuner`, for tuning max throughput for compaction every minute by default.  
`FlushThroughputTuner`, for tuning max throughput for flush every minute by default.  
`CompactedHFilesDischarger`, for cleaning up compacted files which are no longer in use and its entries in block cache every 2 minutes by default.  
`CompactionChecker`, for checking if regions need compaction every 10 seconds by default.  
`FileSystemUtilizationChore`, for computing the size of each region on the fs hosted by regionserver every 5 minutes by default.  
`HealthCheckChore`, for checking health of region server every 10 seconds by default.  
`HeapMemoryTunerChore`, for tuning memories every minute by default.  
`MovedRegionsCleaner`, for cleaning those moved regions cache every 2 minutes by default.  
`PeriodicMemStoreFlusher`, for flushing memstore every 5 minutes by default.  
`QuotaRefresherChore`, for updating quotas of namespace, table and user every 5 minutes by default.  
`SpaceQuotaRefresherChore`, for updating information from hbase:quota every minute by default.  
`StorefileRefresherChore`, for refreshing those store files of secondary regions hosted in the region servers disable by default.

### Chores in Others
`ConnectionCleaner`, for cleaning connections idle for a long time used in REST server and Thrift server every 10 seconds by default.  
`RefreshCredentials`, for refreshing kerberos tgt every 30 seconds by default.  

## Code Analysis

```java
  @Override
  public void run() {
    updateTimeTrackingBeforeRun();
    if (missedStartTime() && isScheduled()) {
      onChoreMissedStartTime();
      if (LOG.isInfoEnabled()) LOG.info("Chore: " + getName() + " missed its start time");
    } else if (stopper.isStopped() || !isScheduled()) {
      cancel(false);
      cleanup();
      if (LOG.isInfoEnabled()) LOG.info("Chore: " + getName() + " was stopped");
    } else {
      try {
        if (!initialChoreComplete) {
          initialChoreComplete = initialChore();
        } else {
          chore();
        }
      } catch (Throwable t) {
        if (LOG.isErrorEnabled()) LOG.error("Caught error", t);
        if (this.stopper.isStopped()) {
          cancel(false);
          cleanup();
        }
      }
    }
  }
```
It is the core pieces of codes in `ScheduledChore`. It's simple, but the tricky part is method `onChoreMissedStartTime()`, it will spawn a new worker in `ChoreService` for current chore missing its start time (only first miss and no more). This dynamic design it's quite efficient and can meet demand accordingly, but also avoids waste too much resources (since each chore could only spawm one worker at most).  
However, the number of workers can't shrink efficiently. I should point out that it happens in the `cancel()` method, but this method it's seldom called unless the region server is going to stop. Worse still, many chores are crashed at the same time because of the same interval but without an initial delay which leads to unnecessary requirement for a new worker. It brings no benefits on reducing the supply if demands are met far more than enough.  
I have to admit that i can't think about a solution for this concern at this moment (but i think the operation should start in `ChoreService` since it's the heart of schedule, and there should be a self-learning scheduler to separate chores properly to make full use of workers and to shrink effectively), and have no evidence that it must being wasted due to the over resourceful. More debugs and designs will coming.

## Table to Conclude

by default:

| ScheduledChore | Interval | Initial Delay | Funtionality |
| --- | :---: | :---: | --- |
| HFileCleaner | 1 minute | 0 | Clean hfile archives |
| LogCleaner | 1 minute | 0 | Clean old WALs |
| BalancerChore | 5 minutes | 0 | Run cluster balancer |
| CatalogJanitor | 5 minutes | 0 | Clean regions in hbase:meta |
| ClusterStatusChore | 1 minute | 0 | Feed cluster status |
| ClusterStatusPublisher | 10 seconds | 0 | Publish dead regionservers |
| ExpiredMobFileCleanerChore | 1 day | 1 day | Clean Mob file |
| MobCompactionChore | 1 week | 1 week | Compact mob file |
| QuotaObserverChore | 1 minute | 15 seconds | Receive quota information |
| RegionNormalizerChore | 5 minute | 0 | Merge or split regions |
| ReplicationMetaCleaner | 1 minute | 0 | Clean data in hbase:meta |
| ReplicationZKNodeCleanerChore | 1 minute | 0 | Clean replication queues |
| SnapshotQuotaObserverChore | 5 minutes | 1 minute | Computing sizes of snapshots |
| TimeoutMonitor | 1 second | 0 | Check split tasks and resume |
| NonceCleaner | 5 minutes | 0 | Clean old nonces |
| CompactionThroughputTuner | 1 minute | 0 | Tune max throughput for compaction |
| FlushThroughputTuner | 1 minute | 1 minute | Tune max throughput for flush |
| CompactedHFilesDischarger | 2 minutes | 0 | Cleaning compacted files |
| CompactionChecker | 10 seconds | 0 | Check the need of regions compaction |
| FileSystemUtilizationChore | 5 minutes | 1 minute | Compute the size of each region |
| HealthCheckChore | 10 seconds | 0 | Check health of region server |
| HeapMemoryTunerChore | 1 minute | 0 | Tune memories |
| MovedRegionsCleaner | 2 minutes | 0 | Clean moved regions cache |
| PeriodicMemStoreFlusher | 5 minutes | 0 | Flush memstore |
| QuotaRefresherChore | 5 minutes | 0 | Update quotas |
| SpaceQuotaRefresherChore | 1 minute | 15 seconds | Update information from hbase:quota |
| StorefileRefresherChore | disable | 0 | Refresh those store files of secondary regions |
| ConnectionCleaner | 10 seconds | 0 | Clean idle connections |
| RefreshCredentials | 30 seconds | 0 | Refresh kerberos tgt |

