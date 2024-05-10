---
title: Agnostic Payment Architecture
type: working-draft
draft: 1
---

# Agnostic Payment Architecture

Agnostic Payments provides for secure payments across multiple assets on different ledgers. The architecture consists of a conceptual model for agnostic payments, a mechanism for securing payments. Colloquially, the whole Agnostic Payments stack is sometimes referred to as "APP".

Agnostic Payments is not a blockchain, a token, nor a central service. Agnostic Payments is a standard way of bridging ledgers. The Agnostic Payments architecture is heavily inspired by the Interledger architecture described in [RFC 0001](https://github.com/interledger/rfcs/blob/main/0001-interledger-architecture/0001-interledger-architecture.md), [RFC 574](0034-connector-requirements/0034-connector-requirements.md) and [RFC 0015](https://github.com/interledger/rfcs/tree/main/0015-APP-addresses).

## Core Concepts

### The Agnostic Payments

The "Agnostic Payments protocol" can be used among any network of participants, public or private. There is no single network that all parties must connect to to use this protocol.

### Sender, Receiver, Connectors

When two parties want to do business online, the one who sends money is the _sender_ and the one who gets money is the _receiver_. If the sender and the receiver don't have some monetary system in common, they need one or more parties to connect them. In the Agnostic Payments architecture, _connectors_ forward money through the network until it reaches the receiver.

![Diagram showing Sender linked to two Connectors and a Receiver in a line](https://raw.githubusercontent.com/Agnostic-Payment-Protocol/rfcs/34c23ad3041b749f16d5088d79a2e2fcc6d47832/graphs/interledger-model.svg)

Within these specifications, we use _node_ as a general term for any participant in the Agnostic Payments. A node may be a sender, a receiver, or a connector. The node may represent a person or business, or perhaps a single device or software program. In the case where a node represents a device or software program, the node is usually connected to another node that represents the device's owner or the person running the software.

You can envision the Agnostic Payments as a graph where the points are individual nodes and the edges are _accounts_ between two parties. Parties with only one account can send or receive through the party on the other side of that account. Parties with two or more accounts are _connectors_, who can facilitate payments to or from anyone they're connected to.

Connectors provide a service of forwarding packets and relaying money, and they take on some risk when they do so. In exchange, connectors can charge fees and derive a profit from these services. In the open network of the Agnostic Payments, connectors are expected to compete among one another to offer the best balance of speed, reliability, coverage, and cost.

#### Ledger Protocols

The Agnostic Payments protocol sits atop a level which we call "ledger protocols". This level represents the existing money systems that Agnostic Payments connects. Even though they are not strictly part of the Agnostic Payments protocol, they are an indispensable part of the Agnostic Payments model.

#### Agnostic Payments Protocol

The Agnostic Payments Protocol (APP) is the core protocol of the entire Agnostic Payments Protocol. This protocol's packets pass through all participants in the chain: from the sender, through one or more connectors, to the receiver. This protocol is compatible with any variety of currencies and underlying ledger systems.

This level is concerned with currency amounts, routing, and whether each step in a payment arrives in time or expires. This protocol finds a path to connect a sender and receiver using any number of intermediaries.

This layer abstracts the layers above and below it from one another, so there can be only one protocol at this layer. Other protocols, including older versions of the Agnostic Payments Protocol, are incompatible. The current protocol is defined by [RFC-02: Agnostic Payments Protocol](../rfcs/agnostic-payments-protocol).

#### Transport Protocols

Transport layer protocols are used for **end-to-end communication** between sender and receiver; connectors are not expected to be involved. This layer is responsible for:

- Grouping and retrying packets to achieve a desired outcome
- Determining the effective exchange rate of a payment
- Adapting to the speed at which packets can be sent, for what amounts
- Encrypting and decrypting data

#### Application Protocols

Protocols at the application level communicate details outside of the minimum information that is technically necessary to complete a payment. For example, application protocols may check that participants are interested in conducting a transaction and are legally allowed to do so.

Protocols on this layer are responsible for:

1. Destination account discovery ("Where should the money be sent, exactly?")
2. Destination amount negotiation ("Is it OK if this much arrives after fees and exchange rates?")
3. Optionally, any other information that should be communicated in APP packet data. ("Use this invoice identifier to credit the payment to your video subscription.")

### Agnostic Payments Protocol Flow

Agnostic Payments moves money by relaying _packets_. In the Agnostic Payments Protocol, a "prepare" packet represents a possible movement of some money. As the packet moves forward through the chain of connectors, the sender and connectors prepare balance changes for the accounts between them. The connectors also adjust the amount for any currency conversions and fees subtracted.

When the prepare packet arrives at the receiver, if the amount of money to be received is acceptable, the receiver fulfills the condition with a "fulfill" packet that confirms the balance change in the account between the last connector and the receiver. Connectors pass the "fulfill" packet back up the chain, confirming the planned balance changes along the way, until the sender's money has been paid to the first connector.

![Diagram: a prepare packet flows forward from Sender to Connector, to another Connector, to Receiver. A fulfill packet flows backward from Receiver to the second Connector, to the first Connector, to the Sender.](https://raw.githubusercontent.com/Agnostic-Payment-Protocol/rfcs/34c23ad3041b749f16d5088d79a2e2fcc6d47832/graphs/packet-flow-happy.svg)

At any step along the way, a connector or the receiver can reject the payment, sending a "reject" packet back up the chain. This can happen if the receiver doesn't want the money, or a connector can't forward it. A prepared payment can also expire, in all these cases, no balances change.

![Diagram: a prepare packet flows forward from Sender to Connector, to another Connector, who rejects it. A reject packet flows backward from the second Connector to the first Connector, then to the Sender.](https://raw.githubusercontent.com/Agnostic-Payment-Protocol/rfcs/34c23ad3041b749f16d5088d79a2e2fcc6d47832/graphs/packet-flow-reject.svg)

The flow is specified in detail in [IL-RFC-27: Agnostic Payments Protocol version 4](../rfcs/agnostic-payments-protocol).

#### Packetized Money

A packet does not have to represent the full amount of a real-world payment. _Transport protocols_ built on top of the Agnostic Payments Protocol can combine many small-value packets into a single large-value payment, rejecting and retrying individual packets as necessary to achieve the desired outcome. Using small packets (in terms of the amount of money represented, not the data size) in this way has several benefits for the network:

- Small packets reduce the "free option problem" where senders can lock up connectors' funds, then either take the promised deal or not depending on whether exchange rates move in their favor.
- Small packets involve small connectors (those with less funds available) in more transactions, and lower the barrier to entry for operating a connector, since moving less than the full amount of a payment can still be useful.

The Agnostic Payments Protocol does not have a specific definition of "small", nor a size limit on packets. Each connector can choose minimum and maximum packet sizes they are willing to relay; as a result, any path's maximum packet size is the smallest maximum packet size among the connectors in that path. To be compatible with as much of the network as possible, one should choose packet sizes that fit between the minimum and maximum values of as many connectors as possible.

### Addresses

_Agnostic Payments addresses_ (also called _APP addresses_) provide a universal way to address senders, receivers and connectors. These addresses are used in several different protocol layers, but their most important feature is to enable routing on the Interleder Protocol layer. Agnostic Payments addresses are hierarchical, dot-separated strings where the left-most segment is most significant. An example address might look like:
`g.us.acmebank.acmecorp.sales.199` or `g.crypto.bitcoin.1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2`.

For more information on the format of APP addresses and how they apply to routing, see [RFC-003: APP Addresses](../rfcs/app-addressses.md).

If two parties in the Agnostic Payments have a "parent/child" connection rather than a peer-to-peer connection, the child can request an Agnostic Payments address that is under the parent's address in the hierarchy. For more information, see [RFC-004: Agnostic Payments Dynamic Configuration Protocol (DCP)](../rfcs/0031-dynamic-configuration-protocol.md).

## Agnostic Payments Security

**Agnostic Payments uses _smart contracts or canisters_ to secure payments across multiple hops.** Connectors take some risk, but this risk can be managed and is primarily based upon the connector's chosen peers.

Because each party is isolated from risks beyond their immediate peers, longer paths are not inherently more risky than shorter paths. This enables longer paths to compete with shorter paths to convey money from any given sender to any given receiver, while reducing the risk to the sender.

### Connector Risk and Mitigation

Agnostic Payments connectors accept some risk in exchange for the revenue they generate from facilitating payments. In the Agnostic Payments payment flow, connectors incur outgoing obligations on the receiver-side account, before they become entitled to incoming obligations on the sender-side account. After each connector receives a `fulfill` packet, they have a window of time to deliver the `fulfill` packet to their counterparty on the sender-side account. Connectors that fail to deliver the `fulfill` packet in time may lose money.

If some connectors in the path of an Agnostic Payments packet receive the `fulfill` packet in time and others don't, the receiver will know the packet was received but the sender will not know. Usually, a single Agnostic Payments packet is part of a larger transaction or payment stream, so the sender may find out what happened when they receive the response for the next packet. Senders MAY also retry packets that expire.