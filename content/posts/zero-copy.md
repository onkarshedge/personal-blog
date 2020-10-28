---
title: "Zero Copy"
date: 2020-10-28T11:42:14+05:30
draft: true
categories: ["kafka"]
tags: ["distributed-systems"]
---

[Unmesh Joshi](https://in.linkedin.com/in/unmesh-joshi-9487635) is taking an internal workshop on distributed systems. As a case study for assignments we are implementing
a simple version of Kafka. Unmesh has done an exceptional work of pulling code snippets from Kafka codebase and creating a minimal working code skeleton. The code snippets are before Kafka
moved away from the zookeeper.

So some background before moving to what is zero copy. A Kafka cluster consists up of multiple brokers/servers. Assume we have a cluster of 3 brokers. For a given topic "t1" with number of partitions 2 and replicas 2,
one of the broker will be a leader for a partition and other 2 brokers will be followers for that partition. Read/Write requests from Consumer/Producer go to the leader. Writes and reads both going to the leader,
gives a stronger consistency.  
Whenever a producer writes a message it will go to the leader. The leader will serialise the message and append to a file in the filesystem. Yes, the data is persisted in an append-only file.
Benefiting from OS page cache and sequential disk access. The broker will also maintain a message offset to file offset map, so that whenever a consumer requests for messages from a particular offset, It can seek to particular offset in file
and do a sequential read upto the end. The broker will maintain a separate file for each topic, partition.

**Replication**: The follower brokers will keep pulling new messages from the leader since the last read offset, behaving like any other consumer. Just that they will send their broker_id i.e replica_id in the read request.
Upon receiving new messages from the leader they will write to their own file, update the offset and send new read request with offset + 1. This will continue in a loop.
In the image below the grids are append-only files with names as topic_t1_partition_pX for simplicity. The follower replicas send request to the leader of that partition for any new messages and leader responds with new messages.
The follower replicas then append to their own files the data.  
In the below image: Broker-2, Broker-1 are followers for partition-1, they are requesting for new messages to Broker-3 which is the leader for partition-1.
Similarly Broker-2, Broker-3 are followers for partition-2, requesting to Broker-2 for new messages in partition-2.

{{< image src="/images/kafka_replication.jpg" caption="Kafka replication">}}

The leader and follower communicate overt TCP socket. Now the term "Zero Copy" would make sense partially as we need to copy the messages from leader to follower brokers.
But what does "Zero" mean here ?.

When a follower sends a pull request for a partition to the leader eg. `consumeRequest(topic, partition, partition.lastOffset() + 1, brokerId)`, the leader has to read messages from a particular file from a given offset
and send it over the network to the follower. It does not need to do any transformation/processing of the messages. Here is how it would look like:
> read(int file_descriptor, void *file_buffer, size_t count);  
> write(int socket_file_descriptor, const void *socket_buffer, size_t count);

There are two system calls fired `read`, `write` but under the hood the data has been copied multiple times. From kernel space buffer to user space buffer in read system call and from user space buffer
to kernel space buffer in write system call.
    
{{< image src="/images/read_write.jpg" caption="Read and Write separate calls">}}

As you can see we the data is only copied to and fro from kernel space and not processed in the user space, hence Kafka uses a `sendfile system call`. This will avoid copying data multiple times and can benefit from OS page cache.
`sendfile(int out_fd, int in_fd, off_t *offset, size_t count);`  
from Kafka code: 
> logFileChannel.transferTo(offSetRequest, size, socketChannel);

{{< image src="/images/send_file.jpg" caption="Send file copy">}}


