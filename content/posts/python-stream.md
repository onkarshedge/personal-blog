---
title: "Http Stream with python"
date: 2017-10-25T19:30:05+05:30
tags: ["http", "python"]
draft: false
---

When downloading a large file via http it has to be in chunks because lot of memory would be used or sometimes it would not fit also depending on RAM size.

Chunked transfer encoding is a streaming data transfer mechanism available in the Hypertext Transfer Protocol (HTTP), version 1.1 and higher. In chunked transfer encoding, the data stream is divided into a series of non-overlapping "chunks". The chunks are sent out and received independently of one another.

##### HTTP header "Transfer-Encoding: chunked"  
When  used with "Content-Encoding: gzip" the stream is first compressed and then chunked. from [wikipedia](https://en.wikipedia.org/wiki/Chunked_transfer_encoding)

###### Encoded data
```
4\r\n
Wiki\r\n
5\r\n
pedia\r\n
E\r\n
 in\r\n
\r\n
chunks.\r\n
0\r\n
\r\n
```

###### Decoded data

```
Wikipedia in

chunks.
```
##### Python requests
In requests library you can use this via.
```
resp = requests.get(uri, stream=True)
with open(local_filename, 'wb') as f:
        for chunk in r.iter_content(chunk_size=1024):
            if chunk: # filter out keep-alive new chunks
                f.write(chunk)
```
