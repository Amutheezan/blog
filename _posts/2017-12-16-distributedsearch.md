---
type: posts
title: Distributed Search
author: Amutheezan Sivagnanam
category: Distributed Systems
date: 2017-12-16
tags:
- distributedsystems
- clusters

---

This is a short descriptive post based on our project done for CS4262 Module, Distributed Systems at the University of
Moratuwa. These particular wordings are my own words and thus it doesn't be exact same of what I have submitted as final
report to the course assignment .

### Implementation

![](https://lh5.googleusercontent.com/rrB9txWqf-1HsZCH8Oq2kbjJAN-DfM5JLWw8b2s2yHPjRTn-uHt6mM4xkLq6MOc2nNM3i4jL3NjKHJACjTdh-nuOzfglTbvzpqNZctqxfZ1m8F8c1jz0L4A1HV7fiOmasMVeyPO0)

We establish an unstructured peer-to-peer overlay. Our overlay network structure similar to the figure. This gives us
good results when the peers get killed iteratively. Each node will broadcast the search queries to all its peers. This
is how the search query is propagating across the network.

In this design, we need to handle the “query duplication” which is the cause to increase the message flow in the
network. We avoid query duplication by introducing a UUID in addition to the existing messaging protocol for search. (
UUID is a 128-bit number which is expected to produced as random, but the probability of generating the same number
again is nearly equal to zero, not zero, so we used this ID for a particular search query to not asked again on the same
node)

Our new protocol.

```length SER UUID IP port file_name hops```

Each search query is uniquely identified UUID. Each node maintains a history of which is the search queries already
arrived at that node with respect to the UUID. If an incoming search query is not in the history, then it will either
find on its system or broadcast to its peers else if it already exists it will just discard the search query. This
strategy prevents the duplication of the same search query as well as unwanted message flows and helps the systems to
stable over message overflow of SER messages. This will considerably reduce the number of messages across the network.

#### If the number of queries (Q) is much larger than the number of nodes (N) (Q >> N)

Since our solution is based on the broadcast of the search query to all the peers it connected when a number of queries
increased each node gets more search query requests. So this will eventually increase the message count and latency for
a search query resolution.

So we could apply some caching methods on each node. Already searched files and their details are cached with their
location in nodes. Then further searches on the same files can be obtained quickly and with negligible latency and hops.
But this will raise another problem such as file collection may outdated with time. Since (N>>Q) the gain we got from
broadcasting must outweigh the cost of broadcasting. So when sending the request nodes may apply a biased random walk
rather than broadcast the query to all nodes. This will reduce the latency and hops considerably due to reduced message
flows in all nodes with fewer amount of requests.

#### If the number of nodes (N) is much larger than the number of queries (Q) (N >> Q)

Since our solution broadcasts the search query to all peers and propagated in the same way to the rest of the network
when the number of nodes is much higher than the number of search query messages drastically increased. This will
increase the latency as well as hop count.

To handle this we introduce we can introduce a limit for hops in other words time to live, thus after n number of hops,
it won’t propagate. This will limit the propagation of query beyond a limit and control the number of search query
messages also reduces the latency due to fewer messages per node. But has some issues on identifying files/ contents if
none of the peers of peers are connected to the files having nodes to a certain level.

We can apply the **super-peer**, which contains the all resources and only the super-peer can broadcast or random walk.
We can also share file collection in order to have a biased random walk between super peers. This will significantly
reduce the number of messages, latency, and hop count for query resolution.

My Implementation can be found in the [repo](https://github.com/Amutheezan/DistributedSearch).

#### Performance Analysis.

My implementation has two version

1. udp (check the ``udp`` branch of the repo)
2. webservice (check the ``ws`` branch of the repo)

##### UDP Sockets

We tried 50 sample queries in three selected nodes and get the minimum, maximum, average, and standard deviation for
hops, latency, node degree, and message per node in different cases (1. all nodes are running, 2. one node is gracefully
departed, 3. two nodes are gracefully departed). Nodes are using UDP sockets for communicating between them. At the end
of each case, we gracefully leave one node and continue with the next iteration. And we obtained the following results

![](https://lh5.googleusercontent.com/W8igfNHPirsNIT-T2LjN4pZ2R-lka_sWaGJqXM4s33bwIb0-zxwsyUZYSIc5Qd89AQve8eCgoFR6NPuzjNbhscuKV4imw8emVyaXyq631u11vyS0Ho69PS9zKETA8LxBJgd-LiIF "Chart")![](https://lh6.googleusercontent.com/8J6JxdKQoxIraRS5Lu1cYhY9TB9lTYB8EH0eMnkHrCc7zq1AkXv5e8B1T_YrL6i95CNK36pKrpzF7C_XJZPzn_ohARgSmzd4cXFao7NRvpEi0eDruf8LUCc7TpRRgYkE5DUj6Syi "Chart")

![](https://lh4.googleusercontent.com/pmZoNTc6kGsFqOVLwGYwB65vnxqGnYkgpQBDY4DcBpaXJ0r7ULzHxgQq6KIDeZ1zFN2kiGI3kfxvCobOkMx37ptQRdiCnFy8w48JQR111jawl5ytuFkrVqIY2X9Zh9reoF9uUwKB "Chart")![](https://lh3.googleusercontent.com/JVMDX3bKwNq9clX_5SLsZJ4jf6Hf7-nMs9FHA6lj6NtzMintZ1aIHhO_SXPE2IAqyG4KdlBE2avbr4eQK71DByhtzZBtdvobj1K1X0LCh9sqjnde6yUuNlZsWET8nTy9zlrZuPis "Chart")

##### REST API/WS

We tried 50 sample queries in three selected nodes and get the minimum, maximum, average, and standard deviation for
hops, latency, node degree, and message per node in in different cases (1. all nodes are running, 2. one node is
gracefully departed, 3. two nodes are gracefully departed). Nodes are using REST API such as GET/POST for communicating
between them. At the end of each case we gracefully leave one node and continue with the next iteration. And we obtain
the following results following results

![](https://lh5.googleusercontent.com/_EivtGlQY1l_M5VQ1-FOzznshr0Pfa62CG6U2vOomRqooPltWeLjdmfHB3cVeuIPUc9cAv69W4HpfhJ7QV6uWo73NG_UI6T2ybWcvuWxcYXkrCrRkpDqywwQfuujSfdqUPxHqA-3 "Chart")![](https://lh3.googleusercontent.com/6gquXSrmCuN0EqJ8ijQTLfQDJ4QETVPVr3P-PpJPRF1PE2f8lTxiv3r1AHtVbFs00A6uNCb-_R7sPajSRwkCUofg9ltEzcD4AgYSd_aq3mAYFop3_4AsdbbbS_1Sw7KdM1wZU0Bf "Chart")![](https://lh4.googleusercontent.com/Xt181qjM3MhZd4p6hONx-BfHcSrra93kVZtP_X1lDmXP0NiH_YeqxozP7JoCmguCfpNUeWkJ8Y0Q2cf-yCtVF_gwAw9b6B48paxSLKOL5UOp5Op4-DMdy-PaFg06hnk0SHlT46J7 "Chart")![](https://lh3.googleusercontent.com/s7NTmoMgAmDIxQMza1MM8igeji7O0rTwd-bjUL31fj5mXndDAtqKdZRujbtMnQ67hh1R37oZ9F3Oj2685VzSaqmlLMKnKUkkZvFSTGYhi3r6gObfCmGybYOBeYagolEw6eiZ_Oii "Chart")

Hope you enjoy this post
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU4NzA5NzY2Niw5MzM4MDMyNDYsNjc2OT
Q0MTNdfQ==
-->