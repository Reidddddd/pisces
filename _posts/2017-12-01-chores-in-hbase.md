---
title: Chores in HBase
categories:
 - HBase
---

> If reasons fail, try code.

## ScheduledChore

`ScheduledChore`s are scheduled and performed periodically in hbase, responsible for various tasks. There are many chores in master, regionserver, thrift server. They are put in a queue owned by `ChoreService`. Because of periodic, each chore has its own start time (configurable) and may be different from others. And in `ChoreService`, there are workers polling chores from queue and executing tasks of chores, the number of workers dynamically changes, starting from one. You may notice it is possible that a chore may miss its start time, and this is the moment of increasing the number of workers, indicating the need of more workers to speed up executing tasks of chores, while decreasing the number of workers is not an usual case which happens only at stopping status.

### Chores in Master

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

### Chores in RegionServer

`nonceCleaner`(inner class), for cleaning up old nonces every 12 seconds by default.  
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

