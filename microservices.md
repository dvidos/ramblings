# On Microservices

## Purpose

This document contains some notes on microservices. Aimed mostly at Engineers and Product Managers.

## What

Microservices, at their essense, are a construct to allow teams to work faster in a corporation.
Their size is dictated by what can be maintained by a single team.

## Why 

Microservices allow for separate deployments. This means a team is free to implement, deploy, monitor, fix and improve a microservice,
without depending on any other team.
In big organizations interteam communication and coordination is difficult, as prioritizations are different from team to team.

## How

By creating separate repositories and allowing each team to deploy whatever services it needs to fulfill the business needs.
The functionality boundaries should be such that the team is empowered to provide the full functionality needed, 
with as little dependencies as possible.

Usually, a virtualized environment is used (i.e. Kubernetes clusters) but other independenct deployment methods also allow for this.
Also, usually direct HTTP services are implemented (as they make good clear contracts), 
but asynchronous communications are also immensely valuable. 
In messaging solutions, RabbitMQ is fast and agile, but Kafka offers durable storage of messages, 
guaranteeing no loss in case of loss of nodes.

## Code specific 

Try to define the logical boundaries of the microservice around a concern, e.g. e-commerce checkout or resources allocation.
This will naturally gravitate towards a few central entities, but avoid the temptation to make a generic CRUD operation.
All write operations should be explicit on their intent.

If the service has any business logic that needs to be maintained, try to approach using DDD.
DDD gives us the guarantee that all changes pass through the domain objects and services, therefore 
maintaining the invariables of our business logic. Sometimes it may seem overkill, but it is very worth it.

If you will have any asynchronous communications (e.g. kafka consumers), you'll need a Dead Letter Queue
or other means to put bad messages aside, to avoid blocking the stream, in case of a bad message.
Engineers or other users should then take a look at those messages to understand what the problem is
and take appropriate action: retry the message after solving the problem, or marking the messages as ignored.

### Scaling the Read Model

If you need to scale the read operations (i.e. reading queries are too slow) then you can build a read model,
tailored to the specific reading use cases. Practically, you will asynchronously maintain a collection
of cached entries (or other fast key-based retrieval technology) to provide very fast reads.

The write side of the services (still operating on the write model in the storage engine) 
will enqueue async commands to an internal topic that will update 
the affected keys after every write operation.

### Concerns of the Write Model

When writing in SQL databases we need to address two important issues: atomic changes and concurrent checks.

Atomic changes mean we will update all or none of our tables, we cannot leave the database in a bad state 
if we crush midway through our updates. SQL gives us the guarantee to do this using transactions.

Concnurrency checks are there to avoid race conditions in read-update-write cycles. 
In DDD and other approaches, we often follow a pattern of loading some data into memory, 
running validations and checks, updating things into memory, then persisting the changes 
in tables. We want to make sure that, if another process had updated the specific rows
while we were working, we'll detect that, so we can start the read-update-write cycle again.

### Dual Writes

Avoid writing to two or mode systems at a time, for example to a database and then to a message queue. 
Such operations should be atomic, either both succeeding or both failing. To achieve that,
prefer to have an Outbox table or list in your storage engine that you insert to, 
atomically with the rest of the changes. 
Then, have a data connector read those messages and populate them to the messaging queue.

Another approach is to have both the service and the messaging engine use the same storage 
engine.

### Scaling the Write model

Usually we will not encounter problems with writing scale, unless we have a very heavily shared
database server, where exceedingly lots of updates are happening at the same time.

The first line of defence is to move to a dedicated database server.
If our writes still struggle and we need to scale, 
we can consider sharding our data, that is using a universal identifier to split the storage 
into two or more stoage solutions.

## Event driven architecture

If we produce events (asynchronous messages) everytime that something interesting happens in our service,
peripheral services may consume those to react to the change. 
For example, if a new customer is registered, maybe a peripheral service will send a welcome email message.

Those events can be from very simple, containing only the semantics of the change and an aggregate identifier,
all the way to very complex, containing big trees of data, including links to extra resources for extra information.

The inverse is also useful, if there are events emitted by peripheral microservices, 
our application can stay in sync with them, without needing a direct, synchronous dependency.

A widespread adoption of events takes us towards an even driven architecture, where most of the 
processing is done at the time of the change, to at the time of the user request. If fast read models
are maintained, this allows for very fast response to otherwise very time consuming queries.

## Avoiding failure propagation

A microservice that consumes events is also avoiding failure propagation. A synchronous HTTP dependency
will cause the failure to expand, as any services that depend on the failed service, will also fail.

Using messages allows us to avoid this pattern. Any consuming service will see messages flow drop, or even stop.
But when the service is back up, any processing will resume from the point it was left.

This allows for a very resilient scheme, where all services just respond to messages with more messages.
The net result is that small downtimes here and there will hardly be noticed or cause a major outage,
provided the messaging medium (e.g. kafka) is healthy.

This is the base of an autonomous, resilient microservice.






