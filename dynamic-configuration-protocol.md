---
title: Dynamic Configuration Protocol
type: working-draft
draft: 3
---

# Dynamic Configuration Protocol (DCP) v1

## Prerequisites
This specification assumes the reader is familiar with the following documents:

- [Agnostic Payments Architecture](../rfcs/architecture.md)
- [Agnostic Payments Protocol(APP)](../rfcs/agnostic-payments-protocol.md)

## Terminology

- A **node** is a participant in an Agnostic Payments network. It may be a [connector](../rfcs/architecture.md#connectors), a sender or a receiver. 
- Nodes are connected to each other and the relationship between two nodes is either:
  - `parent` and `child` (one node is the `parent` of another node that is relatively its `child`), or,
  - `peer` and `peer` (two nodes are peered with one another).

## Overview
In order to participate in the network a node must have an [APP address](../rfcs/app-addressses.md). This address is part of a heirarchical address space where child nodes MAY be allocated addresses within the address space of their parent node. This makes routing on the network more efficient than if all nodes had top-level addresses.

A node is therefore either configured with an address which it broadcasts to its peers or, if the node has a parent in the network hierarchy, the APP address of the child node can be allocated to it by the parent. This is done using DCP.

In short, **DCP is a protocol used for transferring node and ledger information from a parent node to a child node**. The transferred information is:

- An APP address that the child should use for its own address
  - e.g. `g.crypto.foo.bar`
- An asset code of the asset that the two nodes will use to settle
  - e.g. `ICP`
- An asset scale for amounts that will be used in APP packets exchanged between the nodes.
  - e.g. `6`

Future versions of the protocol MAY exchange additional configuration information.

## Protocol Detail

### Procedure
An exchange of configuration information is done using the following procedure.

1. A child node requests configuration information from the parent node
2. The parent node responds with the configuration information to the child node
3. If the request cannot be processed, the parent node responds with an error

### Packet
The request and the response above are transferred in [APP packets](../rfcs/agnostic-payments-protocol.md#specification). 

The packets are addressed directly to the peer using the `peer.` address prefix. DCP packets are specifically identified using the address `peer.config`.

All current implementations of DCP default to an amount of `0` in the APP packets used to request configuration. In future, peers may agree to pay a charge for reserving an address, in which case this may become a non-zero amount indicating the amount paid by the child to reserve an address. Future version of the protocol may allow for the child to explicitly request an address which may not be restricted to being in the parent's address-space.

The packet exchange goes as follows:

- Request
  - The `type` of the APP packet is `APP Prepare` (type id: 12)
  - The `amount` of the APP packet defaults to `0`.
  - The `expiresAt` of the APP packet is arbitrary.
  - The `destination` address of the APP packet is `peer.config`
  - The `data` of the APP packet is empty (size: 0)
  
- Response
  - The `type` of the APP packet is `APP Fulfill` (type id: 13)
  - The `data` of the APP packet
    
- Error
  - The `type` of the APP packet is `APP Reject` (type id: 14)
  - The `code` of the APP packet is an appropriate error code
  - The `message` of the APP packet is an appropriate human-readable message for debugging purposes
  - The `triggeredBy` of the APP packet is the APP address of the parent node
  - The `data` of the APP packet is empty (size: 0) or MAY contain further information for debugging the error.