# A Runtime Cache Service?

## The Problem:

We have N components that need to share and operate on (potentially large) blobs efficiently.  We've been using S3 to communicate between blobs.... and it's durable, but very slow....

Wouldn't it be great to just *assume* that we don't have to worry about latency or speed when accessing blobs as we design the runtime?

# An Idea:

- A centralized cacheing service that is (eventually) backed by S3.
    - All DEAs + CC talk to that cache
    - Needs to "not be a single point of failure" - so, perhaps, ETCD is used to connect SHAs to providers
    - Does something like this exist?

- Advantages:
    - One cache to rule them all
    - Not beholden to S3s performance during performance-critical operations
    - Allows us to decouple the DEA from the containers without worrying about data locality
    - Avoid flodding NAT box
    - Easily replace S3 as the backstore for blobs

- Disadvantages:
    - dealing cache coherency
    - robustness is the cache responsibility (vs just S3)
    - need to keep replicated copies of bits to S3
    - never have true data locality

- Design Considerations
    - Use GUIDs for every entry
    - GUIDs are MD5 checksums
    - Horizontal scaling for cache services
    - Generic enough to be used for anyone

# Technologies to explore?
    - Riak? (riak-CS? backed by S3? somehow? seems to have what we need...)
        - For Riak: check this out: http://docs.basho.com/riak/latest/ops/advanced/backends/multi/
    - Gemfire? (possibly overkill?)
    - Memcached with some sort of backend?
    - Redis? (Redis Cluster?) (as a distributed directory service backed by filesystem

# Things to quantify
    - What is the distribution of droplet sizes on prod?
    - Given a prod-like setup (two availability zones, internal network latency, etc.) how long does it take to transfer data from one node to another?
    - Deploy riak and play around with uploading and downloading blobs of different size to/from it.  How performant is this?
    - How much disk space would we need to keep all our bits and droplets within our own cache?
    - Guesstimate: what is a reasoable cache size?  10% of our S3 footprint? 50%?
    - Think about: what are some worst-case scenarios.  E.g. cold start?

# Example Architecture with Riak
    - Clients talk to a load balanced group of HTTP cache-API servers.  The API is restful and simple: POST a blob to a key (md5), GET a blob by key (md5), DELETE a blob by key (md5)
    - The API talks to a Riak cluster with multiple backends (memory and disk)
    - cahche-API servers put things in Riak and also store a copy in a persistent store (S3 on prod, but this should be a pluggable architecture)
    - Simple GET protocol:
        - check in Riak
        - if not there, then get from persistent store, return to user, put in Riak
        - otherwise, return

    - Simple POST protocol:
        - put into Riak (blocking)
        - at the same time: put into persistent store (not blocking)

    - Simple DELETE protocol:
        - delete from Riak (if present)
        - delete from persistent store
