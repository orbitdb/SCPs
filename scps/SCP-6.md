## SCP-6: Ipfs-log Entry version 3
###### Status: Draft
###### Authors: [@tabcat](https://github.com/tabcat)

### Background

[Ipfs-log](https://github.com/orbitdb/ipfs-log) is a core part of OrbitDB, it is used by the databases to derive its state from an immutable log. This log is built from entries added to the log. Entries are added using [IPFS](https://ipfs.io) and reference previously made entries by their IPFS address.

Besides references to previous entries, entries include other useful data like:
 - entry version
 - which log the entry belongs to
 - ordering information
 - who created the entry
 - a payload field for any JSON value

These pieces together, entry references for log traversal and entry data used for ordering, allow for smooth and stress-free replication of the log over IPFS.

### Motivation

Logs grow in size as entries are appended, this means they take up more space in storage and on the network. It is in users best interests that ipfs-log required entry data is minimum and entries are simple and extensible.

Entry version 3 will be 764 bytes smaller than version 2 by deduplicating frequently used data like identities and reevaluating entry fields. It will also prune and simplify entry fields.

As some of these proposed changes are far reaching and all break backwards compatibility, it may be best to introduce these changes with the [possible future release of OrbitDB version 1](https://github.com/orbitdb/orbit-db/issues/819).

### Entry version 2

Let's start by taking a look at the structure of an entry in version 2 which is the current version.

```
{
  v: 2, // version number
  id: {String},
  clock: { // lamport clock info
    time: {Number},
    id: <defaults to identity.publicKey>
  },
  key: <identity.publicKey>,
  identity: <identity.toJSON()>,
  sig: {String},
  payload: <any JSON.stringify-able data>,
  next: {[String]},
  refs: {[String]},
  hash: null
}
```
This is a rough breakdown of what entry version 2 looks like read from ipfs dag using [dab-cbor](https://www.npmjs.com/package/ipld-dag-cbor). As you can see the `hash` field is null thanks to how hashes and immutable data works. At first glance it seems like we could just remove it without issue, it contains no useful data and I'm not sure why its here to begin with. It's actually included with [data to be signed](https://github.com/orbitdb/ipfs-log/blob/b8e4b76247d1bd9b5fa8ad751a62d7f0f3f3f560/src/entry.js#L41-L51) by the writer.

Skipping field `v`, which is as barebones as it gets, and moving onto `id`. This field can be set and used by ipfs-log users to identify and separate entries made to different logs, it is meant to be a string. There is not much to change here as it's just a string set by the user in userland (like OrbitDB). In entry version 3 `id` is renamed to `log` because id is used in a couple different places and log seems better at describing the property.

The `clock` field's value is an object with two properties, `time` and `id`. It is a Lamport clock logical timestamp and is the heaviest ordering metric by default. The `clock.time` field, like `v`, is simply a positive integer, however `clock.id` by default is set to identity.publicKey which already exists in other places in the entry. This often a good chunk of bytes that could be excluded, either by lazily adding `clock.id` or removing it completely (changing `clock` to what `clock.time` is now).

Fields relating to writer authentication are where the bulk of the bytes can be trimmed from the entry. The `key` field contains the writer's publicKey from their identity. This publicKey also exists at `identity.publicKey` so `key` could be removed from the entry.

As mentioned in [The Road to 1.0](https://github.com/orbitdb/orbit-db/issues/819), `separate identities/keys from oplog entry. CID per identity/key. cuts N bytes from each pubsub message and bitswap/ipfs/ild block transfer.`, the `identity` field could be replaced with the CID of the identity. This deduplication should work well as the `identity` field is the same data in every entry for each writer. However in situations where writer keys are ephemeral this would increase bytes transferred; if entries only needed to be verified once then stored bytes would still be lower.

The `sig` field contains a string, hex encoded digital signature of fields: `v`, `clock`, `id`, `payload`, `next`, `refs`, and `hash`. It is used to verify that `identity` is the one that created the entry. As with hex encoded strings they can instead be encoded into base64 strings to pinch a few bytes (~40B per sig). Base64 is [33% more space efficient](https://stackoverflow.com/questions/3183841/base64-vs-hex-for-sending-binary-content-over-the-internet-in-xml-doc) than base16/hex. If encoding/decoding speeds are similar this is a good change.

Entry version 3 does not propose any changes to fields `v`, `payload`, `next`, or `refs`.

### Entry version 3

What's changed?

```
{
  v: 3,
  log: {String},
  clock: {Number},
  writer: {
    identity: <CID>,
    sig: {String}
  },
  payload: <any JSON.stringify-able data>,
  next: {[String]},
  refs: {[String]}
}
```

Obviously `v` has been incremented to the next version and `hash` has been removed.

Field `id` has been renamed to `log` which is more descriptive and less common.

The old `clock.time` field is now `clock` and `clock.id` is gone. Now `writer.identity`, which is a CID, is used as the old `clock.id`. If custom clock ids are a must they should be lazily added and `clock` will look similar to entry version 2.

The `key` field has been removed.

A new `writer` field  encapsulates the `identity` and `sig` fields. Ipfs-log requires `writer` to contain one field `identity` which must be a CID as a string. This CID could be used like identity.id currently is for access control and ordering. The `writer` field is otherwise controlled and set by a writer class. This allows for more flexibility in entry authentication. Roughly the writer class would be similar to an identity but take the unauthenticated entry and return the entries writer field (which includes the authentication whatever it may be). You can see the `writer.sig` field above, this would have been put there by the writer class not Ipfs-log.

The `payload`, `next`, and `refs` fields remain unchanged.

### Byte change

Let's look at the byte comparison.
> the `payload` has been removed from both entries

#### entry version 2 size: 1112B or 1.1kB
#### entry version 3 size: 348B (without base64 encoded sig, ~300B w/ b64)
#### size difference: 764B


