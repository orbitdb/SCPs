## SCP-6: Ipfs-log Entry version 3
###### Status: Draft
###### Authors: [@tabcat](https://github.com/tabcat)
###### Contents:
 - [Background](#background)
 - [Motivation](#motivation)
 - [Entry Version 2](#entry-version-2)
 - [Entry Version 3](#entry-version-3)
 - [Byte Comparison](#byte-comparison)
 - [Summary](#summary)


### Background

[Ipfs-log](https://github.com/orbitdb/ipfs-log) is a core part of [OrbitDB](https://github.com/orbitdb), it is used by the databases to derive its state from an immutable log. This log is built from entries added to the log. Entries are added using [IPFS](https://ipfs.io) and reference previously made entries by their IPFS address.

Besides references to previous entries, entries include other useful data like:
 - entry version
 - which log the entry belongs to
 - ordering information
 - who created the entry
 - a payload field for any JSON value

These pieces together, entry references for log traversal and data used for conflict-resolution/ordering, allow for smooth and stress-free replication of the log over IPFS.

### Motivation

It is in users best interests that Ipfs-log required entry data is minimum. This minimal-ness keeps the log flexible and extensible by only requiring data and fields that are necessary for all users.

Entry version 3 will be simpler, slimmer, and more friendly to extension by writers. It reduces entry block size by 764 bytes compared to version 2, which is around 75% reduction of required entry data. This reduction is done by deduplicating frequently used data and removing unnecessary entry fields. Version 3 also encapsulates writer related fields making custom entry authentication cleaner.

As some of these proposed changes are far reaching and all break backwards compatibility, it may be best to introduce these changes with the [future release of OrbitDB version 1](https://github.com/orbitdb/orbit-db/issues/819).

### Entry version 2

Let's start by taking a look at the structure of an entry in version 2 which is the current version.

```
{
  v: 2,
  id: String,
  clock: {
    time: Number,
    id: <defaults to identity.publicKey>
  },
  key: <identity.publicKey>,
  identity: <identity.toJSON()>,
  sig: String,
  payload: <any JSON.stringify-able data>,
  next: [<CID>],
  refs: [<CID>],
  hash: null
}
```

This is a rough breakdown of what entry version 2 looks like read from ipld. As you can see the `hash` field is null thanks to how hashes and immutable data works. At first glance it seems like we could just remove it without issue, it contains no useful data and I'm not sure why its here to begin with. It's actually one of the fields included with [data to be signed](https://github.com/orbitdb/ipfs-log/blob/b8e4b76247d1bd9b5fa8ad751a62d7f0f3f3f560/src/entry.js#L41-L51) by the writer.

Skipping the version field `v`, which is as barebones as it gets, and moving onto `id`. This field can be set and used by Ipfs-log users to identify and separate entries made to different logs, it is meant to be a string. There is not much to change here as it's just a string set by the user in userland (like in OrbitDB). However in entry version 3 `id` is renamed to `log` because id is used in a couple different places and log seems much better at describing the property.

The `clock` field's value is an object with two properties, `time` and `id`. It is a Lamport clock logical timestamp and is the heaviest ordering metric of entries by default. The `clock.time` field, like `v`, is simply a positive integer, however `clock.id` by default is set to identity.publicKey which already exists in other places in the entry. This is a good chunk of bytes that can be excluded, either by lazily adding `clock.id` when explicitly set by the user or removing it completely (changing `clock` to what `clock.time` is now).

Fields relating to writer authentication are where the bulk of the bytes can be trimmed from the entry. The `key` field contains the writer's publicKey from their identity. This publicKey also exists at `identity.publicKey` so `key` could be removed from the entry.

As mentioned in [The Road to 1.0](https://github.com/orbitdb/orbit-db/issues/819), `separate identities/keys from oplog entry. CID per identity/key. cuts N bytes from each pubsub message and bitswap/ipfs/ild block transfer.`, the `identity` field could be replaced with the CID of the identity. This deduplication should work well as the `identity` field is the same data in every entry for each unique writer. However in situations where writer keys are ephemeral this is less advantaged and may be a good area to leave flexible.

The `sig` field contains a string, hex encoded digital signature of fields: `v`, `clock`, `id`, `payload`, `next`, `refs`, and `hash`. It is used to verify that `identity` is the one that created the entry and that it wrote the entry. Since the data is currently a hex encoded string, it could instead be encoded into base64 strings to pinch a few bytes (~40B per `sig`). Base64 uses [33% less space](https://stackoverflow.com/questions/3183841/base64-vs-hex-for-sending-binary-content-over-the-internet-in-xml-doc) than base16/hex so if encoding/decoding speeds are similar this change may make sense.

Entry version 3 does not propose any changes to fields `payload`, `next`, or `refs`.

### Entry version 3

What's changed?

```
{
  v: 3,
  log: String,
  clock: Number,
  writer: {
    id: <CID>,
    sig: String
  },
  payload: <any JSON.stringify-able data>,
  next: [<CID>],
  refs: [<CID>]
}
```

Obviously `v` has been incremented to the next version; also `hash` has been removed.

Field `id` has been renamed to `log` which is a more descriptive and less common field name.

The old `clock.time` field is now `clock` and `clock.id` is gone. If custom clock ids are a must, `clock` should stay similar to version 2 entries but with `clock.id` lazily added.

The `key` field has been removed.

A new field `writer` has been added. This field encapsulates parts that are mainly connected to entry authentication. The only Ipfs-log required field inside of `writer` is `writer.id` which must be a string and is used for ordering and access control.
The `writer` field is otherwise controlled and set by a writer function which allows more flexibility for entry authentication. A writer function is used by Ipfs-log to get the `writer` field for an entry. It takes an unauthenticated entry and returns an object to be set as the `writer` field.
Using OrbitDB as an example, `writer.id` could be set to the CID of the orbitdb writer identity while leaving the signature field unchanged but moved to `writer.sig`.

The `payload`, `next`, and `refs` fields remain unchanged.

### Byte Comparison

Let's look at how the changes affected the byte size.

The numbers below show the ipld block size of an orbitdb ipfs-log entry encoded using [dag-cbor](https://github.com/ipld/js-ipld-dag-cbor).
The user space fields `payload` and `id`/`log` have been removed from both entries' blocks.

```
entry version 2 size: 995B
entry version 3 size: 249B (203B w/ b64 encoded sig)
size difference: 746B
```

The most effective change made is that the `identity` field in version 2 has been replaced with `writer.id` in the example above, which becomes a CID reference to the identity, the real number of saved bytes in logs using version 3 entries will depend on the number of unique writers.

```
orbitdb identity block size: 538B
real saved bytes: E * 995B - E * 249B - W * 538B
```

### Summary

The goal of this proposal is to simplify log entries to only include the minimum data required for Ipfs-log to operate while promoting extension by users on a need-be basis.

This proposal does this by:
 - simplifying and removing specific entry fields
 - encapsulating writer related data and authentication
 - (possibly) encoding fields in a different base

As many of the changes suggested are breaking, it may be best to introduce them with the release of OrbitDB version 1 if this proposal is accepted.

Any discussion and input from the community on this topic is appreciated. Thank you.
