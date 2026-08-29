# Naam: Naming As Advertisement

![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

**Author(s)**:

- [Masih Derkani](https://github.com/masih)

**Maintainer(s)**:

- [Masih Derkani](https://github.com/masih)

* * *

**Abstract**

This document defines the **N**aming **A**s **A**dvertise**m**ent (Naam) protocol: a mechanism by
which [IPNI](IPNI.md) can be used to both publish, and
resolve [IPNS](https://github.com/ipfs/specs/blob/main/ipns/IPNS.md) records. Naam utilises the
extensible IPNI [advertisement metadata](IPNI.md#metadata) to encode IPNS records and form an
advertisement. The resulting advertisements are consumable by indexer nodes, just like any other
advertisement. Further, to resolve IPNI records, Naam utilises the existing
IPNI [multihash lookup API](IPNI.md#get-multihashmultihash) to discover IPNI records.

## Table of Contents

* [Introduction](#introduction)
* [Specification](#specification)
    + [IPNS Record Publication](#ipns-record-publication)
    + [Updating IPNS Records](#updating-ipns-records)
    + [Removing IPNS Records](#removing-ipns-records)
    + [Resolving IPNS Records](#resolving-ipns-records)
* [Limitations](#limitations)
    + [IPNS Record Size](#ipns-record-size)
    + [Record Validity Duration](#record-validity-duration)
* [Security](#security)
    + [Advertisement Verifiability](#advertisement-verifiability)
    + [Reputation](#reputation)
* [Privacy](#privacy)
* [Implementations](#implementations)
* [Related Resources](#related-resources)
* [Copyright](#copyright)

## Introduction

IPNI introduces a set of protocols to discover providers for content-addressable data, along with
the protocols over which the data can be retrieved. Content providers maintain a chain of
advertisements that collectively capture the list of multihashes they provide. IPNI network is
actively being integrated across multiple networks, including IPFS and FileCoin.

As the IPFS network grows, it is desirable to enable alternative content routing mechanisms in order
to provide the best user experience possible; e.g. fast time-to-first-byte, low latency lookups,
etc. IPNI already offers a DHT alternative to discover providers for a given CID. However, DHT in
IPFS provides other functionalities that go beyond provider lookup. One such functionality is IPNS:
InterPlanetary Naming System.

IPNS compliments the immutability of IPFS by providing a way to create cryptographically
verifiable, mutable pointers to content-addressed data. It is part of the core IPFS functionality,
which offers an intuitive way to enable naming service, such as mapping domain names to changing
CIDs. For IPNI to become a complete content routing system and a viable alternative to the existing
IPFS DHT it needs to support IPNS. This is where **N**aming **A**s **A**dvertise**m**ent
(Naam) comes in.

## Specification

Naam is a naming protocol that build on top of the existing IPNI protocol to facilitate a naming
system. It enables publication and resolution of IPNS records using the IPNI advertisements and
indexer node find APIs. Naam utilises the extensible advertisement metadata field to embed IPNS
records and produce advertisements that are consumable by existing indexer nodes without requiring
any change to the existing ingestion pipeline nor lookup API.

This approach offers two key advantages; it:

1) reuses the existing IPNI network to implement a name service, which takes IPNI closer to offering
   a fully featured DHT alternative for the IPFS network without having to run any additional
   services dedicated to resolving IPNS records , and
2) preserves an immutable historical catalogue of all IPNS records published by a peer, which can be
   used for reputation analysis and ultimately holding peers responsible for their participation in
   the network.

Naam naming system involves three main components:

1) **Publisher** - which makes a chain of specifically crafted advertisements available for
   ingestion by the network indexers,
2) **Network Indexer** - which is an IPNI node responsible for ingesting advertisements and exposing
   find APIs, and
3) **Resolver** - which with the aid of network indexers given an IPNS key resolves its
   corresponding record.

The flow of information between the three components is similar to the indexing ecosystem depicted
in IPNI spec: publishers create and publish advertisements -> network indexers ingest them -> and
similar to retrieval clients resolvers use the find API to map a given IPNS key to IPNS record. In
this flow no changes are needed on network indexers. Therefore, the reminder of this document
focuses on interactions of Publishers, Resolvers and the data format exchanged between them. For
more information on how network indexers ingest advertisements and provide lookup APIs, please
see [IPNI](IPNI.md).

### IPNS Record Publication

Naam publishes IPNS records via Publishers, which create IPNI-compliant advertisements that
embed IPNS records and are signed by their identity. The produced advertisements consist of:

* **`PreviousID`** - the optional link to the previous Naam advertisement. `nil` indicates no
  previous advertisements.
* **`Provider`** - the publisher peer ID.
* **`Addresses`** - the array of multiaddrs over which the publisher can be reached.
* **`Entries`** - the link to an `EntryChunk` IPLD node, as specified by IPNI, that contains a
  single entry: the multihash of IPNS record key with `nil` as link to `Next`.
* **`ContextID`** - fixed to UTF-8 bytes encoding of the value `/ipni/naam`.
* **`Metadata`** - the bytes representation of Naam metadata, consisting of:
    * the varint representation of `ipns-record` multicodec code, i.e. `0x0300`, as bytes,
    * followed by the marshalled IPNS record.
* **`IsRm`** - the boolean value specifying weather an IPNS entry is being added/updated,
  i.e. `false`, or removed, i.e. `true`.
* **`Signature`** - the advertisement signature calculated as specified by IPNI specification.

#### `Entries`

`Entires` in a typical advertisement are responsible for capturing the list of multihashes
corresponding to the content hosted by a provider. Once ingested, the indexer nodes facilitate
lookup of providers over find APIs via the same multihashes.

Naam repurposes the same advertisement structure by specifically crafting entries of advertisement
such that the indexer node find API can also be repurposed to lookup IPNS records. Naam forms a
single `EntryChunk` IPLD node, that contains a single multihash calculated as:

* The SHA-256 multihash of IPNS Record Key, i.e. `/ipns/<ipns-key>`.
   * The multihash value is encoded according to the IPNI advertisement encoding. See [IPNI Specification](IPNI.md). 
   * There are no restrictions on the concrete IPNS key format as long as it complies with the [IPNS routing record specification](https://github.com/ipfs/specs/blob/main/ipns/IPNS.md#routing-record).

This approach allows the resolvers to deterministically calculate the multihash used for lookup when
interacting with the indexer node find APIs.

#### Context ID

The field `ContextID` in a regular advertisement is used as a way to uniquely identify the metadata
associated to a list of multihashes. Naam uses a fixed value, `/ipni/naam`, as context ID to assure
that the metadata associated to the published advertisement for the same IPNI key is singular and
uniquely identifiable.

Using a fixed value as context ID also enables network indexers to quickly differentiate Naam
advertisements from a typical content advertisement. This opens up future opportunities for
optimisation when it comes to ingesting Naam advertisements. For example, indexers can pro-actively
process IPNS records and offer bespoke APIs that understand IPNS record expiry/validity.

#### Metadata

`Metadata` is typically used to convey the list of protocols over which the advertised content is
retrievable. Further, this field is designed to be extensible: a valid metadata should start with a
`varint` signifying _some_ protocol ID followed by arbitrary bytes. There are currently three well
known protocol IDs. The protocol ID itself need not to be known by the indexer nodes. In fact,
indexers treat metadata as a black box and carry on processing advertisements. Instead, it is meant
to be a signal to the consumers of `Metadata`, i.e. retrieval clients, as a hint on how to decode
the remaining bytes.

Naam utilises this extensibility to directly encode IPNS records as metadata. The Naam advertisement
metadata consists of:

* `ipns-record` multicodec code `0x0300` as protocol ID,
* followed by marshalled IPNS Record.

The use of metadata to capture IPNS records enables Naam to also utilise the built-in IPNI mechanism
for updating metadata to update IPNS records. Because, context ID nor the multihash in Naam
advertisements ever change for the same IPNS key. For more information,
see [Updating IPNS Records](#updating-ipns-records).

### Updating IPNS Records

IPNI offers a built-in mechanism to efficiently update the metadata associated to a list of
multihashes without having to republish the entries. Indexer nodes uniquely identify the metadata by
the advertisement `Provider`, and `ContextID`.

Both of these values remain the same in Naam advertisements: the former is the same peer ID that
forms part of the IPNS key, i.e. `/ipns/<ipns-key>`, and the latter is fixed. As a result, to update
an IPNS record, all that's needed is to publish a new Naam advertisement that:

* links to the previous advertisement in `PrevousID` link as specified by IPNI,
* uses the same `Provider` and `ContextID` values as before,
* sets `IsRm` to `false`, and
* has the updated IPNS record in Naam Metadata format.

Note that there is no need to specify `Entries` when updating an existing record. It is inferred by
the network indexers since `Provider` and `ContextID` in Naam advertisements never change.

### Removing IPNS Records

IPNS has EOL and TTL features built-in. This means explicit removal of records is not strictly
required. Instead, records eventually expire. Though not strictly necessary, Naam protocol can
utilise IPNI to explicitly remove records just like a traditional DNS nameserver.

The process to remove an IPNS record published via Naam protocol is identical to removal
advertisement specified by IPNI. Similar to a regular advertisement, an IPNS record may be removed
by publishing an advertisement that:

* links to the previous advertisement in `PrevousID` link as specified by IPNI,
* uses the same `Provider` and `ContextID` values as before,
* sets `IsRm` to `true`, and
* sets `Entries` link to `NoEntries` as specified by IPNI.

Once ingested, indexer nodes will eventually remove the Naam advertisement metadata along with the
publisher information.

### Resolving IPNS Records

IPNS record resolution involves finding the IPNS record associated to a given IPNS key, i.e.
`/ipns/<ipns-key>`. As eluded to earlier, Naam uses the find API already provided by the network
indexers to achieve this. To resolve an IPNS record, a resolver:

* calculates the multihash of the given IPNS key as its SHA-256,
* looks up the provider info for that multihash via the IPNI find API,
* decodes the metadata from the provider info as IPNS record,
* validates the record, and
* if valid returns its value.

## Limitations

### IPNS Record Size

The maximum size of IPNS records supported by Naam is 1 KiB, even though IPNS specification suggests
a maximum limit of 10 KiB. The 1 KiB limit currently imposed by network indexers on the maximum size
of advertisement metadata.

Basic tests show that a 1KiB limit is sufficiently large for storing simple IPNS records pointing to
IPFS paths, specially for records that do not need the peer's public key to be embedded into IPNS
records. For example, a Ed25519 peer ID also captures the peer's public key. A marshalled IPNS
record with such key type and empty value constructed
using [`go-ipns`](https://github.com/ipfs/go-ipns) library comes to `257` bytes, which leaves
sufficient space for encoding fairly long IPFS paths as the record value.

Because of the size limitation NAAM Records also limit storing large IDENTITY CIDs as record value.  

### Record Validity Duration

The indexers will maintain information from inactive providers for up to a week. This means IPNS
records published by Naam with EOL of beyond a week should be re-published to remain resolvable.

## Security

### Advertisement Verifiability

Naam advertisements will benefit from all the verifiability features of regular advertisements. They
are signed by the IPNS record publisher and are verified for validity before being ingested by the
network indexers.

### Reputation

Similar to the IPNI indexing protocol, the use of immutable advertisement chain by Naam provides a
verifiable and immutable catalog of IPNS records exposed by a network participant. Access to such
historical data enables verifiable analysis of participant behaviour, and offers an opportunity to
design and develop reputation systems, which ultimately can contribute to the health of the network.
As far as I know, this is something that is not currently offered by the existing implementations of
IPNS.

Even though, the specification of such reputation system is beyond the scope of this document, it is
important to point out such potential. Because, as the size of IPFS network grows so will the need
for reputation systems that enable users to choose from ever increasing "choices" across the
network.

## Privacy

To be revisited when double-hashing work is finalised.

## Implementations

* [`ipni/go-naam`](https://github.com/ipni/go-naam) - Golang implementtion of IPNI Naam protocol.

## Related Resources

* [IPNS: InterPlanetary Naming System](https://github.com/ipfs/specs/blob/main/ipns/IPNS.md)
* [IPNS PubSub Router](https://github.com/ipfs/specs/blob/main/ipns/IPNS_PUBSUB.md)
* [IPNI: InterPlanetary Network Indexer](https://IPNI.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
