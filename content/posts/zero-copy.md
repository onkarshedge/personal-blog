---
title: "Zero Copy"
date: 2020-10-28T11:42:14+05:30
draft: false
categories: ["kafka"]
tags: ["distributed-systems"]
---

#### Where & When did I come across this term 
[Unmesh Joshi](https://in.linkedin.com/in/unmesh-joshi-9487635) is taking an internal workshop on distributed systems. As a case study for assignments we are implementing
a simple version of Kafka. Unmesh has done an exceptional work of pulling code snippets from Kafka codebase and creating a minimal working code skeleton. The code snippets are before Kafka
moved away from the zookeeper.

#### Problem
We want to transfer some file contents, which are accessed sequentially over the network through TCP socket. Application process does not transform the data.
How do we transfer it efficiently ?  

#### Solution Iteration 1
We use the high level programming language features to open, read, seek to a particular offset and close the file. Accept socket connection, write to the socket, close it. 
> read(int file_descriptor, void *file_buffer, size_t count);  
> write(int socket_file_descriptor, const void *socket_buffer, size_t count);

Pretty straightforward right ?. Let's see what's happening under the hood. There are two system calls fired `read`, `write`. The data has been copied multiple times. From kernel space buffer to user space buffer in read system call and from user space buffer
to kernel space buffer in write system call. System calls are relatively expensive computationally. There are 4 context switch between userspace and kernel space. One of the reasons the kernel space exists is for protected environment (remember ring0, ring3 in x86 architecture ?), abstraction of access to physical devices etc.  

{{< image src="/images/read_write.jpg" caption="Read and Write separate calls">}}

{{< admonition type=question title="Question" open=true >}}
As we don't want to process the data in application can we reduce the number of data copies ?
{{< /admonition >}}

#### Solution Iteration 2

Other approach is using `mmap()` system call instead of read. The number of system calls will still remain the same.
The number of context switches will still be the same. The difference is that process reads/writes to shared memory region i.e page cache. No copy needs to be created in user space.
Reads/Writes happen in size of page sizes (usually 4kB). OS will lazily load pages on request.
mmap is useful when reading portion of a file, random access, share memory with other processes. Not performant for reading very large files. Time has been traded off for space.  
Can we do better ?

#### Final Solution
As we know the data is only copied to and fro from kernel space and not processed in the user space. We can leverage `sendfile system call`. This will avoid copying data multiple times.
If lots of requests come in for the same file. The OS will keep the page cache in memory and very few requests will go to the disk.
`sendfile(int out_fd, int in_fd, off_t *offset, size_t count);`  

{{< image src="/images/send_file.jpg" caption="Send file copy">}}

{{< admonition type=note title="How is this zero copy ?" open=true >}}
There are still 2 DMA copies and 1 CPU copy. Direct Memory Access (DMA) is a technique for transferring blocks of data between system memory and peripherals
without a processor (e.g., system CPU) having to be involved in each transfer.
It is asynchronous operation. The DMA Controller will interrupt the CPU once the data is copied, meanwhile CPU can do other tasks. No Clock cycles wasted.  
Okay but what about 1 CPU Copy ?. The only CPU copy can also be eliminated if the DMA supports scatter/gather operations. Aggregating multiple physically-disjoint memory regions into a single data transfer is called "scatter-gather".
More on [TCP segmentation](https://lwn.net/Articles/9123/)
{{< /admonition >}}

#### Zero Copy in Kafka
##### Some Kafka terminology
A Kafka cluster consists up of multiple brokers/servers. Assume we have a cluster of 3 brokers. For a given topic "t1" with number of partitions 2 and replicas 2,
one of the broker will be a leader for a partition and other 2 brokers will be followers for that partition. Read/Write requests from Consumer/Producer go to the leader. Writes and reads both going to the leader,
gives a stronger consistency.  
Whenever a producer writes a message it will go to the leader. The leader will serialise the message and append to a file in the filesystem. Yes, the data is persisted in an append-only file.
Benefiting from OS page cache and sequential disk access. The broker will also maintain a message offset to file offset map, so that whenever a consumer requests for messages from a particular offset, It can seek to particular offset in file
and do a sequential read upto the end. The broker will maintain a separate file for each topic, partition.

##### Replication
The follower brokers will keep pulling new messages from the leader since the last read offset, behaving like any other consumer. Just that they will send their broker_id i.e replica_id in the read request.
Upon receiving new messages from the leader they will write to their own file, update the offset and send new read request with offset + 1. This will continue in a loop.
In the image below the grids are append-only files with names as topic_t1_partition_pX for simplicity. The follower replicas send request to the leader of that partition for any new messages and leader responds with new messages.
The follower replicas then append to their own files the data.  
In the below image: Broker-2, Broker-1 are followers for partition-1, they are requesting for new messages to Broker-3 which is the leader for partition-1.
Similarly, Broker-2, Broker-3 are followers for partition-2, requesting to Broker-2 for new messages in partition-2.

{{< image src="/images/kafka_replication.jpg" caption="Kafka replication">}}

The leader and follower communicate overt TCP socket.
When a follower sends a pull request for a partition to the leader eg. `consumeRequest(topic, partition, partition.lastOffset() + 1, brokerId)`, the leader has to read messages from a particular file from a given offset
and send it over the network to the follower. It does not need to do any transformation/processing of the messages.
> logFileChannel.transferTo(offSetRequest, size, socketChannel);
  



