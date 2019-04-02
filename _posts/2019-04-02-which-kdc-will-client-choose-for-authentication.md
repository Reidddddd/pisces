---
title: Which KDC Will Client Choose for Authentication
categories:
 - Kerberos
---

> I never knew what for but I've always known it when it comes

## Background

In kerberized environment, every client should authenticate himself to KDC. For toy play, one KDC is enough, and it can serve all requests without overloading. But in production environment, with thousands of machines and clients, one KDC is not enough, a small auth traffic can lead to an application QPS downgrade. So, can auth requests are distributed to other KDCs in a cluster, and if so, how. 

## Talk is cheap, ___
Following is the snippet of how a client choose a KDC for authentication.
![sendReq](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/sendReq.png)
It is simple, but the code itself is not easy to find in java src.  
`var4`, retrieved from configuration `var3` according to realm, is a string concatenation of KDCs. `var5` is the iterator after the KDCs string processed.  
The question is answered, client will sequentially try the KDC by iterating `var4` through `var5`. So the former KDC in the list, the sooner client will pick it up for authentication.  
And the sequence of the KDCs is determined by the `krb5.conf`,
```
[realms]
 REALM.NAME = {
  kdc = kdc1.com
  kdc = kdc2.com
  kdc = kdc3.com
  kdc = kdc4.com
  kdc = kdc5.com
  kdc = kdc6.com
 }
```
It is an example with 6 KDCs, so a client will first try kdc1.com, if fails then kdc2 and so on.

## How
Back to question in the beginning,

Q1: How a client determine which KDC for authentication?  
A1: By the sequence of KDCs listed in `/etc/krb5.conf`  
Q2: Can auth requests be distributed to other KDCs in a cluster?  
A2: Yes, by giving out `/etc/krb5.conf` with different KDCs sequence.


But there's one thing need to point out, very important, all slave KDCs may have not catch up with the most updated db information in master KDC, so it is possible for a newly added client fail authentication himself to a slave KDC, for a while, only after KDCs sync done.
