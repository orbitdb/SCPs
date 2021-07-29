# SCP-7: Read Access Control
**Status: Draft**

**Authors:**
- [@CSDUMMI](/csdummi)

**Contents:**
- [Read Access](#read-access-in-orbitdb-stores)
- [Read AccessControllers](#read-accesscontrollers)
- [Protocol for Requesting and granting access to an entry](#Protocol-for-Requesting-and-granting-access-to-an-entry)
- [Modifications to the ipfs-log](#modifications-to-the-ipfs-log)

## Read Access in OrbitDB Stores.
Currently OrbitDB's AccessControllers allow 
for the control of `write` access to a data store, but 
not the control of read access.

This is first of all due to the fact, that IPFS Objects
can be read by anyone,
which means, that encryption is the scheme to 
use to control read access while still using IPFS.

In this SCP, I propose the development of an Access control
mechanic, that will fulfill these criteria:
1. All the data in the store is stored in an encrypted form
2. Every entry is individually encrypted and decrypted
3. Read Access can be granted for each entry individually and to each identity individually.

## Read Access Controllers
Let's first define, how read access may be 
granted from the point of view of an OrbitDB user.

I propose adding a `canRead(entry, identity)` method to the `AccessController` class,
which is passed the plaintext entry and the identity of the machine, that is requesting
access to the entry.

If the method returns true, access may be granted to the identity for the entry.
**But only for this entry and only to this identity!**

With such an API, it is comparatively simple to implement even very complicated
read permissions in JS. 

But any user has to remember, that once somebody is given read access to some data,
this access may not be revoked and that they could possibly grant access to the data
to other identities or the public in whichever way they might choose.

For this reason I decided against using a single key to encrypt the entire
store, since that would not make it possible to differentiate between
entries and increases the risk of leaking data.

## Protocol for requesting and granting access to an entry.
In order to securely implement this API, a simple PubSub based protocol may be implemented.

1. It starts by the requesting node publishing a signed request for an entry into a pubsub channel. 
2. Then some node, that has access to the entry in plaintext, checks via `canRead` whether or not the request should be granted.
3. If the request was granted the data is encrypted using a one-time key (that was shared using the Diffie-Hellman Key Exchange) and the CID of the encrypted content published on PubSub.


After this, both the requesting as well as the providing node have read access to the data, but nobody else
observing the PubSub channel has any information, except that the requesting node requested some data and the providing node provided said data. (Which might be a leak of information in some use cases, especially concerning meta data).

### Diffie Hellman Key Exchange.
Sources: [Computerphile video on this](https://invidious.048596.xyz/watch?v=NmM9HA2MQGI) and the [Wikipedia Page](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange).

The Diffie-Hellman key exchange is one of the first public key
cryptography systems proposed and has the goal of exchanging a 
private key over a public medium, without leaking the private key to
anyone but the intended participants of the exchange.
It was implemented and tested for a long time (being proposed in 1976, one year earlier than RSA)
and is very computationally expensive to break, even for supercomputers.

After a key has been shared with Diffie-Hellman, it can be used to 
seed a PRG or directly be used as a key for AES, Chacha or some other
symmetric cipher.

## Verification that correct data was received
In this scheme the `ipfs-log` no longer stores the data of an 
entry directly, instead it stores the CID generated from the plaintext
of the entry.
                                       
This means that no trust between the node granting access and the node
receiving access has to be established, since the node receiving data
can decrypt it with the generated key and verify the CID with the one
in the entry.
																			 
## Modifications to the `ipfs-log`
Because the encryption of some data `D` and the data `D` are
different, `CID(D)` and `CID(E(D))` are not the same.
Thus, the `ipfs-log` has to be modified. 
Instead of storing an encrypted Object directly in the graph,
each node in the graph stores the CID of the Object in plaintext.
(Which is generated, but the object itself is not added to IPFS)

Then the encrypted objects refer to this object using links.
And after somebody receives a version of the 
object, they can decrypt (Through the above described protocol) and
verify it with the CID stored in the `ipfs-log` node.

## Conclusion
It is theoretically possible to create
an OrbitDB Store, that has some read access control,
that fulfills all the criteria named above.

But it is complicated and requires reworking the 
`ipfs-log` and creating a private store for the Diffie-Hellman one time keys.

Implementing this would thus require:
1. Modifying `ipfs-log` to allow for objects, that only store the CIDs of the plaintext entry.
2. Modifiying the key store to store the lot of Diffie-Hellman Keys or some key, that is used to encrypt all my entries, but is never shared with anybody. (To reduce the amount of keys, that have to be managed.)

																			 
# Going forward
I would propose to first create an implementation 
as a custom store to verify and test the system
and then to implement in the `Store` type itself.

I would like it if it would be as easy as passing 
an option to the `orbitdb.create` function to create
a read protected, encrypted database.
											
