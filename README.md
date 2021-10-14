# DistributedSystems

## 8 fallacies of distributed systems
  * [here](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)

## Properties we want in a distributed systems:
  * Idempotence
  * Immutability
  * Location independence
  * Versioning

### Idempotence

* Networks are not reliable. If you don't get back a successful reply, you cannot discern between the server not getting the message, an error on the server and server executing successfully but the response somehow getting lost.
* Our system must be prepared to detect a duplicate write and ignore it.
* We do this by generating a client-side generated id and then POSTing (and PUTing, but PUT is idempotent as it is) against that identifier. Guid is a good choice for this. This will act as an alternate key in the database.
* In the course [Fundamentals of Distributed Systems]() Michael Perry also talks about not returning the Id back to the client. Not sure what that is about. An implication of this is that the client will use the client Id to query the resource and this decision then permeates the interfaces found throughtout the application layers.

### Immutability

* Never change the entity, but rather add a new record. Benefits:
  * We have a natural audit log
  * Nothing gets destroyed
  * We preserve what data each node received ("metadata)
  * Promotes eventual consistency
* Immutability changes everything [here](https://vimeo.com/52831373)
* Facilitated by:
  * Snapshot pattern - allows for detecting concurrent writes and prevents overwrites.
  * Tombstone pattern

### Location indepence
* Location dependent identifiers like auto-incrementing Ids are an anti-pattern in distributed systems: One record cannot be moved from one database to another because the id might be taken by another record.
* Location independent identifiers:
  * Alternate Key
  * Natural Key
  * Public Key
  * Hash - a.k.a. Content Addressed Storage, using the content to identify itself. Advantages:
    * Naturally immutable: if the hash is different, it's a different object.
    * Naturally verifiable: if the hash you calculate does not match the hash of the object you just received, it has been tampered with.
    * Naturally idempotent: if the hash of the object you hold is the same as the hash of the object you want to replace, no need to replace it.
    * Solves cache invalidation: if the cache contains an object with a given hash, you know it's up to date. More [here](https://app.pluralsight.com/course-player?clipId=26c73001-f898-4ea0-a6aa-2cc41fe14185).

### Open points
* Why are we hiding Id from clients?
* Benefits of immutable data?
