## Trend
Althought hadoop ecosystem makes many distributed system/algorithms easier and costs to build up such a system is lower, enterprises and vendors are never satisfied with that, so faster becomes the next issue. Many techniques springs up to seize that chance. Among them, we fix our eyes on alluxio, which focus on speeding up by using memory techniques.  
Alluxio clusters act as a read/write cache for data in connected storage systems. Temporarily storing data in memory, or other media near compute, accelerates access and provides local performance from remote storage. This capability is even more critical with the movement of compute applications to the cloud and data being located in object stores separate from compute. Caching is transparent to the user, and uses read/write buffering to maintain continuity with persistent storage. Intelligent cache management utilizes configurable policies for efficient data placement and supports tiered storage for both memory and disk (SSD/HDD). More details, [link: Alluxio introduction](https://alluxio.com/products)

## Considerations about Alluxio
1. There are plenty of master-slaves architectures in big data ecosystem. Those centralized system have a same problem, they have various states and a great numbers of meta, not to mention the exact data, which means once master is overwhelmed whole system could hardly function normally. We can say those services a sort of heavy-weighed services, while a light-weighed service is stateless or its states does not count importance. Unfornately, alluxio is one of heavy-weighed services and brings pressures on our dev/ops team.
2. To make full use of alluxio, memory is necessary. Both reading from and writing to memory are definitely fast without doubt, but here comes the questions.
* Since cold reading will trigger fetching data from remote, do tasks running on alluxio still well perform over normal cases?
* Do we need to load every data fetched from remote into alluxio?
* If we write data to remote(like HDFS) ultimatly, why do we prefer write through alluxio rather than directly write to HDFS? The former obviously adds some overhead.
* If we do write data to alluxio first, what will happen when machines break down, or alluxio master break down?

## How We Play
The answer to the first consideration is scenario matters.  
We can't change the architecture of the system, but we can choose those metas and its relative data it stores does not matter. In this case, even if a master or worker breaks down, we can just format, restart cluster and reload data from remote, which indicates that those data must not be new created.  
While the performance of writing through is hard to tell, because it is a multi-variable control question. We don't want to take that risk, and keeping fundamental file system stable is our priority. In short, writing scenario is not in our playlist.  
Last but not least, data loaded into alluxio should be frequently read, only in this way can alluxio play its strengths, otherwise causes wasting on memory space as well as network bandwidth to fetch data.  
In conclusion, thinking of easy recoverable and frequently read data, ad-hoc query is on the scene.

## How We Design
First of all, separtion between datanodes and workers is needed. Here are the reasons,
1. Both processes require hard disk to store data, mixed deployment will burden a disk with more violent I/O, increasing the rate of broken disk occurence which is already quite high in our production.
2. Mixed deployment also indicates the sacrifice of storage volume of datanode for purpose of making room for alluxio HDD layer which we refuse to compromise, since most of datanodes already occupied 80~90% of volume, leaving no room for alluxio.  

Second thing comes to mind it is that we need to avoid irrelevant online tasks which do not recognize alluxio scheme being scheduled on alluxio node. And node-label feature of yarn address our concern, we set up an independent and exclusive label cluster for alluxio by marking it "ad_hoc" label.
Following figure shows our architecture.
![Figure 1](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/architecture.jpg)

## Performance Evaluation
To be more prudent to evaluate the performance alluxio can bring, we design an experiment, 4 typical online sqls with differentsizes picked up, and we run these sqls several times on yarn, spark, alluxio and alluxio with only one HDD layer, respectively. More detailed, yarn mode is our online mode, spark mode means tasks running on label cluster but without alluxio as middle layer, and alluxio mode also runs on label cluster with RAM and HDD 2 layers configured, the fourth mode is nearly the same as third mode without RAM layer only. Of course, computation resources are ensured to be obtainable in label cluster as much as online mode.  
General speaking, comparing yarn and alluxio mode is enough, but the former is much more complex than the latter since resources sharing and competition. That's why we include spark mode as a control group. Similiar idea occurs to alluxio 1 mode, as another control group to show the performance gap between with and without MEM layer.  
Following table shows the information of sqls, and figure shows the performance outcome, and cold reading(first time) counts. Unit is seconds for y-axis. 

| Online SQL | Input Size |
| :---: | :---: |
| SQL 1 | 1T |
| SQL 2 | 1.5T |
| SQL 3 | 5T |
| SQL 4 | 300G |
 
![Figure 2](https://raw.githubusercontent.com/Reidddddd/reidddddd.github.io/master/assets/images/performance.png)

From performance figure, we can draw few inferences:
* Overall, alluxio can boost up performance as expectation.
* Even with cold reading, alluxio mode still well performs over others in most cases.
* By comparing alluxio 2 and 1 mode, the performance gap is not so big enough to make user have to use a MEM layer.
* Spark has its own cache and memory mechanism, we have tried some toy examples with small input size, spark is as fast as alluxio or even faster, but as size grows exceeding all executors's jvm memory (in fact, alluxio uses offheap techniques), its performance can't follow.
* One EXCEPTION on SQL 4, we find it reads whole data of a block which may not suit the passive cache mechanism in alluxio, especially to cold reading. 


## What We Have Done
* For spark thrift server, we develop a whitelist feature with which alluxio will load data accordingly, in this way, space in alluxio is made in full use without unnecessary loading and eviction.
* What's more, to make alluxio transparent to user, meaning without modifying any codes or sqls from client side, automatically switching between schemes is developed.  

Therefore, if sql is a query and table involved is in whitelist, then the path of table will be transformed to an uri with alluxio scheme, and application can read it from alluxio. If sql is an execution, it remains the same as origin, runs and writes to remote(hdfs in our case) directly.

## Summary and Future
There's no doubt that alluxio does improve performance in most cases, and we are planing to put it into greater practice by migrating more tasks running on it. In addition, we will take more efforts on security, stability and job monitoring issues and keeping pace with community.  
But few things need second thoughts:
* Fetching data from remote is still an expensive behaviors, and its speed is restrained by network bandwidth;
* Size of label cluster is limited compared to online cluster which comprises thousands of machines, such that RAM we can provided has a low upperlimit;
* What write scenario fits alluxio.

We will come back soon, cheers!

## About MOMO
...
