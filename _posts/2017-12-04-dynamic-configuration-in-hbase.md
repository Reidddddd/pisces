---
title: Dynamic Configuration in HBase
categories:
 - HBase
---

> The significant problems we face cannot be solved at the same level of thinking we were at when we created them.

## Introduction

Since [HBASE-12147](https://issues.apache.org/jira/browse/HBASE-12147), hbase supports dynamically update configuration (only a subset) without restarting `HMaster` or `HRegionServer`, but the offical site doesn't provide a full list of available configurations that support this useful feature. So here i am. 

| Key (omit "hbase." prefix) | Value (default) |
| --- | ---: |
| ipc.server.fallback-to-simple-auth-allowed | false |
| cleaner.scan.dir.concurrent.size | 0.5 |
| regionserver.thread.compaction.large | 1 |
| regionserver.thread.compaction.small | 1 |
| regionserver.thread.split | 1 |
| regionserver.throughput.controller | PressureAwareCompactionThroughputController.class |
| regionserver.thread.hfilecleaner.throttle | 64*1024*1024 (64M) |
| regionserver.hfilecleaner.large.queue.size | 10240 |
| regionserver.hfilecleaner.small.queue.size | 10240 |
| regionserver.hfilecleaner.large.thread.count | 1 |
| regionserver.hfilecleaner.small.thread.count | 1 |
| regionserver.flush.throughput.controller | NoLimitThroughputController.class |
| hstore.compaction.max.size | 0x7fffffffffffffff (Long.MAX) |
| hstore.compaction.max.size.offpeak | 0x7fffffffffffffff (Long.MAX) |
| hstore.compaction.min.size | 1024*1024*128 (128M) |
| hstore.compaction.min | 3 |
| hstore.compaction.max | 10 |
| hstore.compaction.ratio | 1.2f |
| hstore.compaction.ratio.offpeak | 5.0f |
| regionserver.thread.compaction.throttle | 2*10*1024*1024*128 (2560M) |
| hregion.majorcompaction | 1000*60*60*24*7 (1 week) |
| hregion.majorcompaction.jitter | 0.5f |
| hstore.min.locality.to.skip.major.compact | 0.0f |
| hstore.compaction.date.tiered.max.storefile.age.millis | 0x7fffffffffffffff (Long.MAX) |
| hstore.compaction.date.tiered.incoming.window.min | 6 |
| hstore.compaction.date.tiered.window.policy.class | ExploringCompactionPolicy.class |
| hstore.compaction.date.tiered.single.output.for.minor.compaction | true |
| hstore.compaction.date.tiered.window.factory.class | ExponentialCompactionWindowFactory.class |
| offpeak.start.hour | -1 |
| offpeak.end.hour | -1 |
| oldwals.cleaner.thread.size | 2 |
| procedure.worker.keep.alive.time.msec | 0x7fffffffffffffff (Long.MAX) |
| procedure.worker.add.stuck.percentage | 0.5f |
| procedure.worker.monitor.interval.msec | 5000 (5 seconds) |
| procedure.worker.stuck.threshold.msec | 10000 (10 seconds) |
| regions.slop | 0.2 |
| regions.overallSlop | 0.2 |
| balancer.tablesOnMaster | false |
| balancer.tablesOnMaster.systemTablesOnly | false |
| util.ip.to.rack.determiner | ScriptBasedMapping.class |
| ipc.server.max.callqueue.length | 10*30 |
| ipc.server.priority.max.callqueue.length | 10*30 |
| ipc.server.callqueue.type | fifo |
| ipc.server.callqueue.codel.target.delay | 100 |
| ipc.server.callqueue.codel.interval | 100 |
| ipc.server.callqueue.codel.lifo.threshold | 0.8 |
| master.balancer.stochastic.maxSteps | 1000000 |
| master.balancer.stochastic.stepsPerRegion | 800 |
| master.balancer.stochastic.maxRunningTime | 30*1000 (30 seconds) |
| master.balancer.stochastic.runMaxSteps | false |
| master.balancer.stochastic.numRegionLoadsToRemember | 15 |
| master.loadbalance.bytable | false |
| master.balancer.stochastic.minCostNeedBalance | 0.05f |
| master.balancer.stochastic.localityCost | 25 |
| master.balancer.stochastic.rackLocalityCost | 15 |
| master.balancer.stochastic.readRequestCost | 5 |
| master.balancer.stochastic.writeRequestCost | 5 |
| master.balancer.stochastic.memstoreSizeCost | 5 |
| master.balancer.stochastic.storefileSizeCost | 5 |
| master.balancer.stochastic.regionReplicaHostCostKey | 100000 |
| master.balancer.stochastic.regionReplicaRackCostKey | 10000 |
| master.balancer.stochastic.regionCountCost | 500 |
| master.balancer.stochastic.primaryRegionCountCost | 500 |
| master.balancer.stochastic.moveCost | 7 |
| master.balancer.stochastic.maxMovePercent | 0.25f |
| master.balancer.stochastic.tableSkewCost | 35 |

## Code

This feature is implemented using `Observer` design pattern: a `ConfigurationManager` holds all the registered observer, and notifies all on receiving configuration change request. Each impl of `ConfigurationObserver` registers itself to `ConfigurationManager`.

```java
  // In ConfigurationManager

  public void notifyAllObservers(Configuration conf) {
    LOG.info("Starting to notify all observers that config changed.");
    synchronized (configurationObservers) {
      for (ConfigurationObserver observer : configurationObservers) {
        try {
          if (observer != null) {
            observer.onConfigurationChange(conf);
          }
        } catch (Throwable t) {
          LOG.error("Encountered a throwable while notifying observers: " + " of type : " +
              observer.getClass().getCanonicalName() + "(" + observer + ")", t);
        }
      }
    }
  }

  public void registerObserver(ConfigurationObserver observer) {
    synchronized (configurationObservers) {
      configurationObservers.add(observer);
      if (observer instanceof PropagatingConfigurationObserver) {
        ((PropagatingConfigurationObserver) observer).registerChildren(this);
      }
    }
  }
```
```java
  // Register an observer.
  ConfigurationManager configurationManager = new ConfigurationManager();
  // Initialize an observer: exampleChore
  configurationManager.registerObserver(exampleChore);
```

## Thoughts

1. First of all, `*.hfilecleaner.*` configurations above are misleading, since the component, which it belongs to, `HFileCleaner`, runs in master, not in regionserver.  
2. Online configuration change is a very useful feature, but rarely used in code base, only a few classes. It is better if we can apply it to more components.  
3. Some of the components implement an online configuration change by adding client interface on client side, and updating configuration on server side with `Coprocessor` framework. There are dulplicate works which is not pleasant in software management.  
