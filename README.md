# About

Qbox is a pure-Python prototype network proxy. It is the first network proxy capable of coordinating the execution of a distributed saga within a service mesh. 

It solves one of the major problems in migrating monoliths to microservices (preserving all-or-nothing database transactions when data is split across multiple services or shards) while requiring *no* codebase changes, introducing *no* maintenance burden, and being fully distributed (no central points of failure!). 

# Why?

When migrating a monolith, one of the big problems encountered is ensuring data consistency. 

In a monolith, you can issue requests to a single database and trust each request will succeed or fail without leaving the database in a partially-committed state. 

In a microservices-oriented architecture, the "database" is split across many services. If you issue requests to all of these services, some may succeed and some may fail, leading to data inconsistency across the system. Imagine a trip service that tried to register your trip across a hotel, a flight and a car rental - but only the hotel registration succeeded! 

You need a way to preserve the all-or-nothing property a single database provided.

A distributed saga does this. A saga is composed of _transactions_ (requests to a service) and _compensating transactions_ (requests that undo the effect of a transaction). If any transaction fails, compensating transactions are sent out to all services at once. Both compensating transactions and transactions are idempotent: if they are sent twice, the effect should be the same as if they were sent once. Distributed sagas arise naturally where engineering teams have several microservices, each of which have dependencies on each other. 

However, modern implementations suffer from drawbacks:

1. They require a central message broker. This is responsible for ferrying transactions and compensating transactions to microservices. Central message brokers become central points of failure and present a long-term maintenance burden. 

2. Implementing the pattern usually requires changes to your codebase. Enabling APIs that accept idempotent transactions and compensating transactions takes time!

3. They require brand new services called *coordinators*. Coordinators are responsible for checking the state of a saga. They decide when to issue compensating transactions.  

4. Nested sagas (a saga that's started as part of a larger saga) require roundtrips to the central broker, which can be problematic.

All of these issues blow out time to migrate monoliths to microservices substantially. You need expertise in managing a message broker, and you need to test, develop and verify your changed codebase works well in this brave new world. 

We need a simpler way to adopt this pattern.

# Our Solution

Service meshes like Istio have changed the way services coordinate. They act as an invisible message broker between all services in a cluster. Layer 7 network proxies in front of all services intercept and filter messages before forwarding them to applications, allowing security screening, authorization verification and other activities. Best of all, they are capable of *rewriting* messages applications consume or send.

Our solution is simple: turn a service's attached network proxy into a coordinator. Applications can just send a simple high-level request ("please book this trip") and have it be intercepted by the coordinator, which turns it into corresponding low-level transactions and forwards to each of the respective services. Timeouts or failure responses back to the coordinator cause it to issue compensating transactions. 

Why does this help?

1. No central message broker. Service meshes are fully distributed and generally resilient to failures.

2. No codebase changes needed. Sagas are *registered* as configuration with the network proxy. An application can just send what it usually sends, and rely on the proxy's configuration to do the rest. 

3. No separate coordinators - you get one for free with a service mesh.

4. Nested sagas are significantly easy to pull off. 

# Our Work

Under the hood, Qbox is just a glorified state machine embedded in a Layer 7 proxy. 

Source code for Qbox is [here](https://github.com/CSCI-2952-F/qbox). We present a report that doubles as high-level design documentation and benchmark results [here](https://github.com/AkshatM/about-qbox/blob/master/qbox_final_report.pdf). 

To see how well this performs, we wrote Terraform scripts that automatically built GKE clusters in a variety of service mesh topologies using microservices adapted from Istio's bookinfo demo [here](https://github.com/CSCI-2952-F/qbox-cluster). We also built a baseline cluster with a central broker (RabbitMQ) and the same services [here](https://github.com/CSCI-2952-F/baseline-qbox-cluster). 

# Should you use this in production?

Qbox was and will remain a *prototype*. It is a minimum viable product. A short summary why you shouldn't use as-is in production:

1. We chose to only implement support for HTTP requests and no other protocols for simplicity.
2. We chose to implement serial sequential unicast (sending transactions in order one after the other), again for simplicity. This greatly impacted performance, and we now believe sending transactions in parallel would have improved our performance significantly. 
3. Qbox is written in pure Python 3 using its inbuilt HTTP server, rather than anything optimized for fast performance at runtime. 

We believe future work should build on top of existing high-performance service mesh proxies like Envoy, and embed our framework for translating network behaviour into state machine bahviour. This solves all of the prior problems. 

# Authors

Akshat Mahajan and Changhao "Gordon" Wu @ Brown University.
