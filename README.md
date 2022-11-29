# IPNI Specifications

> This repository contains the specs for the InterPlanetary Network Indexer (IPNI) Protocol and its
> associated subsystems.

* [Contribute](#contribute)
    + [Labels](#labels)
* [Code of Conduct](#code-of-conduct)

## Contribute

Suggestions, contributions, criticisms are welcome. To contribute, simply open a PR on this repo
with suggested changes to existing specification, or introduce a new one.

New specification proposals should:

* use the format outlined in [template](TEMPLATE.md) file.
* be marked with appropriate Label; see [Labels](#labels).

Proposed changes to existing specifications should:

* clearly document the motivation for change
* ideally avoid breaking changes.
    * when breaking changes are unavoidable, the proposal should outline affected systems, risk and
      mitigation.

### Labels

Specifications use the following labels to identify their state:

- ![wip](https://img.shields.io/badge/status-wip-orange.svg?style=flat-square) - A work-in-progress,
  possibly to describe an idea before actually committing to a full draft.
- ![draft](https://img.shields.io/badge/status-draft-yellow.svg?style=flat-square) - A draft that is
  ready to review, with sufficient detail to be _implementable_.
- ![reliable](https://img.shields.io/badge/status-reliable-green.svg?style=flat-square) - The spec
  has been adopted, implemented and can be used as a reference point to learn how the system works.
- ![stable](https://img.shields.io/badge/status-stable-brightgreen.svg?style=flat-square) - The spec
  is finalised. It may alter in the future but the system it specifies will not change
  fundamentally.
- ![permanent](https://img.shields.io/badge/status-permanent-blue.svg?style=flat-square) - The spec
  will not change.
- ![deprecated](https://img.shields.io/badge/status-deprecated-red.svg?style=flat-square) - The
  spec is no longer in use and should not be extended nor built upon.

## Code of Conduct

This repository follows the IPFS
Community [Code of Conduct](https://github.com/ipfs/community/blob/master/code-of-conduct.md).
