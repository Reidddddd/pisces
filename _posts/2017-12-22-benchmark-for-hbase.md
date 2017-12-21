---
title: Benchmark for HBase
categories:
 - HBase
---

> But what wish, whether it be good or evil, will not always happen according to our desires.

## Introduction

As far as i known, there are two ways to benchmark HBase. One is quite famous, Yahoo Cloud Serving Benchmark (YCSB), which developing a framework and common set of workloads for evaluating the performance of different "key-value" and "cloud" serving stores, and HBase is included. The other is embedded in HBase, and serves HBase only of course, named `PerformanceEvaluation`, containing serveral workloads also, which can be used to evaluating HBase performance and scalability. I will try to cover both in source code level.

## PerformanceEvaluation

There are two modes for running evaluation, one is map-reduce mode, each mapper runs a single client, the other is multi-threads mode. Each test client, by default, does about 1GB of data. Here are the workloads (based on the most updated version of HBase):

### Workloads

| Workload | Description |
| --- | --- |
| AsyncRandomReadTest | Async random read test |
| AsyncRandomWriteTest | Async random write test |
| AsyncSequentialReadTest | Async sequential read test |
| AsyncSequentialWriteTest | Async sequential write test |
| AsyncScanTest | Async scan test (read every row) |
| RandomReadTest | Random read test |
| RandomSeekScanTest | Random seek and scan 100 test |
| RandomScanWithRange10Test | Random seek scan with both start and stop row (max 10 rows) |
| RandomScanWithRange100Test | Random seek scan with both start and stop row (max 100 rows) |
| RandomScanWithRange1000Test | Random seek scan with both start and stop row (max 1000 rows) |
| RandomScanWithRange10000Test | Random seek scan with both start and stop row (max 10000 rows) |
| RandomWriteTest | Random write test |
| SequentialReadTest | Sequential read test |
| SequentialWriteTest | Sequential write test |
| ScanTest | Scan test (read every row) |
| FilteredScanTest | Scan test using a filter to find a specific row based on it's value |
| IncrementTest | Increment on each row |
| AppendTest | Append on each row |
| CheckAndMutateTest | CheckAndMutate on each row |
| CheckAndPutTest | CheckAndPut on each row |
| CheckAndDeleteTest | CheckAndDelete on each row |

### Codes

All tests are based on class `TestBase` which defines the framework of a test (Template Pattern). The core in this framework is the method `test()`. Also **please pay attention to the my added docs paragragh /\*\*...\*/, they are explanations of codes**:
```java
long test() throws IOException, InterruptedException {
  /**
   * In testSetup() method, each test should define how it creates connection
   * and acts on start up by implementing createConnection()
   * and onStartup() method.
   */
  testSetup();
  LOG.info("Timed test starting in thread " + Thread.currentThread().getName());
  final long startTime = System.nanoTime();
  try {
    testTimed();
  } finally {
    /**
     * Symmetric to testSetup() method, each test should define how it acts on
     * take down and closes connection by implementing onTakedown() and
     * closeConnection() method.
     */
    testTakedown();
  }
  return (System.nanoTime() - startTime) / 1000000;
}
```
In testTimed() method:
```java
void testTimed() throws IOException, InterruptedException {
  int startRow = getStartRow();
  int lastRow = getLastRow();
  TraceUtil.addSampler(traceSampler);
  // Report on completion of 1/10th of total.
  for (int ii = 0; ii < opts.cycles; ii++) {
    /**
     * cycles define how many times this test should run,
     * by default only run 1 time.
     */
    if (opts.cycles > 1) LOG.info("Cycle=" + ii + " of " + opts.cycles);
    for (int i = startRow; i < lastRow; i++) {
      /**
       * everyN is sample rate, by default 1, meaning executing every row.
       * If it is 5, meaning executing every 5 rows.
       */
      if (i % everyN != 0) continue;
      long startTime = System.nanoTime();
      try (TraceScope scope = TraceUtil.createTrace("test row");){
        /**
         * It exactly executes the test's action, like scan, put, read.
         */
        testRow(i);
      }
      if ( (i - startRow) > opts.measureAfter) {
        // If multiget is enabled, say set to 10, testRow() returns immediately first 9 times
        // and sends the actual get request in the 10th iteration. We should only set latency
        // when actual request is sent because otherwise it turns out to be 0.
        if (opts.multiGet == 0 || (i - startRow + 1) % opts.multiGet == 0) {
          latencyHistogram.update((System.nanoTime() - startTime) / 1000);
        }
        if (status != null && i > 0 && (i % getReportingPeriod()) == 0) {
          status.setStatus(generateStatus(startRow, i, lastRow));
        }
      }
    }
  }
}
```

### Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| nomapred | false | Run multiple clients using threads rather than use mapreduce |
| filterAll | false | Helps to filter out all the rows on the server side |
| startRow | 0 | Rows to start | 
| size | 1.0f | Total size in GiB |
| perClientRunRows | 1048576 | Rows each client runs |
| numClientThreads | 1 | Threads of client |
| totalRows | 1048576 | Total rows for test | 
| measureAfter | 0 | Start to measure the latency once measureAfter rows have been treated |
| sampleRate | 1.0f | Execute test on a sample of total rows. Only supported by randomRead |
| traceRate | 0.0 | Enable HTrace spans. Initiate tracing every N rows |
| tableName | TestTable | Alternate table name |
| flushCommits | true | Used to determine if the test should flush the table |
| writeToWAL | true | Set writeToWAL on puts |
| autoFlush | false | Set autoFlush on htable |
| oneCon | false | all the threads share the same connection |
| useTags | false | Writes tags along with KVs. Use with HFile V3 |
| noOfTags | 1 | Specify the no of tags that would be needed |
| reportLatency | false | Set to report operation latencies |
| multiGet | 0 | Batch gets together into groups of N |
| randomSleep | 0 | Do a random sleep before each get between 0 and entered value |
| inMemoryCF | false | Tries to keep the HFiles of the CF inmemory as far as possible |
| presplitRegions | 0 | Create presplit table |
| replicas | 1 | Enable region replica testing |
| splitPolicy | null | Specify a custom RegionSplitPolicy for the table |
| compression | **NONE**, LZO, GZ, SNAPPY, LZ4, BZIP2, ZSTD | Compression type to use |
| bloomType | NONE, **ROW**, ROWCOL | Bloom filter type | 
| blockSize | 65536 | Blocksize to use when writing out hfiles |
| blockEncoding | **NONE**, PREFIX, DIFF, FAST_DIFF, ROW_INDEX_V1 | Block encoding to use | 
| valueRandom | false | Set if we should vary value size between 0 and valueSize |
| valueZipf | false | Set if we should vary value size between 0 and valueSize in zipf |
| valueSize | 1000 | Value size to use | 
| period | perClientRunRows / 10 | Report every period rows |
| cycles | 1 | How many times to cycle the test | 
| columns | 1 | Columns to write per row |
| caching | 30 | Scan caching to use |
| addColumns | true | Adds columns to scans/gets explicitly |
| inMemoryCompaction | NONE, **BASIC**, EAGER, ADAPTIVE | Makes the column family to do inmemory flushes/compactions. |
| asyncPrefetch | false | Enable asyncPrefetch for scan |
| cacheBlocks | true | Set the cacheBlocks option for scan |
| scanReadType | **DEFAULT**, STREAM, PREAD | Set the readType option for scan |

## YCSB

YCSB contains mainly contains two components, one is `Client`, the other is `DB`. `Client` just runs a specific workload against db. `DB` defines the common interfaces, such as `scan`, `update`, `insert`, `delete`, `read`, etc. And each specific db, like `HBase`, `Cassandra`, implements them.

### Workloads

| Workload | Brief | Distribution | Description |
| --- | --- | --- | --- |
| A | Update Heavy | zipfian | 50 percent reads and 50 percent updates |
| B | Read Heavy | zipfian | 95 percent reads and 5 percent updates |
| C | Read Only | zipfian | 100 percent reads |
| D | Read Latest | latest | 95 percent reads and 5 percent inserts |
| E | Short Ranges | uniform | 95 percents scans and 5 percent inserts |
| F | Read-modify-write | zipfian | 50 percents reads and 50 percents reads-modifies-writes |

### Distribution

Distributions means the probability of a record to be performed (insert, update, read, scan). There are following values:

1. `Uniform`, choose a record at random, and all records are equally likely to be chosen.
2. `Zipfian`, some records will be popular, while some are unpopular.
3. `Latest`, similiar with `Zipfian`, but the most recently inserted is the most popular.
4. `Sequential`, records are picked sequentially between a range [start, end).
5. `Hotspot`, divides records in two parts, one is hot, other is cold. And a probability to perform records in hot data.
6. `Exponential`, records are in exponential distribution, the lower number of records are more popular.

### Parameters

- HBase Configurations:

| HBase DB | Value | Description |
| --- | --- | --- |
| clientbuffering | false | buffer mutations on the client |
| writebuffersize | null | buffer size for client |
| durability | **USE_DEFAULT**, SKIP_WAL, ASYNC_WAL, SYNC_WAL, FSYNC_WAL | durability of a mutation |
| kerberos | **SIMPLE**, KERBEROS | client authentication |
| principal | null | if kerberos enable, principal of client |
| hbase.security.authentication | null | if kerberos enable, location of principal |
| table | usertable | table name |
| debug | null (true/false) | debug message |
| hbase.usepagefilter | true | use page filter |
| columnfamily | (required) | column family of a table |

- Core Workload Configurations:

| Core Workload | Default | Description |
| --- | --- | --- |
| recordcount | (required) | The number of records in the table to be inserted in |
| table | usertable | table name |
| fieldcount | 10 | number of fields in a record |
| fieldlengthdistribution | **constant**, uniform, zipfian, histogram  | field length distribution |
| fieldlength | 100 | length of a field in bytes |
| fieldlengthhistogram | hist.txt | filename containing the field length histogram |
| readallfields | true | whether to read one field (false) or all fields (true) of a record |
| writeallfields | false | whether to write one field (false) or all fields (true) |
| dataintegrity | false | whether to check all returned data integrity |
| readproportion | 0.95 | proportion of transactions that are reads |
| updateproportion | 0.05 | proportion of transactions that are updates |
| insertproportion | 0.0 | proportion of transactions that are inserts |
| scanproportion | 0.0 | proportion of transactions that are scans |
| readmodifywriteproportion | 0.0 | proportion of transactions that are read-modify-write |
| requestdistribution | (ditto) | distribution of requests across the keyspace |
| zeropadding | 1 | zero padding to record numbers in order to match string sort order |
| maxscanlength | 1000 | max scan length |
| scanlengthdistribution | **uniform**, zipfian | scan length distribution |
| insertorder | **hashed**, ordered | order to insert records |
| hotspotdatafraction | 0.2 | percentage data items that constitute the hot set |
| hotspotopnfraction | 0.8 | percentage operations that access the hot set |
| core_workload_insertion_retry_limit | 0 | times to retry when an insertion to a DB fails |
| core_workload_insertion_retry_interval | 3 | wait between the retries, seconds |
| operationcount | 3000000 | operations to use during the run phase |
| insertstart | 0 | offset of the first insertion |
| measurementtype | histogram | latency measurements are presented |
| histogram.buckets | 1000 | range of latencies to track in the histogram (milliseconds) |
| timeseries.granularity | 1000 | granularity for time series (in milliseconds) |

## Overall

I tried both, `YCSB` is much more authoritative in test cases and more informative on result report. While `PerformanceEvaluation` is more like a functional test, it can train you how to write a client program in effective way.
