# On Microservices

## Why 

For separate deployment, to avoid one team slowing down another. In big organizations interteam communication and coordination is costly.

## How

By creating separate repositories and allowing each team to deploy whatever services it needs to fulfill the business needs.
The functionality boundaries should be such that the team is empoered to provide the full functionality needed, 
with as little dependencies as possible.

Usually, a virtualized environment is used (i.e. Kubernetes clusters) but other independenct deployment methods also allow for this.
Also, usually direct HHTTP services are implemented (as they make good contracts), 
but asynchronous communications are also immensely valuable. 
RabbitMQ is fast and agile, but Kafka offers durable storage of messages, guaranteeing no loss in case of failures

## Code specific 

Try to define the boundaries of the microservice around a concern, e.g. e-commerce checkout or resources allocation.
This will naturally gravitate towards a few central entities, but avoid the temptation to make a generic CRUD operation.
All read-write operations should be explicit on their intent.

If the code has any business logic that needs to be maintained, try to approach using DDD.
DDD gives us the guarantee that all changes pass through the domain objects and services, therefore 
maintaining the invariables of our business logic. Sometimes it may seem overkill, but it is very worth it.

If you will have any asynchronous communications (e.g. kafka consumers), you'll need a Dead Letter Queue
or other means to put bad messages aside, to avoid blocking the stream, in case of a bad message.
Engineers or other users should then take a look at those messages to understand what the problem is
and take appropriate action: retry the message after solving the problem, or marking the messages as ignored.

### Scaling the Read Model

If you need to scale the read model (i.e. reading queries are too slow) then you can build a read model,
tailored to the specific reading use cases. Practically, you will asynchronously maintain a collection
of cache (or other fast key-based retrieval technology) to provide very fast reads.
The write side of the services will enqueue async commands to an internal topic that will update 
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

### Scaling the Write model

Usually we will not encounter problems with writing scale, unless we have a heavily shared
database server, where exceedingly lots of updates are happening at the same time.

If we move to a dedicated database server and our writes still struggle and we need to scale, 
we can consider sharding our data, that is using a universal identifier to split the storage 
into two or more stoage solutions.



