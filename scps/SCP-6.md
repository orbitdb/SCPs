## SCP-6: Ipfs-log Entry version 3
###### Status: Draft
###### Authors:
 - [@tabcat](https://github.com/tabcat)
 - [@haadcode](https://github.com/haadcode)
 
###### Contents:
 - [Background](#background)
 - [Motivation](#motivation)
 - [Entry Version 2](#entry-version-2)
 - [Entry Version 3](#entry-version-3)
 - [Byte Comparison](#byte-comparison)
 - [Summary](#summary)


### Background

[Ipfs-log](https://github.com/orbitdb/ipfs-log) is a core part of [OrbitDB](https://orbitdb.org), it is used by the database to derive its state from an immutable log. This log is built from entries added to the log. Entries are immutable and reference previous entries by [content-address](https://docs.ipfs.io/concepts/content-addressing) (aka content-id or CID); creating causal and traverse-able links.

Besides references to previous entries, entries include other useful data like:
 - entry version
 - which log the entry belongs to
 - ordering information
 - who created the entry
 - a JSON value payload

These pieces together, entry references for log traversal and data used for conflict-resolution/ordering, allow for smooth and stress-free replication of the log using the [IPFS](https://ipfs.io) network.

### Motivation

It is in the user's best interest that ipfs-log required entry data is minimum. This minimal-ness keeps the log flexible and extensible by only requiring data and fields that are necessary for all users.

Entry version 3 will be simpler and slimmer. It reduces entry block size by 820 bytes compared to version 2, which is around 80% reduction of required entry data. This reduction is done by de-duplicating frequently used data and simplifying or removing unnecessary entry fields.

As some of these proposed changes are far reaching and all break backwards compatibility, it may be best to introduce these changes with the [future release of OrbitDB version 1](https://github.com/orbitdb/orbit-db/issues/819).

### Entry version 2

Let's start by taking a look at the structure of an entry in version 2, the version currently used by ipfs-log.

```
{
  hash: null
  v: 2,
  id: String,
  clock: {
    time: Number,
    id: <identity.publicKey>
  },
  key: <identity.publicKey>,
  identity: <identity.toJSON()>,
  sig: String,
  payload: <JSON>,
  next: [<CID>],
  refs: [<CID>]
}
```
This is a rough breakdown of what entry version 2 looks like read from [ipld](https://docs.ipld.io).

- `hash`: This field as shown above is set to null when uploaded to ipld thanks to how immutable data works. This field can be removed without issue, it contains no useful data and shouldn't be included with the [signed](https://github.com/orbitdb/ipfs-log/blob/b8e4b76247d1bd9b5fa8ad751a62d7f0f3f3f560/src/entry.js#L41-L51) and immutable data.

- `v`: This field states the entry's version for anyone reading. As this is a very simple field there is not anything to change here.

- `id`: This field is a string and states what log the entry belongs to. It helps to prevent mixing up entries into the wrong log and replay attacks. There is not much to change here as it's just a string set by the user in userland (like in OrbitDB). However in entry version 3 `id` is renamed to `log` because 'id' is used in a couple different places and 'log' seems much better at describing the property.

- `clock`: This field's value is an object with two properties, `time` and `id`. It is a Lamport clock logical timestamp and is the heaviest ordering metric of entries by default. The `clock.time` field, like `v`, is simply a positive integer, however `clock.id` by default is set to identity.publicKey which already exists in other places in the entry. This is a good chunk of bytes that can be excluded, either by lazily adding `clock.id` when explicitly set by the user or removing it completely (changing `clock` to what `clock.time` is now).

- `key`: This field contains the writer's identity.publicKey. This publicKey also exists at `identity.publicKey` so `key` could be removed from the entry.

- `identity`: This field contains the entry writer's identity which is used by readers of the log to verify and sometimes order entries. As mentioned in [The Road to 1.0](https://github.com/orbitdb/orbit-db/issues/819), "separate identities/keys from oplog entry. CID per identity/key. cuts N bytes from each pubsub message and bitswap/ipfs/ild block transfer", the `identity` field could be replaced with the CID of the identity. This de-duplication should work well as the `identity` field is the same data in every entry for each unique writer. However in situations where writer keys are ephemeral this is less advantaged.

- `sig`: This field contains a string, hex encoded digital signature of fields: `v`, `clock`, `id`, `payload`, `next`, `refs`, and `hash`. It is used to verify that `identity` created the entry. Since the data is currently a hex encoded string, it could instead be encoded into base64 strings to pinch a few bytes (~40B per signature). Base64 uses [33% less space](https://stackoverflow.com/questions/3183841/base64-vs-hex-for-sending-binary-content-over-the-internet-in-xml-doc) than base16/hex.

- `payload`: This field, like `id`, is also userland and is set to any valid JSON data. Nothing to change here.

- `next`: This field is an array of content-addresses. The goal of this field is to reference all entries that have not been reference by another entry; also known as head entries or heads. By doing this we ensure that all but the latest entries in the log are referenced and are not lost. This allows the log to be traversed by anyone starting from the latest entries.

- `refs`: This field is similar to `next` in that they are both arrays of CIDs. The difference between them is that `refs` references entries at an exponentially growing distance away in the log. The length of this array depends on the length of the log. Also it will exclude any CID already existing in `next`. This field was added to speed up loading/replicating the log by exposing distant sections of the log for concurrent fetching and traversal.

### Entry version 3

What's changed?

```
{
  v: 3,
  log: String,
  clock: Number,
  auth: <CID>,
  sig: String,
  payload: <JSON>,
  refs: [<CID>]
}
```

- `hash`: This field has been removed.

- `v`: This field, which states the version, has been incremented to 3.

- `log`: This field replaces/renames `id` from version 2 to be more descriptive of its purpose.

- `clock`: This field becomes the value of `clock.time` from version 2 which is a number representing a logical timestamp.

- `key`: This field has been removed.

- `auth` This field replaces `identity` from version 2. It is a content-address for the identity used to authenticate the entry. In entry version 2 the `identity` field is relatively large and includes `identity.id` which is used for conflict-resolution/ordering. In entry version 3, since `identity` has been replaced with a reference to what was `identity`, the reference could take `identity.id`'s job of conflict-resolution if need be.

- `sig`: This field is now encoded into base-64 instead of hex to reduce entry size.

- `payload`: This field does not change.

- `next`: This field has been removed. Its array values from version 2 have been moved into the `refs` array.

- `refs`: This field's value is now the set union of the values of version 2 `next` and `refs` fields. This fields array is sorted alphabetically to promote determinism.

### Byte Comparison

Let's look at how the changes affected the byte size.

The numbers below show the ipld block size of an orbitdb ipfs-log entry encoded using [dag-cbor](https://github.com/ipld/js-ipld-dag-cbor).
The user space fields `payload` and `id`/`log` have been removed from both entries' blocks.

```
entry version 2 size: 995B
entry version 3 size: 175B
size difference: 820B
```

The most effective change made is that the `identity` field in version 2 has been replaced with `auth` in the example above, which becomes the CID reference of the identity, the real number of saved bytes in logs using version 3 entries will depend on the number of unique writers.

```
orbit-db identity block size: 538B
real saved bytes: E * 995B - E * 175B - W * 538B
```

### Summary

The goal of this proposal is to simplify log entries to only include the minimum data required for ipfs-log to operate.

This proposal does this by:
 - simplifying and removing specific entry fields
 - fields using a more space-efficient base encoding

As many of the changes suggested are breaking, it may be best to introduce them with the release of OrbitDB version 1 if this proposal is accepted.

Any discussion and input from the community on this topic is appreciated. Thank you.
