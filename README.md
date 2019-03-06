Disque is an ongoing experiment to build a distributed, in-memory, message
broker.
Its goal is to capture the essence of the "Redis as a jobs queue" use case,
which is usually implemented using blocking list operations, and move
it into an ad-hoc, self-contained, scalable, and fault tolerant design, with
simple to understand properties and guarantees, but still resembling Redis
in terms of simplicity, performance, and implementation as a C non-blocking
networked server.

Currently (2 Jan 2019) the project is in release candidate state. People are
encouraged to start evaluating it and report bugs and experiences.


Dead letter queue
------

Disque jobs are uniquely identified by an ID like the following:

    D-dcb833cf-8YL1NT17e9+wsA/09NqxscQI-05a1

Job IDs are composed of exactly 40 characters and start with the prefix `D-`.

We can split an ID into multiple parts:

    D- | dcb833cf | 8YL1NT17e9+wsA/09NqxscQI | 05a1

1. `D-` is the prefix.
2. `dcb833cf` is the first 8 bytes of the node ID where the message was generated.
3. `8YL1NT17e9+wsA/09NqxscQI` is the 144 bit ID pseudo-random part encoded in base64.
4. `05a1` is the Job TTL in minutes. Because of it, message IDs can be expired safely even without having the job representation.

IDs are returned by ADDJOB when a job is successfully created, are part of
the GETJOB output, and are used in order to acknowledge that a job was
correctly processed by a worker.

Part of the node ID is included in the message so that a worker processing
messages for a given queue can easily guess what are the nodes where jobs
are created, and move directly to these nodes to increase efficiency instead
of listening for messages in a node that will require to fetch messages from
other nodes.

Only 32 bits of the original node ID is included in the message, however
in a cluster with 100 Disque nodes, the probability of two nodes having
identical 32 bit ID prefixes is given by the birthday paradox:

    P(100,2^32) = .000001164

In case of collisions, the workers may just make a non-efficient choice.

Collisions in the 144 bits random part are believed to be impossible,
since it is computed as follows.

    144 bit ID = HIGH_144_BITS_OF_SHA1(seed || counter)

Where:

* **seed** is a seed generated via `/dev/urandom` at startup.
* **counter** is a 64 bit counter incremented at every ID generation.

So there are 22300745198530623141535718272648361505980416 possible IDs,
selected in a uniform way. While the probability of a collision is non-zero
mathematically, in practice each ID can be regarded as unique.

The encoded TTL in minutes has a special property: it is always even for
at most once jobs (job retry value set to 0), and is always odd otherwise.
This changes the encoded TTL precision to 2 minutes, but allows to tell
if a Job ID is about a job with deliveries guarantees or not.
Note that this fact does not mean that Disque jobs TTLs have a precision of
two minutes. The TTL field is only used to expire job IDs of jobs a given
node does not actually have a copy, search "dummy ACK" in this documentation
for more information.

Setup
===

To play with Disque please do the following:

1. Compile Disque - if you can compile Redis, you can compile Disque, it's the usual "no external deps" thing. Just type `make`. Binaries (`disque` and `disque-server`) will end up in the `src` directory.
2. Run a few Disque nodes on different ports. Create different `disque.conf` files following the example `disque.conf` in the source distribution.
3. After you have them running, you need to join the cluster. Just select a random node among the nodes you are running, and send the command `CLUSTER MEET <ip> <port>` for every other node in the cluster.

**Please note that you need to open two TCP ports on each node**, the base port of the Disque instance, for example 7711, plus the cluster bus port, which is always at a fixed offset, obtained summing 10000 to the base port, so in the above example, you need to open both 7711 and 17711. Disque uses the base port to communicate with clients and the cluster bus port to communicate with other Disque processes.

To run a node, just call `./disque-server`.

For example, if you are running three Disque servers in port 7711, 7712, 7713, in order to join the cluster you should use the `disque` command line tool and run the following commands:

    ./disque -p 7711 cluster meet 127.0.0.1 7712
    ./disque -p 7711 cluster meet 127.0.0.1 7713

Your cluster should now be ready. You can try to add a job and fetch it back
in order to test if everything is working:

    ./disque -p 7711
    127.0.0.1:7711> ADDJOB queue body 0
    D-dcb833cf-8YL1NT17e9+wsA/09NqxscQI-05a1
    127.0.0.1:7711> GETJOB FROM queue
    1) 1) "queue"
       2) "D-dcb833cf-8YL1NT17e9+wsA/09NqxscQI-05a1"
       3) "body"

Remember that you can add and get jobs from different nodes as Disque
is multi master. Also remember that you need to acknowledge jobs otherwise
they'll never go away from the server memory (unless the time-to-live is
reached).

Main API
===

The Disque API is composed of a small set of commands, since the system solves a
single very specific problem. The three main commands are:

#### `ADDJOB queue_name job <ms-timeout> [REPLICATE <count>] [DELAY <sec>] [RETRY <sec>] [TTL <sec>] [MAXLEN <count>] [ASYNC]`

Adds a job to the specified queue. Arguments are as follows:

* *queue_name* is the name of the queue, any string, basically. You don't need to create queues, if a queue does not exist, it gets created automatically. If one has no more jobs, it gets removed.
* *job* is a string representing the job. Disque is job meaning agnostic, for it a job is just a message to deliver. Job max size is 4GB.
* *ms-timeout* is the command timeout in milliseconds. If no ASYNC is specified, and the replication level specified is not reached in the specified number of milliseconds, the command returns with an error, and the node does a best-effort cleanup, that is, it will try to delete copies of the job across the cluster. However the job may still be delivered later. Note that the actual timeout resolution is 1/10 of second or worse with the default server hz.
* *REPLICATE count* is the number of nodes the job should be replicated to.
* *DELAY sec* is the number of seconds that should elapse before the job is queued by any server. By default there is no delay.
* *RETRY sec* period after which, if no ACK is received, the job is put into the queue again for delivery. If RETRY is 0, the job has at-most-once delivery semantics. The default retry time is 5 minutes, with the exception of jobs having a TTL so small that 10% of TTL is less than 5 minutes. In this case the default RETRY is set to TTL/10 (with a minimum value of 1 second).
* *TTL sec* is the max job life in seconds. After this time, the job is deleted even if it was not successfully delivered. If not specified, the default TTL is one day.
* *MAXLEN count* specifies that if there are already *count* messages queued for the specified queue name, the message is refused and an error reported to the client.
* *ASYNC* asks the server to let the command return ASAP and replicate the job to other nodes in the background. The job gets queued ASAP, while normally the job is put into the queue only when the client gets a positive reply.

The command returns the Job ID of the added job, assuming ASYNC is specified, or if the job was replicated correctly to the specified number of nodes. Otherwise an error is returned.


===

* Disque is new code, not tested, and will require quite some time to reach production quality. It is likely very buggy and may contain wrong assumptions or tradeoffs.
* As long as the software is non stable, the API may change in random ways without prior notification.
* It is possible that Disque spends too much effort in approximating single delivery during failures. The **fast acknowledge** concept and command makes the user able to opt-out this efforts, but yet I may change the Disque implementation and internals in the future if I see the user base really not caring about multiple deliveries during partitions.
* There is yet a lot of Redis dead code inside probably that could be removed.
* Disque was designed a bit in *astronaut mode*, not triggered by an actual use case of mine, but more in response to what I was seeing people doing with Redis as a message queue and with other message queues. However I'm not an expert, if I succeeded to ship something useful for most users, this is kinda of an accomplishment. Otherwise it may just be that Disque is pretty useless.
* As Redis, Disque is single threaded. While in Redis there are stronger reasons to do so, in Disque there is no manipulation of complex data structures, so maybe in the future it should be moved into a threaded server. We need to see what happens in real use cases in order to understand if it's worth it or not.
* The number of jobs in a Disque process is limited to the amount of memory available. Again while this in Redis makes sense (IMHO), in Disque there are definitely simple ways in order to circumvent this limitation, like logging messages on disk when the server is out of memory and consuming back the messages when memory pressure is already acceptable. However in general, like in Redis, manipulating data structures in memory is a big advantage from the point of view of the implementation simplicity and the functionality we can provide to users.
* Disque is completely not optimized for speed, was never profiled so far. I'm currently not aware of the fact it's slow, fast, or average, compared to other messaging solutions. For sure it is not going to have Redis-alike numbers because it does a lot more work at each command. For example when a job is added, it is serialized and transmitted to other `N` servers. There is a lot more message passing between nodes involved, and so forth. The good news is that being totally unoptimized, there is room for improvements.
* Ability of federation to handle well low and high loads without incurring into congestion or high latency, was not tested well enough. The algorithm is reasonable but may fail short under many load patterns.
* Amount of tested code path and possible states is not enough.

Disque routing is not static, the cluster automatically tries to provide
messages to nodes where consumers are attached. When there is an high
enough traffic (even one message per second is enough) nodes remember other
nodes that recently were sources for jobs in a given queue, so it is possible
to aggressively send messages asking for more jobs, every time there are
consumers waiting for more messages and the local queue is empty.

However when the traffic is very low, informations about recent sources of
messages are discarded, and nodes rely on a more generic mechanism in order to
discover other nodes that may have messages in the queues we need them (which
is also used in high traffic conditions as well, in order to discover new
sources of messages for a given queue).

For example imagine a setup with two nodes, A and B.

1. A client attaches to node A and asks for jobs in the queue `myqueue`. Node A has no jobs enqueued, so the client is blocked.
2. After a few seconds another client produces messages into `myqueue`, but sending them to node B.

During step `1` if there was no recent traffic of imported messages for this queue, node A has no idea about who may have messages for the queue `myqueue`. Every other node may have, or none may have. So it starts to broadcast `NEEDJOBS` messages to the whole cluster. However we can't spam the cluster with messages, so if no reply is received after the first broadcast, the next will be sent with a larger delay, and so foth. The delay is exponential, with a maximum value of 30 seconds (this parameters will be configurable in the future, likely).

When there is some traffic instead, nodes send `NEEDJOBS` messages ASAP to other nodes that were recent sources of messages. Even when no reply is received, the next `NEEDJOBS` messages will be sent more aggressively to the subset of nodes that had messages in the past, with a delay that starts at 25 milliseconds and has a maximum value of two seconds.

In order to minimize the latency, `NEEDJOBS` messages are not throttled at all when:

1. A client consumed the last message from a given queue. Source nodes are informed immediately in order to receive messages before the node asks for more.
2. Blocked clients are served the last message available in the queue.

For more information, please refer to the file `queue.c`, especially the function `needJobsForQueue` and its callers.

Are messages re-enqueued in the queue tail or head or what?
---

Messages are put into the queue according to their *creation time* attribute. This means that they are enqueued in a best effort order in the local node queue. Messages that need to be put back into the queue again because their delivery failed are usually (but not always) older than messages already in queue, so they'll likely be among the first to be delivered to workers.

What Disque means?
---

DIStributed QUEue but is also a joke with "dis" as negation (like in *dis*order) of the strict concept of queue, since Disque is not able to guarantee the strict ordering you expect from something called *queue*. And because of this tradeof it gains many other interesting things.

Community: how to get help and how to help
===

Get in touch with us in one of the following ways:

1. Post on [Stack Overflow](http://stackoverflow.com) using the `disque` tag. This is the preferred method to get general help about Disque: other users will easily find previous questions so we can incrementally build a knowledge base.
2. Join the `#disque` IRC channel at **irc.freenode.net**.
3. Create an Issue or Pull request if your question or issue is about the Disque implementation itself.

Thanks
===

