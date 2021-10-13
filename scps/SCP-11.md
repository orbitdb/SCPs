# SCP-11: OrbitDB Address Formats and Class

##### Status: Draft

##### Category: Architecture

##### Authors
- [@tabcat](https://gitub.com/tabcat)
- [@csdummi](https://github.com/csdummi)

##### Contents
- [Motivation](#Motivation)
- [String Format](#String-Format)
- [Binary Format](#Binary-Format)
- [Address Class](#Address-Class)

### Motivation

With the current implementation of [OrbitDB](https://github.com/orbitdb/orbit-db)'s Address there are many  supported string formats:

- `/orbitdb/zdpuAmm4Lxe7GHePm1JChphanurFC6PEZ93MTHR1oz5rh6Jcg/example`
- `/orbitdb/zdpuAmm4Lxe7GHePm1JChphanurFC6PEZ93MTHR1oz5rh6Jcg`
- `zdpuAmm4Lxe7GHePm1JChphanurFC6PEZ93MTHR1oz5rh6Jcg/example`
- `zdpuAmm4Lxe7GHePm1JChphanurFC6PEZ93MTHR1oz5rh6Jcg`

They all center around the manifest [CID](https://github.com/multiformats/cid), the only necessary part of the address.

The purpose of this document is to provide a reference specification for the String and Binary Formats of the [OrbitDB Addresses](https://github.com/orbitdb/orbit-db/blob/main/GUIDE.md#address) and to present a refactored `OrbitDBAddress` class to be used to load OrbitDB Manifests and Stores.

### String Format

We redifine the string format:

`/orbitdb/<root>/<path>` &rarr; `/orbitdb/<cid>`

The `path` has been removed, this part should match the database name found in the manifest. We change how the stringified hash of the manifest is refered to from `root` to `cid`.
This format fixes some issues which are mainly related with the `path` not able to be verified as matching the database name until the manifest is fetched.

### Binary Format

We define a binary format for the address:

`<cid>`

The format is simply the byte array of the manifest's decoded CID. This is useful for situations where saving space is valuable. There are two specific cases we see that can benefit from this:

  - users encoding addresses into qr codes
  - replacing their stringified versions used in log entries


Building on this format we can support the [ENS contenthash](https://eips.ethereum.org/EIPS/eip-1577#specification) format:

`<orbitdb protocode><cid>`

This requires a [multicodec](https://github.com/multiformats/multicodec) prefix added to the decoded CID which we would get by reserving a space in the [multicodec table](https://github.com/multiformats/multicodec/blob/master/table.csv) for orbitdb.

### Address Class

```
class Address {
  cid: CID

  constructor({ cid: CID })

  static asAddress(Address | { cid: CID }, [force: boolean]): Address | null

  static fromBytes(Uint8Array): Address

  static fromString(string): Address

  toBytes(): Uint8Array

  toString([MultibaseEncoder]): string

  equals(Address): boolean
}
```

Above is some psuedo typescript of the refactored Address class interface.

Most of the changes made have to do with supporting the two serialization formats, string and buffer/bytes. The term bytes is used here to distance from the NodeJS Buffer data type since we use their superclass, Uint8Arrays.

The `join` method has been removed, since paths have been removed from the address there wasn't much reason to keep it. It's also been a problem for compatibilty with windows.

Methods `parse` and `isValid` have been replaced with `fromString` and `asAddress` respectively. The `toString` method now allows an optional parameter for a multibase encoder for the cid hash.

The new `asAddress` method can be used in place of `isValid` by supplying a truthy value as a second parameter, here referenced as `force`. This will cause an error to be thrown if the first parameter isn't an address. Without that second parameter the method will return an Address or null. This method is very similar to `CID.asCID`.

Other new methods include `toBytes`, `fromBytes`, and `equals`. The bytes methods are for working with the buffer format, `cid.bytes`. The `equals` method can be used to compare an address with another one, it will return true if the addresses are the same. It uses the `cid.equals` method to compare the cid of the two addresses.
