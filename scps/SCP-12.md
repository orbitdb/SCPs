# SCP-12: OrbitDB Manifest Format and Class

##### Status: Draft

##### Category: Architecture

##### Authors
- [@tabcat](https://gitub.com/tabcat)

##### Contents
- [Motivation](#Motivation)
- [Manifest Class](#Manifest-Class)
- [Binary Format](#Binary-Format)

### Motivation

The [manifest](https://github.com/orbitdb/orbit-db/blob/main/GUIDE.md#manifest) is a core part of [OrbitDB](https://github.com/orbitdb/orbit-db/). It's used as a shared setup for  databases, and its content address is used as a unique key to represent them.

Creating a class for the manifest and adjusting OrbitDB's API to use them will improve developer experience by more closely representing what is happening.

Many OrbitDB users have asked why they cannot open a store remotely using the store's address. Splitting up fetching the manifest and opening the store will make this more clear.

### Manifest Class

```
class Manifest {
  version: number
  name: string
  type: string
  acl: CID
  meta: any

  constructor({ version, name, type, acl, meta })

  static async create(IPFS, options): Manifest

  static async open(IPFS, Address): Manifest

  toBytes(): Uint8Array

  get address(): Address
}
```

This class will allow for easy creation and interaction with database manifests.

The `create` static method takes an instance of [IPFS](https://github.com/ipfs/js-ipfs) and options, to create a database manifest. It's similar to a combination of the [createDBManifest](https://github.com/orbitdb/orbit-db/blob/5df477ea27f23ad143cae80860767271618ca365/src/db-manifest.js#L5) function; and an OrbitDB instance's [create](https://github.com/orbitdb/orbit-db/blob/5df477ea27f23ad143cae80860767271618ca365/src/OrbitDB.js#L353) method except that it returns a manifest instance.

The `open` static method gets the manifest from IPFS by looking up the CID in the Address; this is currently done [inside OrbitDB's open method](https://github.com/orbitdb/orbit-db/blob/5df477ea27f23ad143cae80860767271618ca365/src/OrbitDB.js#L447). If the manifest can be resolved, it can be read and used to open an OrbitDB Store instance immediately.

### Binary Format

The binary format for the manifest is a [dag-cbor](https://github.com/ipld/specs/blob/master/block-layer/codecs/dag-cbor.md#specification-dag-cbor) encoded map/object of the structure:

###### [CBOR](https://www.rfc-editor.org/rfc/rfc8949.html#section-3.1) types:
```
{
  version: type 0,
  name: type 3,
  type: type 3,
  acl: tag 42,
  [meta: type 0-5 | tag 42]
}
```

###### Decoded to Javascript types:
```
{
  version: number,
  name: string,
  type: string,
  ac: CID,
  [meta: any]
}
```

The square brackets around the `meta` field is to say that it can be optionally included.

---

Some things have been changed from what is currently the manifest.

There is a new field named `version` and the `accessController` field has been replaced with the `ac` field and its type is now a CID instead of a string.

It will be possible to still support reading, and through an option creating old manifests.
