# DistributedSystems

## 8 fallacies of distributed systems
  * [here](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)
  * The network is reliable - make sure your code is resillient to network failure.
  * Latency is zero;
  * Bandwidth is infinite;
  * The network is secure;
  * Topology doesn't change - single database is an example of this. Be prepared for database failovers and/or distributed databases.
  * There is one administrator
  * Transport cost is zero
  * The network is homogeneous - happens when the system is designed so two different applications need to be deployed at the same time. Take care of versioning to mitigate this.

## Properties we want in a distributed systems:
  * Idempotence
  * Immutability
  * Location independence
  * Versioning

### Idempotence

* Networks are not reliable. If you don't get back a successful reply, you cannot discern between the server not getting the message, an error on the server and server executing successfully but the response somehow getting lost.
* Our system must be prepared to detect a duplicate write and ignore it.
* We do this by generating a client-side generated id and then POSTing (and PUTing, but PUT is idempotent as it is) against that identifier. Guid is a good choice for this. This will act as an alternate key in the database.
* In the course [Fundamentals of Distributed Systems](https://app.pluralsight.com/course-player?clipId=d4145cd4-2477-4a20-ae65-af9476ef06fc) Michael Perry also talks about not returning the Id back to the client. Not sure what that is about. An implication of this is that the client will use the client Id to query the resource and this decision then permeates the interfaces found throughout the application layers.

### Immutability

* Never change the entity, but rather add a new record. Benefits:
  * We have a natural audit log
  * Nothing gets destroyed
  * We preserve what data each node received ("metadata)
  * Promotes eventual consistency
* Immutability changes everything [here](https://vimeo.com/52831373)
* Facilitated by:
  * Snapshot pattern - allows for detecting concurrent writes by comparing the incoming tick count with the last stored tick count. Important thing to note here is the concurrency boundary begins when the data is read. More on my discussion with Michael Perry in the bottom of this article [here](#Discussion-with-Michael-Perry-on-snapshot-pattern-and-concurrency-checks).
  * Tombstone pattern

### Location independence
* Location dependent identifiers like auto-incrementing Ids are an anti-pattern in distributed systems: One record cannot be moved from one database to another because the id might be taken by another record.
* To maintain location independence you should hide or even not use auto-incrementing IDs.
* Location independent identifiers:
  * Alternate Key
  * Natural Key
  * Public Key
  * Hash - a.k.a. Content Addressed Storage, using the content to identify itself. 
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

* One of the fallacies of distributed computing says "Network is homogenous". What this means is that we erroneously might believe all the nodes out there are of the same version (e.g. all instances some specific service are of the same version). This is not true if we want to have things like rolling deploys and backwards compatibility.
* Versioning plays into this by allowing us to maintain contracts across different versions of applications across different locations and different times these are deployed.
* Versioning can be achieved through several approaches:
  * HTTP headers: `Accept=application/some_resource.v1+json`, `Content-Type=application/some_resource.v1+json`
  * Query string: `/api/some_resource?version=v1`, but this kind of defeats the purpose of query string
  * Route: `/api/v1/some_resource`
  * Additive versioning: different versions of the API work with different versions of the payload. Newer versions simply know how to read additional properties in the payload.

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
* Allows the delivery to take place and server to validate the incoming payload. Business invariant cannot be checked, but information can be returned to the client on how top check in later.
* Commands should be done using an asynchronous communication. Client should be returned a 202 Accepted and the processing should be done in the background as this decouples the client and the server. This does make the system more complicated overall as the client now has to check back to see the result of the processing.
* Events should be done asynchronously because the system of record is publishing these to let others know it has finished an action - it is not interested in who picks up the message and therefore decoupling is paramount.

### Publishing Contract Packages

* For services to communicate asynchronously they need to know which events each of them works with - events form an interface.
* To stay on top of the interfaces and any changes that might happen to these interfaces a service should publish events it works with (consumes). This is best done by creating a separate project describing the events and make it part of same solution as the consuming service. Publish the contract package to NuGet (private feed) so other services know what the events should like. Use semantic versioning to convey information whether the new interface introduces a breaking change.
* The section below talks about how to keep track of Dtos, but a different approach is suggested there. Since the above approach is suggested by Michael Perry and the approach below by Roland Guijt, I still have to see if the two are in conflict.

#### API Data Transfer Objects

* You might be tempted to share Dtos used by microservices, but don't do that. This leads to tight coupling between microservices.
* Another option is generating Dtos from OpenApi specification. If you are ok with the generated code, it's a viable approach. Updating the Dtos is now done manually.
* Third option is to simply for each service to have its own copy of Dtos and go with that. Updating the Dtos is now done manually.

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
* An example is creating a new show for an act and then renaming that act. This was important for Perry's search service. When the act is renamed all shows get the new name. When a new show is created the act name is embedded in the event. If these two events get processed the other way round then first all the acts get renamed and only then the new show will be created, but with the old name. He solved the problem by indexing the act name so any show created afterward can read the newest act name.
* I think details on how to implement commutativity are very case-specific.

## Developing with Microservices

tbd
