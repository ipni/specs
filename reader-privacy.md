# Reader Privacy Preservation
​
![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)
​

**Author(s)**:
<!-- keep names alphabetically sorted -->
​
- [Andrew Gillis](https://github.com/gammazero)
- [Ivan Schasny](https://github.com/ischasny)
- [Masih Derkani](https://github.com/masih)
- [Will Scott](https://github.com/willscott)
​

**Maintainer(s)**:
​
- [Andrew Gillis](https://github.com/gammazero)
- [Ivan Schasny](https://github.com/ischasny)
- [Masih Derkani](https://github.com/masih)
- [Will Scott](https://github.com/willscott)
​
* * *
​
**Abstract**

The lookup APIs provided by IPNI nodes are able to observe what data is being accessed by the clients.
This is true regardless of whether the data itself is public or not. Because IPNI nodes continuously
catalogue the content hosted by all the providers, and provide a central lookup API the need for
reader privacy is amplified. This makes IPNI a difficult choice as an alternative routing system in
projects such as IPFS, which use a more decentrailsed routing system that by nature reduces the
possibility of mass query snooping.
​
There is ongoing work on IPFS side to integrate a reader privacy technique, a.k.a, double hashing.
Building on top of the existing approach, this document specifies how a similar technique is applied
to IPNI in order to preserve the reader's privacy while continuing to facilitate low-latency
provider lookup.
​
## Table of Contents

- [Introduction](#introduction)
- [Background](#background)
- [Specification](#specification)
    - [Security](#security)
- [Related Resources](#related-resources)
​
## Introduction
​
IPFS is currently lacking of many privacy protections. One of its main weak points lies in the lack 
of privacy protections for the content routing subsystem. Currently neither readers (clients accessing files) 
nor writers (hosts storing and distributing content) have much privacy with regard to content they publish or 
consume. It is very easy for a content router node or a passive observer to learn which file is requested by 
which client during the routing process, as the potential adversary easily learns about the requested `CID`. 
A curious actor could request the same `CID` and download the associated file to monitor the user’s behavior. 
This is obviously undesirable and has been for some time now a strong request from the community.

The changes described in this specification introduce a IPNI Readres Privacy upgrade. It will prevent 
passive observers from tracking user's actions as described above. It will also be a first step towards
fully private IPNI protocol that will eliminate indexers as centralised observers.  

### Non Goals

* Writer, i.e. content provider or publisher, Privacy, which will be done in a separate specification
* Retrieval Privacy, which is out of scope for the content routing subsystem.
​
## Background
​
Network indexers build their indexes by ingesting chains of Advertisements. Advertisement is a
construct that allows Storage Providers to publish their CIDs in bulk (FIL deals) instead of doing
that individually for each CID. A group of CIDs is represented by a unique ContextID as can be seen
on the diagram below:
​
![Index building flow](resources/readers-privacy-1.png)

## Specification
​
This specification improves the reader privacy by proposing changes to the Step 3, depicted above, where the client supplies the content CID directly in order to lookup its corresponding providers.

In order to protect the reader's privacy the proposal changes the way CID lookup works to the following:

* A client who wants to do a lookup will calculate a hash over the CID (`hash(CID)`) and use it for the
lookup query (hence the name double hashing);
* In response to the hashed find request, the indexer will return a set of encrypted `ProviderRecordKey`s. 
`ProviderRecordKey` will consist of two concatenated hashes - one over `peerID` and the other over `contextID`. 
Each `ProviderRecordKey` will be encrypted with a key derived from the *original* CID value: 
`enc(hash(peerID) || hash(contextID), CID)`, where `hash` is a hash over the value, and `||` is concatenation 
and `enc` is encryption over the value. In order to make sense of that payload, a passive observer would need 
to get hold of the original CID that isn't revealed during the communication round;
* Using the original CID, the client would decrypt `ProviderRecordKey`s and then calculate another hash
over the decrypted `hash(peerID)` part of it. Using that hash for each `ProviderRecordKey` the client would do another lookup 
to get an encrypted `ProviderRecord` in response. `ProviderRecord` will contain information about provider, 
such as it's *peerID*, *multiaddrs*, *supported protocols* and so on. Each `ProviderRecord` will be encrypted 
with a key derived from `hash(peerID)`. In order to make sense of that payload, a passive observer would need to 
get hold of the decrypted `ProviderRecordKey` that isn't revealed during the communication round;
* Using the `hash(peerID)` from `ProviderRecordKey`s, the client would decrypt `ProviderRecord`s and then reach out to the 
provider directly to fetch the desired content. 

By utilising such scheme only a party that knows original CID can decode the protocol,
and that CID is never revealed. 

### Security
​
Security model of the Reader's Privacy proposal boils down to inability to *algorithmically* derive the original CID value for a 
`hash(CID)` that is used for IPNI lookups. Right now indexer advertisments are not encrypted, but authenticated and contain plain CID values in them. 
That is going to change once *Writer Privacy* is implemented. Until then, an attacker could build a map of `hash(CID) -> CID` 
by re-ingesting advertisements chain from each publisher in order to collect all original CIDs which can then be used to decrypt provider records and so on. 
Doing that will require significant resources as it involves crawling the entire network. However, it will eventually be eliminated by *Writer Privacy* upgrade.

Reader Privacy is a first step towards fully private content routing protocol. 

Wider security implications are discussed in the IPFS Reader Privacy specification: TODO link here.
​
## Related Resources
​
TODO: link to corresponding IPFS spec once materialised.
​
* [Double Hashing and Content Routing](https://youtu.be/ZPIDU1-JnVc)
* [Duble Hashing as a way to increase reader privacy](https://youtu.be/VBlx-VvIZqU)
* [Deployment and transition options of Double Hashing](https://youtu.be/m-6_VZ8e1tk)
​
## Copyright
​
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).