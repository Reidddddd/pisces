---
title: DataBlock Encoding in HBase
categories:
 - HBase
---

> What would you do if you were not afraid? 

## Introduction

HBase stores each cell individually, with its key and value. When a row has many cells, much space would be consumed by writing the same key many times, possibly. Therefore, encoding can save much space which has great positive impact on large rows. There are 4 available encoders in hbase. This article focus on `PrefixKeyDeltaEncoder`, `DiffKeyDeltaEncoder` and `FastDiffDeltaEncoder`. I will try to explain in illustration which may be more clear and understandable.
![encoder](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/Encoder.png)

## PrefixKeyDeltaEncoder
`PrefixKeyDeltaEncoder` tries to find the common prefix of key, excluding timestamp and type. It is the simplest one.

![prefix](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/prefix_key_encoder.png)

- cell_0~2 is a case of multi versions. They are only different in timestamp, so the common length between cell_0 and cell_1 and cell_1 and cell_2 are 10 (2+5+1+1+1), the key length of cell_1 and cell_2 are 9 (19-10).
- cell_2 and cell_3 has different qualifier but in the same row and the same family, however, their quailifiers still have common part which is 'q', so the 'q' is counted as the common part.

## DiffKeyDeltaEncoder
`DiffKeyDeltaEncoder` introduces a byte of flags to indicate some information, like two cells has same key/value length. In addition, it only write family once at the first cell, and ignore the rest, since hbase always deals with cells under the same column family.

![diff](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/diff_key_encoder.png)

- Flag is a byte, each bit has different meaning. Pos[0] is set, meaning previous cell and current cell have same key length, so key length of current cell will be omitted. Ditto for pos[1~2]. Pos[3] is set, meaning timestamps of two cells is different, and will write the difference only. Pos[4~6] three bits are used for recording the length of this timestamp or different timestamp, a long is 8 bytes, so there is a minus 1 before encoding. Pos[7] is used for recording if diff is signed.
- cell_0 is the first cell, so it has a family part ahead, flag is 0x50, because timestamp occupies 5 bytes to store, then key/value length, following 0 common length, and the rest.
- cell_1 flag is 0x0F, because cell_1's key/value length and type are equal to cell_0, note, their timestamp are different, so ts diff is set. The difference is 1 (15678918100-15678918099), so it is 0 after minus 1. Common length is 10, from row to qualifier. Ditto for cell_2.
- cell_3 flag is 0x8E, because key length is different, the different between two timestamps is -2 (previous - current), so pos[7] is set.

## FastDiffDeltaEncoder
`FastDiffDeltaEncoder` is nearly the same as `DiffKeyDeltaEncoder`, except for the flag and the way to encode timestamp and family.

![fast](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/fast_diff_key_encoder.png)

- From the flag we know, `FastDiffDeltaEncoder` will further encode value part, and it takes no care whether the timestamp is diff or not because it directly encode the different part of timestamp, the ts_len here is used for recording common length of timestamp.
- cell_0 has no previous cell, so its common length is zero.
- cell_1's flag is 0x7F, it has the same key/value length, same type and same value as cell_0. ts_len is 111, because they are different only in last byte, so after common length which is 10, encode the different part of cell_1 which is 0xD3, last byte of 15678918099. Ditto for cell_2.
- cell_3 and cell_2 has different key length, so pos[3] is unset. 

## Thought
- `FastDiffDeltaEncoder` compression rate is the highest, followed by `DiffKeyDeltaEncoder` and `PrefixKeyDeltaEncoder`.
- In order for read/write balance, users always add salt key before key which may have negative effect on encoding, because all are based on prefix.
