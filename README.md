# DistributedSystems

## 8 fallacies of distributed systems
  * [here](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)
  * The network is reliable - make sure your code is resillient to network failure.
  * Latency is zero
  * Bandwidth is infinite
  * The network is secure
  * Topology doesn't change - single database is an example of this. Be prepared for database failovers and/or distributed databases.
  * There is one administrator
  * Transport cost is zero
  * The network is homogeneous - happens when the system is designed so two different applications need to be deployed at the same time. Take care of versioning to mitigate this.

## Properties of reliable systems

  * Idempotence
  * Immutability
  * Location independence
  * Versioning
  * These properties are discussed in detail below. The reason for these properties is they mitigate the fallacies we highlighted above: the network is reliable (idempotence), topology doesn't change (location independence) and the network is homogeneous (versioning).
  * The above properties can be applied to any system, they are not meant necessarily for distributed systems. Applying these principles will bring value to any type of system.

### Idempotence

* Networks are not reliable. If you don't get back a successful reply, you cannot discern between the server not getting the message, an error on the server and server executing successfully but the response somehow getting lost.
* Our system must be prepared to detect a duplicate write and ignore it.
* We do this by generating a client-side generated id and then POSTing (and PUTing, but PUT is idempotent as it is) against that identifier. Guid is a good choice for this. This will act as an alternate key in the database.
* In the course [Fundamentals of Distributed Systems](https://app.pluralsight.com/course-player?clipId=d4145cd4-2477-4a20-ae65-af9476ef06fc) Michael Perry also talks about not returning the Id back to the client. Not sure what that is about. An implication of this is that the client will use the client Id to query the resource and this decision then permeates the interfaces found throughout the application layers.

### Immutability

* Never change the entity, but rather add a new record. Benefits:
  * We have a natural audit log
  * Nothing gets destroyed
  * We preserve what data each node received ("metadata")
  * Promotes eventual consistency (not sure how immutability relates to this one?)
* Immutability changes everything [here](https://vimeo.com/52831373)
* Facilitated by:
  * Snapshot pattern - a separate snapshot table keeping each change, thus simulating updates. Besides keeping the data it also allows for detecting concurrent writes by comparing the incoming tick number with the last stored tick number. The caller must provide the exact same tick number otherwise they are working with an out-of-date version. Important thing to note here is the concurrency boundary begins when the data is read. More on my discussion with Michael Perry in the bottom of this article [here](#Discussion-with-Michael-Perry-on-snapshot-pattern-and-concurrency-checks).
  * Tombstone pattern is used to simulate deletes.

### Location independence
* Location dependent identifiers like auto-incrementing Ids are an anti-pattern in distributed systems: One record cannot be moved from one database to another because the id might be taken by another record.
* To maintain location independence you should hide or even not use auto-incrementing IDs. Use client generated IDs.
* Location independent identifiers:
  * Alternate Key
  * Natural Key
  * Public Key
  * Hash - a.k.a. Content Addressed Storage, using the content to identify itself. This value is the same irrelevant of which node does the calculation.
    * One use is to do cache busting for client-side cached components (e.g. JS files). However, if caching is done based on query string (`file.js?v=123`), then this does not play well with rolling deployments where we might have multiple versions of `file.js` deployed - request for `file.js?v=456` will bust the local cache, but the load balancer will still forward the request to one of the deployed versions of the application and it might not necessarily be the one where `file.js?v=456` is stored, thus potentially retrieving an older version (`file.js?v=123`) and causing the browser to store the wrong version as `file.js?v=456`. If we name our resources according to the hash derived from the resource (`file-123.js`) than cache busting will work with multiple versions deployed.
    * Advantages:
      * Naturally immutable: if the hash is different, it's a different object.
      * Naturally verifiable: if the hash you calculate does not match the hash of the object you just received, it has been tampered with.
      * Naturally idempotent: if the hash of the object you hold is the same as the hash of the object you want to replace, no need to replace it.
      * Solves cache invalidation: if the cache contains an object with a given hash, you know it's up to date. More [here](https://app.pluralsight.com/course-player?clipId=26c73001-f898-4ea0-a6aa-2cc41fe14185).

### Versioning

#### Code versioning

* Source control management software (Git)
* CI/CD pipeline and separate components for versioning assemblies

#### Database versioning

* When versioning the database (e.g. new columns) favour creating new tables as opposed to just adding new columns. By just adding new columns we have to make them nullable or provide a default value to accomodate existing records - this sends a message that all records have this default value (which is probably not true) or that the columns is optional. By creating a new table we are sending a clear message - data is optional. Of course if the new columns are not optional, consider including them in the existing tables, but think about how that relates to the nature of having immutable records.

#### API versioning

* One of the fallacies of distributed computing says "Network is homogenous". What this means is that we erroneously might believe all the nodes out there are of the same version (e.g. all instances of some specific service are of the same version). This is not true if we want to have things like rolling deploys and backwards compatibility.
* Versioning plays into this by allowing us to maintain contracts across different versions of applications across different locations and different times these are deployed.
* Versioning can be achieved through several approaches:
  * HTTP headers: `Accept=application/some_resource.v1+json`, `Content-Type=application/some_resource.v1+json`
  * Query string: `/api/some_resource?version=v1`, but this kind of defeats the purpose of query string
  * Route: `/api/v1/some_resource`
  * Additive versioning: different versions of the API work with different versions of the payload. Newer versions simply know how to read additional properties in the payload.
  * Or you can avoid it altogether by using GraphQL, allowing the clients to shape their own responses.

### Open points
* Why are we hiding Id from clients?
* Benefits of immutable data?
* Content Addressed Storage - how does load balancer know where to find `file-123.js` as opposed to `file-456.js`?

### Discussion with Michael Perry on snapshot pattern and concurrency checks

![image](https://user-images.githubusercontent.com/26722936/140316543-f75669ff-62ad-41a0-9c6c-2cd912918aa2.png)

## Connecting Services

### Types of messages

#### Commands

* Clients tell the system of record to perform some action. As a rule commands should return any value.

#### Queries
* Clients ask the system of record to return some data. As a rule queries should not change anything.

#### Events
* Emitted by the system of record to signal an action was performed resulting in change of the system's internal state. As a rule should be expressed in past tense.

### Temporal Coupling

#### Synchronous Coupling
* HTTP, gRPC, REST, SOAP
* Allows the delivery to take place and server to validate the incoming payload, do the processing and return the result indicating how the processing went.
* Commands can be done using a synchronous manner, but then the client and the server are temporaly coupled, which is something we should strive to avoid. Benefits are the client is informed whether the command went through successfully or not.
* Queries should use synchronous coupling, but in specific cases (e.g. performance) they can be executed using an asynchronous call.

#### Asynchronous Coupling
* HTTP (202 Accepted), AMQP, Apache Kafka, Amazon SQS and others
* Allows the delivery to take place and server to validate the incoming payload. Business invariant cannot be checked, but information can be returned to the client on how to check in later.
* Commands should be done using an asynchronous communication. Client should be returned a 202 Accepted and the processing should be done in the background as this decouples the client from the server. This does make the system more complicated overall as the client now has to check back to see the result of the processing.
* Events should be done asynchronously because the system of record is publishing these to let others know it has finished an action - it is not interested in who picks up the message and therefore decoupling is paramount.

### Publishing Contract Packages

* For services to communicate asynchronously they need to know which events each of them works with - events form an interface.
* To stay on top of the interfaces and any changes that might happen to these interfaces a service should publish events it works with (consumes). This is best done by creating a separate project describing the events and make it part of same solution as the consuming service. Publish the contract package to NuGet (private feed) so other services know what the events should like. Use semantic versioning to convey information whether the new interface introduces a breaking change.
* The section below talks about how to keep track of Dtos, but a different approach is suggested there. Since the above approach is suggested by Michael Perry and the approach below by Roland Guijt, I still have to see if the two are in conflict.

#### API Data Transfer Objects

* You might be tempted to share Dtos used by microservices, but don't do that. This leads to tight coupling between microservices.
* Another option is generating Dtos from OpenApi specification. If you are ok with the generated code, it's a viable approach. Updating the Dtos is now done manually.
* Third option is to simply for each service to have its own copy of Dtos and go with that. Updating the Dtos is now done manually.

#### Generating Data Transfer Objects

* This approach utilizes OpenApi specification and tooling to generate Dtos. If you are ok with the generated classes and they way they look like, this is an ok approach.
* Otherwise, you might have to resort to manually creating the Dtos as per the specification - there are tools online for this as well, so you just c/p resulting classes.

### Integrating external Partners

* This section describes how external clients might integrate with your distributed system.
* Use HTTP in an asynchronous manner, returning 202 Accepted. This way you can avoid temporaly coupling with your partner's APIs and you also sidestep a few other issues your partners might have with publishing messages to your messaging queue:
  * Messaging queues tend to be implementation based and not standards based (see list above, they are all products, not standards).
  * Authentication issues by 3rd parties.
* By using HTTPs you can also:
  * Create OpenAPI specification for your partners to consume.
  * Utilize your platform's API Management tools to issue API keys your partners can use to authenticate.
  * Create Postman collections and reference implementations for your partners to integrate more easily.

## Connecting Microservices

### Commutativity

* It is a real possibility two messages which have been queued in a specific order might be processed out of order. Therefore it is important for our system to be able to process them out of order and yet have the desired result happen.
* An example is first creating a new show for an act (show added event) and then renaming that act (act description changed event). This was important for Perry's search service. When the act is renamed all shows get the new name. When a new show is created the act name is embedded in the event. If these two events get processed the other way round then first all the shows belonging to the specific act get renamed and only then the new show will be created, but with the old name. He solved the problem by indexing the act name in a separate index. This way any show created can read the last indexed act name from the act index and compare to the act name in the show added event - if the indexed act name is newer, use that - if the indexed act name is older, replace with the act name from the show added event. This approaches hinges on the act index storing metadata (last modified date).
* I think details on how to implement commutativity are very case-specific.

## Why use microservice architecture at all?

* Flexible scaling
* Independent deployment
* Team separation
* Easy to expand
* Reliable
* Different technologies: languages, types of interfaces (REST, gRPC)

## Tracing

* Every incoming HTTP request should get the web framework to generate an identifier. ASP.NET Core implements the HTTP Trace Context standard.

## Saga

* Essentially a distributed transaction that executes as a series of local database transactions.
* Consists of three phases executed consecutively:
  1. Compensatable transactions - these can be reverted by applying transactions that cancel out original transactions.
  2. Single pivot transaction - last compensatable transaction which can still possibly fail. If it fails compensatable transactions must be reverted.
  3. Retriable transaction - cannot be reverted using compensatable transactions. Must be retried until they finally succeeded.
* Are ACD (not ACID). Lack of isolation leads to dirty reads until the whole saga is finished. Some steps can be taken to mitigate dirty reads through countermeasures - these are design techniques that make the saga more ACID-like. An example is a Semantic Lock - it is an application level lock saying don't read or update this object (i.e. it is a new column or property on a database row or object).

### Communication and Coordination

* Respective services need to communicate with one another to pass commands and queries. Having one service call another synchronously (over HTTP) makes for brittle communication - availability suffers because if one services in the chain goes down or timeouts, entire operation fails.
* The solution to above problem is to use a message broker with at-least-once delivery. This way a saga will work even if participants are down. Another interesting property a message broker should have is ordered delivery - this way we can scale consumers and preserve ordering.
* Return a response from the API to the client before Saga finishes. Let them know how to fetch more data on how the Saga is progressing.
* Coordination options (choreography and orchestration) are described below.
* Most resilient communication pattern is sending messages between services. An issue with this approach is: what if, after we finish the database transaction locally, we cannot contact the messaging queue?
* Approaches below have in common saving the event locally and then having it read from the store and sent to the remote service.

#### Event Sourcing

* Sequence of events is stored in event store and a handler publishes to message broker.
* Example: [Event sourcing database](https://www.eventsource.com)

#### Transactional Outbox pattern

* Insert into an `Outbox` table as part of your transaction. A separate process reads from the database and sends it to the remote service.
* Reading from the database can be done by having a dedicated process utilize transaction log tailing (reading from the transaction log through database-specific APIs). This approach is very database specific. It is most efficient of Transactional Outbox pattern approaches.
* Another way to read the database for new events is to poll the `Outbox` table. This approach is universal because it does not depend on any database-specific APIs, but there is the question of how frequently we want to poll the database to not overtax it too much.
* After the message is sent, the reading process marks it as such.

### Saga influences API design

* Sagas are initiated using a POST request to an API. The question is: when do we send the response back to the caller?
* Several options exists:
  1. Send response after the saga finishes:
    * Keeps the API unchanged, but introduces runtime coupling because the caller must wait on the saga to finish.
    * All saga participants must be online. This reduces availability of the whole system.
  2. Send response immediately after starting the saga:
    * Recommended approach.
    * Higher availability as there is no runtime coupling.
    * API will need to be changed since the response does not specify the result of the operation. 
    * Client will have to poll the system for result or be notified some other way (push notifications?).

### Choreography

* Event driven - coordination logic is distributed among participants.
* Participants publish event and consumers react. Lacks a centralized place where the logic is stored - hard to reason about (in complex sagas), but simple to implement.
* Relies on events:
  * Typically DDD domain events.
  * A change to DDD aggregate. Inidicated something that happened and that is of interest to domain experts.
  * Recommended approach on what to store in an event says to store everything the consumer needs:
    * It's more self-sufficient and does not require callbacks from consuming services.
    * Does introduce design-time coupling as both producing and consuming service need to coordinate, thus providing less stability.
    * Removes the inconsistency risk - if we were to send only an entity identifier in the event there would be a chance the entity changed in the period between the consuming service receiving the event and issuing a callback to the producing service.
  * Another approach on what to store in an event is just the minimal - entity identifier. This requires the consumer to call the oiginating service, thus introducing runtime coupling. There is a risk of inconsistenncy: original data may have changed since the event was published.
* Utilizes a pub-sub principle with potentially many subscribed consumers. The producing service doesn't know which services are listening for events.

### Orchestration

* Saga orchestrator is a dedicated class (e.g. `CreateOrderSaga`) that calls participating services in turn. It uses asynchronous request-response via message broker. Saga orchestrator is a **persisted** object that implements a state machine and invokes the participants. Gets stored in the database waiting for the response to come in. Once a reply comes in it gets retrieved from persistence store and continues executed: calls next participant, updates its own internal state, persists its own internal state and waits for the next reply to come in.
* The saga orchestrator object gets created by the service itself.
* Due to it being a state machine you will probably need an orchestration framework of some sort.
* Utilizes point-to-point communication channel where the message and the channel are owned by the consuming service.

## Cross-cutting concerts

### Health Checks

* Check all dependencies of the project: message broker, database, etc...
* You can also check downstream APIs and services whether they are healthy (check out the package `AspNetCore.HealthChecks.Uris`. However, this approach is risky because it opens you up to cascading failures. It is appropriate only for the most critical services. In such cases, if downstream services are not healthy, it's best to mark your service's readiness status as `Degraded` instead of `Unhealthy`. For most situations, though, it is best not to couple your services health status to downstream services, but rather let the orchestrator handle such situations through monitoring and health checks of those downstream services.
* A good thing to test is whether message broker's topics and queues are reachable. You can even mark the service as `Unhealthy` if the broker is not reachable.

## Documenting distributed systems

### Good engineering practices

* The practices mentioned here are elaborated in the next section. They are here to provide overview in one place.
* Runbook - make it part of your development culture to write stuff to runbook:
  * Write to runbook as you develop code.
  * Regularly verify the runbook with you QA team.
  * Updated with each new release.
* Queues - ops team should maintain good practices:
  * Have a monitoring dashboard to keep an eye on metrics.
  * Periodically take a baseline of the above metrics and record that in the runbook. Use it to detect anomalies.
  * Have a ticketing system so messages in DLQ can be quickly resolved.
* Logging and tracing:
  * Standards should be specified in the runbook - what is logged, how is it logged and how is correlated tracing done.
  * Logging code should be a target of code reviews, to make sure useful stuff gets logged.
  * Document a query (in the runbook) allowing the ops and QA teams to find logs they need in the logging management system. Developers should work with QA on these queries to make sure the queries provide valuable results.

### Runbook

* Maintain a runbook of your system. It should contain information of value to operators, DBAs, developers, testers.
* It is a living document about how your system looks like and how to run in.
* Describe different aspects of your system: databases, application dependency map, how to mitigate known problems, maintain queue metrics.

#### Application databases

* Describe the ERD in a general sense (do not include each column for brevity).
* Describe which tables are expected to grow.
* List long running queries.
* In general, whatever DBA might need so as not to fly blind.

#### Application dependency map

* Create a diagram of externally visible applications. Essentially, these are any web or mobile applications your end users will use. Do not let the work "externally" fool you, it should also include applications used internally by your organization.
* Create a diagram any internal APIs the above applications dependend on. Who depends on whom?
* Describe how these application are hosted, machines, ports, OS, machine names etc...
* How are the applications and APIs configured? Where is the configuration: environment variables or a configuration service? How to change it? How does one machine find another: is there a discovery service or are machine IPs configured statically?
* Describe how the message brokers work, what the topics and queues are named, who is expected to subsribe to each.
* When creating the diagram, show each node:
 * Is it publically accessible?
 * What URL is it reachable on?
 * What port?
 * Which message broker and queue/topic does it use? What does it listen to?
 * Which other node does it depend on?

#### Problems and mitigations

* Problems in production will arise as time goes by. Log them here as they show up and describe how it was solved.
* Is the problem a technical debt? Log it as such.
* Some issues might require a cross-functional research team: network people, DBAs, developers, ops.

#### Queue metrics

* Each broker has different way metrics are measured. Describe the most important metrics relevant to your broker. Reading the metrics correctly will tell you if there is a problem or if you need to do scaling up or down.
* Message delivery - number of messages in each stage of delivery:
 * Number of ready messages: should be close to 0. If it starts to go up, it means there is a bottleneck downstream.
 * Number of unacknowledged messages: should be close to 0. If it starts to go up, it means the consumer it taking a long time to process each message. It might also mean the message is failing to get processed and the consumer is in an exponential backoff.
 * Number of delivered messages: should increase over time. Indicated a total throughput of the system.
* Brokers run on nodes, just like any other service. Node health should be monitored as well:
  * CPU
  * Memory
  * Storage
  * Sockets
* Nodes are in cluster, allowing the system to achieve availability and scalability. Clusted usage should be monitored as well:
  * Connections
  * Channels
  * Queues
  * Consumers
* Monitoring and dashboard tools available: Prometheus, Grafana, Kibana.
* Periodically take a baseline of the above metrics and record that in the runbook. Use it to detect anomalies.

#### Dead Letter Queues

* A strategy is needed for handling messages in DLQ:
  * Alert: at a minimum, an alert is needed whenever anything ends up in DLQ.
  * Locate: a message ending in the DLQ is usually a sign of a problem somewhere else, like a consumer failing to properly process a message. The source of the problem will have to be located by the ops team by figuring out where the message came from - looking at the application dependency map in the runbook is the best way to go about it.
  * Diagnose: why did the service fail to process the message right now? Is something wrong with the message? Has the service been updated recently? Does a service's dependency cause issues (availability, processing error)?
  * Correct
  * Restore: bring the message back into the appropriate queue/topic.
  * Monitor: monitor the handler to make sure the message is processed after the fix.
  * Document: write down the findings in the runbook if they are not noted already.

#### Logging

* When logging an exception, log as much context as you can. Have the operations in mind - they will read the log entry and try to rebound from the error.
* Write logging practices in the runbook and make sure any code relating to logging is code reviewed during PRs.
* Logging proposal:
  * Entry points:
    * Log all inputs (before parsing) - info level
    * If input data contains a lot of details, log the details separately - debug level
    * Log return message so you know what was returned and how long it took - debug level
  * Exit points:
    * Outgoing message (any API call or message sent) - info level, so as to have a basic timing mechanism
    * Optional debug details - debug level
    * Log return message - debug level
  * Storage:
    * Log every time you access the database. Log SQL and parameters - debug level
  * Domain core decisions:
    * Log any decision made in the domain core. No need to log when you have basic CRUD logic, but do log when you have a more complex business logic. Log parameters that went into making the decision - info level.
    * Log exceptions - error level

#### Correlated tracing

* Allows operations to patch together logs from different services.
* Write a query allowing the ops and QA teasm to find logs they need in the logging management system.

## Complex Scenarios

### Sagas

### Compensating transactions

### Maintaining ACID isolation property
