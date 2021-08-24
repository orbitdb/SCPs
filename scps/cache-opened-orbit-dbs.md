Currently, orbit-db caches opened stores in `this.stores[address]` but it does not seem to use this cache when the same DB is requested again.
This can lead to situations where two parts of the code request the same DB and receive instead two unique open instances of the DB that did not sync correctly with each other.

I would propose to:
* Modify orbit-db so that cached dbs from this.stores are returned if available, **before** trying to reopen a db.
* Use Semaphore (or similar) to avoid concurrency problems in async code.

[This](https://github.com/phillmac/orbit-db-managers/blob/develop/src/DBManager.js) implementation of a cache by [phillmac](https://github.com/phillmac) seems much more sophisticated than what I proposed above.
Perhaps we can discuss more on which approach to use?
