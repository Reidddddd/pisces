---
title: Chores in HBase
categories:
 - HBase
---

## Chores

Chores are scheduled and performed periodically in hbase. There are many chores in master, regionserver, thrift server. They are queued in a FIFO queue owned by `ChoreService`. Because of periodic, each chore has its own start time and may be different from others or not. And in `ChoreService`, there are workers polling chores from queue and executing tasks of chores, the number of workers dynamically changes, starting from one. You may notice it is possible that a chore may miss its start time, and this is the moment of increasing the number of workers, indicating the need of more workers to speed up executing tasks of chores, while decreasing the number of workers is not an usual case which happens only at stopping status.
