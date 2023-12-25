# IPNI Federation Protocol

![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

**Author(s)**:

- [Masih Derkani](https://github.com/masih)

**Maintainer(s)**:

- [Andrew Gillis](https://github.com/gammazero)
- [Ivan Schasny](https://github.com/ischasny)
- [Masih Derkani](https://github.com/masih)
- [Will Scott](https://github.com/willscott)

* * *

**Abstract**

This document provides an overview of the IPNI federation protocol: a protocol by which a set of IPNI instances
collaborate in order to provide a unified eventually consistent view of the index records.

## Introduction

The Inter-Planetary Network Indexer (IPNI) offers a routing system that enables mass advertisement of content addressed
data and lookup performance in order of milliseconds.
This is achieved by a design where a single IPNI instance strives to maintain full network state knowledge.
Many IPNI instances can be instantiated across the globe to offer fast local ingestion and lookup of advertised content.
The velocity and and extent of content discovery across IPNI instances is entirely dependant on their list of providers.
As a result, there is no guarantee that a content consumer would find what they are looking for by searching any one of
the instances.

This document presents a way to get consistent snapshots across a set of collaborating IPNI indexers and return
consistent set of providers and metadata for a given CID.

## Background

The proliferation of distributed and decentralized systems, especially in the domain of planetary-scale networks, has
led to the emergence of multiple challenges, primary among them being data consistency and lookup efficiency. The IPNI
protocol was conceived as a response to these challenges, aiming to provide a robust and efficient routing system for
content-addressed data.

In traditional networks, the data lookup was based on centralized or hierarchical systems, which, although
efficient, were vulnerable to single points of failure and scalability concerns. With the advent of decentralized
architectures, particularly those inspired by concepts like the InterPlanetary File System (IPFS), the paradigm shifted
towards distributed hash tables (DHTs) and other peer-to-peer methodologies.

IPNI evolved as an advanced indexer to overcome the limitations observed in early decentralized routing systems. The
primary design goal was to achieve a balance between data advertisement rates and lookup performance. This led to the
concept of a singular IPNI instance maintaining full network state knowledge, a bold departure from conventional
distributed systems where nodes often possess partial or fragmented knowledge.

However, the realization of the vision of IPNI was not without its challenges. As the adoption grew and instances
sprouted globally, the heterogeneity in content discovery, attributed to the variability in provider lists across
instances, became evident. A singular instance, despite its comprehensive network state, could only discover content at
a pace and breadth defined by its provider list.

Furthermore, the rise of content-centric networks, where data is retrieved based on content rather than its location,
emphasized the importance of a unified and consistent view. The fragmented state of index records across instances
threatened the core utility of IPNI as a reliable and quick lookup mechanism.

The necessity for a federation protocol, therefore, became paramount. It was envisioned not just as a bridge to connect
multiple IPNI instances but as a robust mechanism to ensure that the distributed nature of the system did not compromise
on the uniformity and reliability of content discovery. This document delves deep into this protocol, exploring its
intricacies and mechanisms, laying the foundation for a truly inter-planetary scale network indexer.

## Motivation

The underlying motivation behind enhancing IPNI is manifold. Foremost, guaranteeing an eventually consistent view of
index records across multiple IPNI instances holds the potential for transformative benefits. Such consistency paves the
way for reducing trust in an individual indexer.

## Non-goals

This section clarifies aspects that the IPNI federation protocol will not address in its current iteration.
Here are the non-goals for the IPNI federation protocol:

* **Permissionless Membership**: This specification assumes a permissioned network, where the membership is controlled
  and not open to arbitrary participants.
* **Real-time synchronization**: The protocol strives for eventual consistency, but real-time synchronization across all
  nodes, especially in a globally distributed setup, is not the objective.
* **Network Topology Design**: The IPNI federation protocol does not define or restrict the underlying network topology.
  Whether nodes are structured in a mesh, star, or hierarchical format, the protocol remains agnostic.
* **Strong consistency**: While the IPNI federation protocol aims to ensure data consistency and availability, it does
  not aim to atomically replicate every piece of indexed data across all nodes. For example, it is possible for some
  records to remain partially available on one node until deletions are propagate through the network.

## Terminology

* **Epoch**: An epoch in the context of the IPNI federation protocol refers to a specific time period or iteration of
  the protocol's operation. It can be thought of as a version or generation number that increments each time there's a
  major operation or update, aiding in synchronization efforts across instances.

* **Reconciliation**: Reconciliation is the process of ensuring that two or more IPNI instances have consistent index
  records. It involves comparing the content of instances, identifying discrepancies, and making updates so that all
  participating instances have the same content.

* **Vector Clock (VC)**: Vector Clock is a mechanism used to determine the order of events in a distributed system. In
  IPNI federation protocol, a vector clock helps track the sequence of index updates across multiple instances, thereby
  assisting in maintaining the eventual consistency of the index.

* **Snapshot**: A snapshot refers to a captured state of the index records at a particular point in time consisting of
  the providers list known by the indexer, the latest processed advertisement CID for each provider, an epoch number and
  link to the previous snapshot. IPNI instances can share snapshots with each other to quickly update and reconcile
  their content.

## Specification

The IPNI federation protocol is designed to address the challenges that arise when multiple IPNI instances operate
concurrently across different regions or domains. The fundamental premise of this specification is to provide a
systematic method for these instances to interact, synchronize, and offer a unified view of the index records to
end-users. To achieve this, each node in a federated IPNI network performs the following:

1. Initialization - the process by which a node joins a federated network.
2. Periodic snapshots - the process by which a node periodically captures an immutable snapshot of its providers along
   with associated metadata.
3. Snapshot exchange - the process by which each node propagates their latest snapshot to the rest of the network.
4. Reconciliation - the process by which snapshots received from other nodes is reflected on to bring the network state
   knowledge up-to-date.

Finally, the specification expands on cases where conflict might occur and how a node should resolve it.

### Initialization

At the outset, every participating IPNI instance undergoes an initialization process. This involves setting up its
vector clock to the default or baseline state and establishing secure communication channels with other participating
instances. It's during this phase that instances also share their initial state and any configuration parameters crucial
for the federation protocol. The initialization ensures a foundational layer of consistency, preparing the system for
subsequent operations like snapshot exchange and reconciliation.

### Periodic Snapshot

Recognizing that continuous real-time synchronization could be resource-intensive and inefficient, the IPNI federation
protocol employs a periodic snapshot mechanism. At predetermined intervals, or upon detecting significant changes, each
IPNI instance captures a comprehensive snapshot of its index. This snapshot, which is essentially a compact
representation of the instance's current state, serves as the primary data unit for synchronization among instances. It
ensures that instances can exchange substantial amounts of data without overwhelming the network or causing excessive
data traffic.

The snapshot mechanism also offers a strategic advantage. In the event of any system failures or communication
breakdowns, snapshots can be used to quickly restore or reconcile the state of an instance, minimizing data loss and
ensuring rapid recovery.

A snapshot consists of:

* **Epoch**: a monotonically increasing value that represents the time at which snapshot was formed.
* **Vector Clock**: the monotonically increasing vector clock value that corresponds to the IPNI instance.
* **Provider to Ingest State Map**: a map of provider ID to the latest advertisement CID processed by the IPNI
  instance, along with an optional advertisement height if known by the indexer. The height is provided as a _hint_ to
  other indexers in order to aid the reconciliation and conflict resolution process.

As the IPNI federation protocol continues to evolve, these specification components—initialization and periodic
snapshots—act as the backbone, ensuring that the system remains resilient, efficient, and, most importantly,
consistently synchronized across all instances. Future enhancements may introduce additional layers of optimization or
fault-tolerance, but the essence will remain rooted in these foundational principles.

### Snapshot Exchange

Once snapshots are captured, the next pivotal step is their exchange among IPNI instances. The snapshot exchange process
is underpinned by a two-fold objective: firstly, to disseminate updates and new records rapidly across the network, and
secondly, to identify discrepancies or potential conflicts in the data.

IPNI instances use a peer-to-peer communication model for this purpose. Depending on the topology and preferences, an
instance might push its snapshot to others or allow peers to pull snapshots at their convenience. Given the vast scale
of the network and the potential data volume, snapshots are often compressed or encrypted to ensure efficient
transmission and security.

To further optimize the exchange, instances can use metadata or summary vectors of their snapshots. These summaries
allow instances to quickly ascertain if they need the full snapshot or if they're already up-to-date, thus preventing
unnecessary data transfers.

### Reconciliation

Post snapshot exchange, reconciliation becomes a centerpiece of the protocol. This phase is crucial in addressing any
inconsistencies or conflicts arising from concurrent data operations across multiple instances.

Upon receiving a snapshot from a peer, an IPNI instance uses its vector clock to compare the snapshot's state against
its own. Discrepancies are flagged based on a set of predefined conflict resolution rules. For instance, if two
instances have added different data for the same key, the system might prioritize data from the instance with a more
recent vector clock timestamp.

Reconciliation is not just about resolving conflicts. It's also about merging updates. If an instance discovers newer
data from a snapshot, it updates its index accordingly. Through continuous reconciliation processes, the protocol
ensures that all IPNI instances trend towards eventual consistency.

### Conflict Resolution

In the vast landscape of the Inter-Planetary Network Indexer (IPNI), advertisement chains serve as the core mechanism
for content propagation. As multiple IPNI instances exist across the globe, the potential for discrepancies in the state
of these chains is undeniable. It becomes essential, therefore, to devise strategies that can reconcile these
discrepancies effectively.

#### Case 1: A provider is unknown by another IPNI instance

**Scenario:**  
Instance A is aware of Provider P1 and its advertisement chain, but Instance B is unaware of Provider P1 entirely.

**Resolution Strategy:**  
When instances exchange information, and Instance B identifies a missing provider, it should:

1. Request the complete advertisement chain of Provider P1 from Instance A.
2. Incorporate Provider P1's advertisement chain into its local index.
3. Broadcast this new knowledge to other connected IPNI instances to ensure widespread awareness and data consistency.

#### Case 2: A provider is known by both with a consistent head advertisement

**Scenario:**  
Both Instance A and Instance B are aware of Provider P1 and have the same head (latest) advertisement.

**Resolution Strategy:**  
This represents the ideal state of eventual consistency. In this scenario:

1. Both instances recognize that they have the same version of the advertisement chain for Provider P1.
2. No reconciliation action is required.
3. The instances can continue to propagate other advertisement chains or updates, knowing that, for Provider P1, they
   are in sync.

#### Case 3: A provider is known by both but with different head advertisements

**Scenario:**  
Instance A and Instance B both know Provider P1, but they have different latest advertisements (heads) for P1's chain.

**Resolution Strategy:**  
This represents a divergence in the advertisement chain, and careful reconciliation is required:

1. Instance A and B determine which advertisement is the later one in the chain of advertisements provided by P1.
2. They converge on the advertisement has been published the latest, i.e. has the shorter distance to P1's current
   advertisement head.

While traversing advertisement chains, instances should prefer pulling ad chain from eachother's mirror instead of going
directly to the provider where possible.

### Failure Tolerance

In any distributed system, handling failures is of paramount importance. The IPNI federation protocol is designed with a
robust failure tolerance mechanism to ensure that the system remains functional even in the face of node failures,
communication breakdowns, or data corruption.

If an IPNI instance becomes temporarily unavailable, upon rejoining the network it should reach out to each of its peers
and perform a reconciliation cycle using other peer's federation state snapshot. The instance may merge the aggretage of
other peers federation state snapshot with its own after having verified that its list of providers are still
responsive.

Furthermore, the protocol has in-built monitoring and alerting systems. If an instance repeatedly fails to exchange
snapshots or if there's a significant drift in its data, alerts can be raised for manual intervention or deeper
diagnostics.

In essence, the IPNI federation protocol's specification, through its various components, aims to build a system that is
both efficient in its operation and resilient in its design. Each phase, from initialization to failure tolerance, is
meticulously crafted to ensure that the global network of IPNI instances remains coherent, consistent, and always
available for its users.

## Federation State API

Each IPNI indexer participating in the federation must expose a set of APIs to facilitate snapshot exchange and conflict
resolution. This document does not impose any limits on the transport protocol over which these APIs may be exposed. Two
of the most common such protocols are vanilla HTTP and HTTP over libp2p.

### GET `/ipni/v1/fed/head`

This endpoint exposes information about the most recent change in the federation state of an indexer node for the ingest
topic an IPNI indexer is ingesting from.

Note that this endpoint may get updated on periodic snapshots and care must be taken to assure caching does not
interfere with the freshness of information exposed by this endpoint. See [Caching](#caching) for further information.

#### Response

The response from the `/ipni/v1/fed/head` endpoint includes the following fields:

- **`head`** (required): The CID of the latest snapshot taken by the indexer.
- **`epoch`** (required): The epoch to which indexer's federation state snapshot is associated.
- **`topic`** (optional): The topic name on which the indexer is subscribed to ingest. If not specified, the default
  value
  of `/indexer/ingest/mainnet` is assumed.
- **`pubkey`** (required): The serialized public key of the indexer in protobuf format using the libp2p standard. The
  public key can be marshalled using
  the [`crypto.MarshalPublicKey`](https://pkg.go.dev/github.com/libp2p/go-libp2p-core/crypto#MarshalPublicKey) function.
- **`sig`** (required): The signature associated with the `head` CID, obtained by concatenating the bytes of the `head`
  CID and the UTF-8 bytes of the `topic` (if present). The signature is verified against the `pubkey`.

Please note that the response provides the necessary information to validate the authenticity of the snapshot CID
and verify its integrity using the indexer's public key and the associated signature.

The following snippet represents the IPLD schema of the signed federation state head:

```ipldsch
type FederationStateHead  struct {
    head   Link
    topic  optional String
    pubkey Bytes
    sig    Bytes
}
```

#### Example

The following represents an abbreviated example of head federation state response in `DAG-JSON` encoding:

```json
{
  "head": {
    "/": "baguqeerazx6qhbdzckrnuwvl3reqnrpysz3ry5qtdxzpo23losoeiywij65a"
  },
  "topic": "/indexer/ingest/devnet",
  "pubkey": {
    "/": {
      "bytes": "CAASpgIwggEiMA0G..."
    }
  },
  "sig": {
    "/": {
      "bytes": "LrZiicKdqDqkG2UFR..."
    }
  }
}
```

### GET `/ipni/v1/fed/{CID}`

This endpoint retrieves the content associated with a federation state snapshot CID.
All links encountered during the traversal of the federation state snapshot CID are served by this endpoint.
The `CID` specified in the URL parameter must match the response body, as the data returned by this endpoint is
immutable.

To ensure proper caching and immutability, implementers must include the following response header:

```
Cache-Control: public, max-age=29030400, immutable
```

By setting this response header, it instructs client-side caches and intermediate proxies to store the response for a
long duration (`max-age=29030400`), as well as treat it as immutable. This helps improve performance and ensures that
the same content is served consistently for the specified `CID`.

### Response

The response from the `/ipni/v1/fed/{CID}` endpoint returns the data associated to a federation state snapshot CID.

It contains the following fields:

- **`epoch`** (required): The epoch to which indexer's federation state snapshot is associated.
- **`vc`** (required): The monotonically increasing vector clock.
- **`providers`** (required): The map of provider peer ID to its latest ingest state as known by the indexer. The ingest
  state consists of the latest processed advertisement CID and its optional corresponding height if known by the
  indexer.
- **`previous`** (required): The link to previous federation state snapshot.

The following snippet represents the IPLD schema of a federation state snapshot:

```ipldsch
type FederationStateSnapshot struct {
    epoch     uint64
    vc        uint64
    providers {String:IngestState}
    previous optional Link
}

type IngestState struct {
   lastAdvertisement Link
   height  optional  uint64
}
```

#### Example

The following represents and abbreviated federation state snapshot encoded in `DAG-JSON`:

```json
{
  "epoch": 123456,
  "vc": 42,
  "providers": {
    "123bAff1edF1sh...": {
      "lastAdvertisement": {
        "/": "baguqeeraog3uqu7au2mctrfslzvua5lzl3wyoquqhbyoig6hnputypej2uvq"
      },
      "height": 1413
    },
    "123hUngry10bstER...": {
      "lastAdvertisement": {
        "/": "baguqeeranqfhpdacbe7ls44ndcgeis6yufvhk4zntydlqfaplzz6mdyuu6gq"
      }
    },
    "previous": {
      "/": "baguqeerazx6qhbdzckrnuwvl3reqnrpysz3ry5qtdxzpo23losoeiywij65a"
    }
  }
}
```

## Caching

Implementers should include explicit Cache-Control headers to manage caching behavior. This is beneficial for the
following reasons:

- **Efficient Discovery**: Indexer nodes only need to be aware of the latest head federation state. Since federation
  state snapshots
  are chained, previous ads will be automatically discovered. By setting appropriate cache parameters on the response,
  indexers can determine how often they need to contact providers to discover a new head. This approach optimizes
  traffic between the provider and indexer nodes, improving overall efficiency.

- **Frequency of Head Changes**: The head federation state of a typical indexer may not change frequently. By setting
  cache parameters, providers can indicate the appropriate caching behavior to indexers. This helps indexers decide how
  often they should request updates for the head federation state. Setting optimal cache parameters can result in more
  efficient utilization of network resources.

To disable HTTP caching and ensure that participating peers always receive the latest response, providers should set the
following
Cache-Control header:

```
Cache-Control: no-cache, no-store, must-revalidate
```

Alternatively, if providers want to allow accepting a cached response and revalidating it in the background, they should
use an appropriate `Cache-Control` header with a `max-age` value that reflects the frequency at which the provider
generates new advertisements. Additionally, the `stale-while-revalidate` directive can be used to specify a period
during which a stale cached response can still be served while a revalidation request is sent in the background.

For more information on caching headers, you can refer to [RFC5861](https://datatracker.ietf.org/doc/html/rfc5861).

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
