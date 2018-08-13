---
title: DataBlock Encoding in HBase
categories:
 - HBase
---

> What would you do if you were not afraid? 

## Introduction

HBase stores each cell individually, with its key and value. When a row has many cells, much space would be consumed by writing the same key many times, possibly. Therefore, encoding can save much space which has great positive impact on large rows. There are 4 available encoders in hbase. This article focus on `PrefixKeyDeltaEncoder`, `DiffKeyDeltaEncoder` and `FastDiffDeltaEncoder`.
![encoder](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/Encoder.png)

## PrefixKeyDeltaEncoder
![prefix](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/prefix_key_encoder.png)

- cell_0~2 is a case of multi versions. They are only different in timestamp, so the common length between cell_0 and cell_1 is 10(2+5+1+1+1), the key length of cell_1 is 9(19-10).

- cell_2 and cell_3 has different qualifier but they are still in the same row and the same family, however, their quailifiers still have common part which is 'q', so the 'q' is counted as the common part.
 
## Thought
- Family byte[] is good to keep as short as possible, it may be the only part we can handler to limit the size of a cell.
- For a cell, it occupies at least 20 bytes, excluding the size of row byte[].
- Type after timestamp is type of an operation, like `Put`, `Delete`.
