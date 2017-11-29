---
title: Chores in HBase
categories:
 - HBase
---

> If reasons fail, try code.

## Chores

Chores are scheduled and performed periodically in hbase. There are many chores in master, regionserver, thrift server. They are queued in a queue owned by `ChoreService`. Because of periodic, each chore has its own start time (configurable) and may be different from others or not. And in `ChoreService`, there are workers polling chores from queue and executing tasks of chores, the number of workers dynamically changes, starting from one. You may notice it is possible that a chore may miss its start time, and this is the moment of increasing the number of workers, indicating the need of more workers to speed up executing tasks of chores, while decreasing the number of workers is not an usual case which happens only at stopping status.

* Chores in Master

`HFileCleaner`, for cleaning hfile archives under ${hbase.rootdir}/archive folder every minute by default.  
`LogCleaner`, for cleaning oldWALs under ${hbase.rootdir}/oldWALs folder every minute by default.  
`BalancerChore`, for balancing regions in cluster every 5 minutes by default.  
`CatalogJanitor`, for cleaning merged regions and splited regions in hbase:meta every by default.  
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
