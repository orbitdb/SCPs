# SCP-001 - Replicator Refactoring

> Use Case: I have a database that has been replicated locally. I want to get the current state of the db as
fast as possible when opening the db (in order to return the first query as fast as possible).

As of right now, the store replicator uses the `next` field in a log entry to replicate,
whereas it could use the new `refs` field, as loading now does.

This is by and large the most effective improvement we can make, and perhaps the most
often requested and discussed in the community.

Relevant bits of code:
- https://github.com/orbitdb/ipfs-log/blob/master/src/log.js#L270-L289, AKA the "right way" to do it.
- https://github.com/orbitdb/orbit-db-store/blob/master/src/Replicator.js#L128-L154, the `orbit-db-store` Replicator still using the `nexts` field

There are other possible ways to address the initial query and loading performance that we may want to take up on. Discuss!
