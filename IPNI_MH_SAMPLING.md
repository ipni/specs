# IPNI Multihash Sampling

![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square)

**Author(s)**:

- [Masih Derkani](https://github.com/masih)

**Maintainer(s)**:

- [Masih Derkani](https://github.com/masih)
- [Will Scott](https://github.com/willscott)

* * *

**Abstract**

This document outlines the specification for the **IPNI Multihash Sampling**, a complimentary component of the IPNI
ecosystem designed to facilitate efficient and scalable sampling of multihash records. The API provides a standardized
interface for querying multihash records advertised by a provider for a given context ID, enabling use cases such as
retrieval checking, data integrity verification, and decentralized reputation systems.

## Table of Contents

- [Introduction](#introduction)
- [Motivation](#motivation)
- [Background](#background)
- [Specification](#specification)
    - [Rationale](#rationale)
    - [Determinism and Consistency](#determinism-and-consistency)
    - [API](#api)
- [Notes to Implementers](#notes-to-implementers)
    - [Security Considerations](#security-considerations)
    - [Storage Complexity](#storage-complexity)
    - [Performance Optimizations](#performance-optimizations)
        - [Indexing](#indexing)
        - [Caching](#caching)
        - [Concurrent Ingestion](#concurrent-ingestion)
- [Related Resources](#related-resources)

## Introduction

The **IPNI Multihash Sampling** is designed to address the growing need for efficient data sampling in decentralized
systems. By leveraging IPNI's structured multihash advertising framework, this API enables users to perform targeted
queries on large datasets without incurring excessive computational overhead. The API is optimized for performance,
scalability, and interoperability with existing IPNI tools and protocols.

## Motivation

The development of the **IPNI Multihash Sampling** is driven by several key requirements:

1. **Efficient Data Sampling**: Traditional approaches to data sampling often require exhaustive scans of entire
   datasets, leading to high resource consumption and latency. This API provides a more efficient alternative by
   enabling direct queries on specific provider for a specific context ID.

2. **Decentralized Data Integrity**: In decentralized systems, verifying the integrity of distributed data is critical.
   This API supports retrieval checking by allowing users to sample and verify subsets of data without requiring access
   to the entire dataset using a deterministic sampling algorithm. This is particularly useful for assessing data
   quality and reliability in decentralized checker networks where checkers are incentivized to provide accurate results
   with no or loose centralized coordination.

3. **Scalability**: The API is designed to scale with the growth of decentralized content, ensuring that sampling
   operations remain efficient even as the size and distribution of content increases. This is achieved through a random
   deterministic sampling algorithm that can be applied to large datasets without sacrificing performance. As a result,
   the API facilitates a probabilistic approach to retrieval checking, allowing both users and indexer operators to tune
   the tradeoff between accuracy and resource consumption.

4. **Interoperability**: The API is compatible with existing IPNI tools and protocols, enabling seamless integration
   into broader workflows and ecosystems, such as Filecoin data identification (i.e. Piece CID), IPFS content addressing
   and major implementations of both IPFS and Filecoin.

## Background

InterPlanetary Network Indexer (IPNI) facilitates the discovery of retrieval providers for given content IDs and the
protocols over which it can be retrieved. This API extends IPNI's utility by enabling granular sampling of provider
advertisements to efficiently check data retrievability. This aligns with broader efforts to link on-chain storage
activity with retrieval performance, as highlighted in
the [Retrieval Checking Requirements FRC](https://github.com/filecoin-project/FIPs/pull/1089). Additionally, it can be
used as a way to score federated indexers in a decentralized reputation system, to better select the most reliable ones
for content routing as part of [ambient content router discovery](https://github.com/ipfs/specs/pull/342).

## Specification

This specification outlines the key components of the **IPNI Multihash Sampling** API, including the rationale behind
its design, the deterministic sampling algorithms employed, and the API endpoints for sampling multihashes.

### Rationale

Two key rationale for the **IPNI Multihash Sampling** API are:

1. **Efficient Data Sampling**: Traditional data sampling methods often require exhaustive scans of entire datasets,
   leading to high resource consumption and latency. The API provides a more efficient alternative by enabling direct
   queries on specific provider for a specific context ID.
2. **Tunable Accuracy**: The API allows users to tune the tradeoff between accuracy and resource consumption by
   specifying the number of samples desired. This feature is particularly useful for probabilistic retrieval checking,
   where users can adjust the sampling size based on their specific requirements, and the reputation of the provider.
3. **Deterministic Consistency**: The API ensures deterministic consistency and reproducibility across all
   implementations, enabling identical sampling results for the same input parameters across federated indexers. This
   feature is crucial for a smooth and fast operation of decentralized checkers that may be geographically distributed
   and communicate with different indexer instances to check retrievability.

### Determinism and Consistency

The **IPNI Multihash Sampling** uses deterministic algorithms with a uniform seed to produce consistent, repeatable
sampling sequences. This reduces sampling bias and ensures the same results for repeated queries. Key components
include:

* **Deterministic Sampling Algorithm**: The API employs a predefined algorithm that yields identical sampling results
  for the same input parameters across a node, even as new multihashes are incrementally ingested within the same
  context ID by the same provider.

* **Federation Epoch Awareness**: The API maintains consistent multihash sampling across indexer instances within the
  same federation by considering the federation epoch.

* **Seed-Based Sampling**: Users can specify a seed derived from decentralized randomness beacons like Drand. This
  guarantees deterministic sampling and just-in-time candidate selection. Queries repeated with the same seed produce
  identical results, facilitating reproducibility.

The sampling process utilizes the PCG (Permuted Congruential Generator) random number generator, seeded to ensure
deterministic output. To efficiently perform sampling, two pre-emptive data organizational strategies are proposed:

1. **Counting Multihashes per Context ID**: Indexers pre-calculate the total multihash count per provider ID for each
   context ID.
2. **Sorting Multihashes by Hash Value**: Indexers store multihash records grouped by provider ID and context ID, sorted
   by hash value. This complements a global multihash index linked to provider information.

The sampling process involves initializing the PCG with the seed and selecting the required indices with the upper bound
being the total multihashes per context ID. The selected multihashes are then retrieved from the sorted list by hash
value.

While other deterministic sampling algorithms don't require these pre-emptive data organizational strategies, the
proposed method is more efficient for large datasets. It also supports efficient bulk deletions and deeper meta-analysis
of indexed records, such as:

* The number of records advertised by a provider per context ID.
* The number of unique content IDs advertised by a provider.
* The total number of unique hashes known by an indexer.
* And more.

A notable benefit of this approach is efficient bulk deletion. The sorted storage of multihashes allows for more
efficient deletion of ranges rather than individual multihashes. This capability reduces the need for opportunistic
deletions during lookup flows, which have low cleanup rates and high write amplification.

### API

#### `GET /ipni/v0/sample/{provider-id}/{context-id}`

Samples a set of multihashes ingested by an IPNI indexer for a given provider ID and Context ID.

- **Path Parameters**:
    - `provider-id`: (string, required) The multibase encoded peer ID of the provider.
        - Example: `12D3KooWN34sqTaMfZE3ReELyVF3no3qU7883Mi6j2VWsv6dwhPL`
    - `context-id`: (string, required) The multibase encoded context ID.

- **Query Parameters**:
    - `seed`: (optional) The seed for deterministic sampling. Ensures repeatability of samples.
    - `max`: (optional) The maximum number of multihashes to return. Defaults to one if unspecified.
    - `federation_epoch`: (optional) The IPNI federation epoch, currently only accepting zero, pending review of IPNI
      federation protocol.

- **Response**:
    - `200 OK`: A list of sampled multihash objects, each containing information based on the input parameters.
    - `404 Not Found`: If the provider ID or context ID does not exist.
    - `400 Bad Request`: If the request parameters are invalid.

## Notes to Implementers

### Security Considerations

The **IPNI Multihash Sampling** implementation demands several key security measures to safeguard against misuse:

1. **Maximum Sample Size**: Implementers must enforce a maximum sample size to prevent overconsumption of resources and
   mitigate risks associated with check exhaustion against particular providers.

2. **Rate Limiting**: To deter abuse and ensure equitable usage, the API enforces rate limits on requests per client.
   These limits can be adjusted at the discretion of the implementer but should remain set to reasonable values to avoid
   impairing the system's ability to process advertisements efficiently.

3. **Federation Epoch Look-Back**: The API restricts sampling to the current federation epoch and optionally a limited
   number of previous epochs, preventing potential network splitting attacks by limiting exposure to older samples that
   may no longer be representative of the current state of the network.

### Storage Complexity

Implementers need to consider the storage architecture that efficiently manages indices and records by context IDs while
addressing a key challenge: all information must be namespaced by both provider ID and context ID. This requirement
arises because a single multihash can be provided by one or multiple providers across various context IDs, leading to
increased complexity in data organization. Approaches to manage this complexity include:

1. **Compaction Techniques**: Implementing compact storage formats is essential for addressing redundancies in multihash
   storage. By storing each duplicate hash value once and maintaining references to all associated providers and context
   IDs, systems can reduce physical storage requirements while ensuring efficient data retrieval.

2. **Database Partitioning**: Database partitioning strategies can help manage the namespace complexity by logically
   segmenting data. For instance, partitioning data by provider ID and context ID allows systems to handle large
   datasets more effectively and optimize query performance specific to particular namespaces.

3. **Indexing Optimization**: Advanced indexing techniques, such as hierarchical indexes, can be tailored to navigate
   the namespace complexity introduced by multiple providers and context IDs. By optimizing index paths, systems can
   improve lookup speeds while minimizing redundant data storage.

4. **Deduplication Algorithms**: Deduplication algorithms play a vital role in mitigating storage redundancy. By
   identifying duplicate hash values across different namespaces, these algorithms enable the system to maintain a
   single instance of each multihash with metadata that links to all relevant providers and context IDs.

5. **Efficient Provider Lookup**: To handle the complexity of namespaced information, facilitating efficient lookup
   paths is critical. These paths should be designed to consider provider IDs and context IDs, ensuring minimal latency
   and computational burden during queries. Caching strategies can be employed to speed up access to frequently queried
   data.

6. **Bulk Data Removal**: The use of namespacing records by provider and context ID facilitates bulk data removal
   processes. This design decision is particularly valuable in managing datasets with wide variety of provider records,
   as it allows for the swift elimination of outdated or irrelevant data, reducing wasted space more effectively than
   opportunistic per multihash data deletion.

### Performance Optimizations

To achieve high performance and scalability, the implementers should employ several optimizations:

1. **Indexing**: Multihash records are indexed by their provider ID, context ID and hash value, enabling efficient
   lookups, bulk deletion, better bookkeeping/meta-analysis and random sampling.

2. **Caching**: In a distributed checker network scenario, caching sampled multihashes can reduce the load on the
   sampling service and improve response times for subsequent queries. Because, the expected usage pattern is typically
   to sample the same multihashes multiple times via a swarm of checkers.

3. **Concurrent Ingestion**: Advertisements from multiple providers across context IDs are concurrently ingestable. This
   means that the maximum theoretical degree of concurrent ingestion per provider is the number of context IDs they
   advertise. Although, the practical concurrency is limited by the underlying storage system's write throughput.

#### Indexing

The following list outlines the desirable properties of the indexing mechanism that should be considered during the
implementation of the sampling service:

- **Provider Size Profiling**: Implement mechanisms to track and profile providers based on their advertisement volume
  and data diversity metrics. This information aids in load balancing, resource allocation, and better quality assurance
  per provider.
- **Unique Content Monitoring**: Monitor and report on the total count of unique content IDs indexed, aiding in
  assessing dataset coverage and redundancy.
- **Optimized Deletions**: Ensure the capability for bulk removals when providers deprecate data, maintaining an
  up-to-date index that reflects current provider statuses.

#### Caching

To enhance performance and data availability, implementers should employ caching strategies using HTTP headers like
`Cache-Control`:

- **Dynamic Caching**: Configure caches to store frequently requested multihash data temporarily, improving response
  times for repeated queries.

- **Cache Management**: Use cache directives such as `no-cache`, `max-age`, and `stale-while-revalidate` to manage cache
  lifecycles effectively, aligning cache staleness parameters with provider update frequencies to optimize resource use
  without sacrificing data accuracy.

#### Concurrent Ingestion

Efficient data grouping by context ID is essential for improving lookup speeds and minimizing the scope of data queries.
Implement indexing strategies that allow rapid filtering and retrieval based on context IDs, possibly through clustered
indexing or partitioning that aligns data storage with context-based access patterns. This approach has the added
benefit of reducing the blast radius of churn in mutlihash records advertised by a provider relative to other providers.

## Related Resources

- [Retrieval Checking Requirements FRC](https://github.com/filecoin-project/FIPs/pull/1089) - Establishes complementary
  specifications for assessing retrieval success across decentralized datasets.
- [IPNI Federation Protocol Proposal](https://github.com/ipni/specs/pull/27) - Envisions uniformity in sampling
  operations across federated indexers.
- [Reverse Index OpenAPI Spec](https://github.com/ipni/xedni/blob/master/openapi.yaml) - The OpenAPI specification for
  the reverse index service.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).