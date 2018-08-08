---
title: Key/Value, Unit of HBase
categories:
 - HBase
---

> Work consists of whatever a body is obliged to do, and that Play consists of whatever a body is not obliged to do. 

## Introduction

HBase is a key-value, NoSQL, distributed database, and well known around the world. But how its key-value looks like, and here we are. An illustration will help us understand quickly.

## KeyValue/Cell in HBase
![hbase_kv](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/hbase_kv.png)

## Thought
- Family byte[] is good to keep as short as possible, it may be the only part we can handler to limit the size of a cell.
- For a cell, it occupies at least 20 bytes, excluding the size of row byte[].
- Type after timestamp is type of an operation, like `Put`, `Delete`.
