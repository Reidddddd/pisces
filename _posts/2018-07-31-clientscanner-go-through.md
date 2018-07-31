---
title: ClientScanner Go through
categories:
 - HBase
---

> In order to make a man or a boy covet a thing, it is only necessary to make the thing difficult to attain.

## Introduction

We often write client codes like below. In fact, `ResultScanner` has two implementations, one is `ReversedClientScanner` the other is `ClientScanner` which is current topic.
```java
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
```

## UML
![ClientScannerUML](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/ClientScanner.png)

## Prerequisite
Few important parameters and `Scan` options should be mentioned before going through codes, they can possibly enlighten us about tuning client side performance.
- `hbase.client.replicaCallTimeout.scan`(1000sec by default), time to wait for primary replica.
- `hbase.client.retries.number` (31 by default), max retries if operation fails.
- `hbase.client.scanner.max.result.size` (2MB by default), maximum number of bytes returned when calling a scanner's next()
- `hbase.client.scanner.timeout.period` (60sec by default), client scanner timeout.
- `hbase.client.scanner.caching` (Integer.MAX_VALUE by default), number of `Result` cache in ClientScanner
