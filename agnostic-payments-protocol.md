---
title: Agnostic Payments Protocol (APP)
type: working-draft
draft: 1
---

# Agnostic Payments Protocol

Agnostic Payments is a protocol for sending packets of money across different payment networks or ledgers. APP can be integrated with any type of ledger, including those not built for interoperability, and it is designed to be used with a variety of higher-level protocols that implement features ranging from quoting to sending larger amounts of value with chunked payments.

## Overview

### Design Goals

- **Neutrality** - Agnostic Payments is not tied to any company, currency, or network.
- **Interoperability** - APP should be usable across any type of ledger, even those that were not built for interoperability.
- **Security** - Senders, connectors, and receivers should be protected from one another and especially isolated from risks posed by parties that they are indirectly connected to. Connectors should not be able to steal money from senders, and senders should not be able to tie up too much of connectors' funds or otherwise interfere with their operation.
- **Simplicity** - The core APP should be as simple as possible. The Agnostic Payments layer is the one part where widespread agreement is needed for interoperability. Simplifying the core minimizes what people need to agree upon and enables broader adoption.
- **End-to-End Principle** - Inspired by the Internet, any features that do not need to be implemented by the core Agnostic Payments Protocol and network of connectors should be built into the edges of the network (the sender and receiver).

### Terminology

- **Account** - Two peers establish an account with one another.
  * **Asset, Scale, and Precision** - Each account is denominated in a single asset and peers agree on the precision and scale they will use for the amounts of that asset when they express amounts in messages between one another (for example, a scale of 9 means that an amount of 1000000000 in an APP packet sent between those peers represents 1 unit of that asset). When a connector receives an APP Prepare packet, they use the asset type and scale of the connection through which the packet was received to determine the meaning of the amount in the packet.
  * **Difference from "Ledger Accounts"** - Note that the mentions of "accounts" in this document refer to those that Agnostic Payments participants _hold with one another_, rather than the "accounts" they may hold on underlying ledgers.
- **Bandwidth** - Connectors may limit the total value of APP packets an account can send during a given period. The limit may be used to prevent one account from tying up all of the connector's bandwidth with its peers.
- **Ledger** - A generic term for any system that tracks transfer of value between, and balances on, accounts.
- **Participant** - A participant in an Agnostic Payments payment has accounts with one or more other participants and takes one or more of the following roles:
  * **Sender** - The party that originates payment
  * **Receiver** - The final recipient of a payment
  * **Connector** - The intermediaries between a sender and a receiver that forward APP Packets. They may generate revenue from spreads on currency conversion, through subscription fees, or other means.
- **Payment** - In the context of this specification, a payment is understood to mean the transfer of value from the sender (payer) to the receiver (payee). Higher-level protocols may execute a "payment" by sending a series of APP Packets whose sum is equal to the desired payment value.
- **Peer** - A participant with which another participant holds an account.
- **Transport and Application Layer Protocols** - Senders and receivers generally use higher-level protocols to agree on the condition and other details for each APP packet. These protocols may also handle encryption and authentication of the data sent with APP packets.

### Why Unconditional Payment Channels

The only requirement for ledgers to be integrated into APP is that they must be able to make simple transfers.

## Flow

### Prerequisites

1. Sender has an account with at least one connector
2. Sender and connector have funds on some shared ledger
3. Sender and connector have an authenticated communication channel through which they will send APP packets, such as a Secure WebSocket (WSS) or Hypertext Transfer Protocol (HTTP). Note that connectors use their authenticated communication channels with their peers and customers to determine the source of each packet (packets do not contain the source address).
4. Sender and receiver use a higher-level protocol and/or out-of-band authenticated communication channel to agree on the destination account.

### APP Packet Lifecycle

1. Sender constructs an APP Prepare packet with the receiver's destination account and an amount and expiry of their choice. The sender may include additional data, the formatting of which is determined by the higher-level protocol.
2. Sender sends the APP Prepare packet to a connector over their authenticated communication channel.
3. Connector gets the packet and determines which account to debit based on which communication channel it was received through (each account holder will either have an open connection, such as a WebSocket, or each message will be sent with authentication information, such as in an HTTPS request).
4. Connector checks whether the sender has sufficient balance or credit to send the amount specified in the packet. If not, the connector returns an APP Reject packet.
5. Connector uses its local routing tables and the destination APP address from the packet to determine which next hop to forward the packet to. The connector may use an exchange orderbook or any other method it chooses to set its exchange rate between the incoming and outgoing assets.
6. Connector modifies the packet amount and expiry to apply their exchange rate and move the expiry timestamp earlier (for example, each connector may subtract one second from the expiry to give themselves a minimum of one second to pass the Fulfill on in step 10).
7. Connector forwards the packet to the next hop, which may be another connector. All subsequent connectors go through steps 3-7 (treating the previous connector as the sender) until the packet reaches the receiver.
8. Receiver checks the APP Prepare packet, according to whatever is stipulated by the higher-level protocol (such as checking whether the amount received is above some minimum specified by the sender).
9. To accept the packet, the receiver returns an APP Fulfill packet. The receiver sends the Fulfill back through the same communication channel the Prepare packet was received from. The receiver may include additional data in the Fulfill packet, the formatting of which is determined by the higher-level protocol. If the receiver does not want the APP Prepare packet or it does not pass one of the checks, the receiver returns an APP Reject packet instead.
10. If the receiver returned an APP Fulfill, the connector checks that the receiver returned it before the expiry in the APP Prepare packet. If the fulfillment is valid, the connector returns the same APP Fulfill packet to the previous connector (using the same communication channel they originally received the APP Prepare from) and the connector provides the receiver's account with the amount in the original APP Prepare packet. If the receiver returned an APP Reject packet or the Prepare expired before the Fulfill was returned, the connector returns an APP Reject packet to the previous connector (note that connectors SHOULD return APP Reject packets when the Prepare expires, even if they have not yet received a response from the next participant). Each connector repeats this step (sending funds to the account of the participant that returned the Fulfill) until the Fulfill (or Reject) packet reaches the sender.
11. Sender repeats the process, starting from step 1, as many times as is necessary to transfer the desired total amount of value.

## Specification

#### APP Prepare

APP Prepare packets are `type` 12.

The `amount` and `expiresAt` are fixed-length fields so that they may be modified in-place (without copying the rest of the packet) by each connector. These are the only fields that are modified by connectors as the packet is forwarded.

| Field | Type | Description |
|---|---|---|
| `amount` | UInt64 | Local amount, denominated in the minimum divisible unit of the asset of the bilateral relationship. This field is modified by each connector, whoapplies their exchange rate and adjusts the amount to theappropriate scale and precision of the outgoing account |
| `expiresAt` | Fixed-Length [Agnostic Payments Timestamp](../asn1/Agnostic PaymentsTypes.asn) | Date and time when the packet expires. Each connector changes the value of this field to set the expiry to an earlier time, before forwarding the packet. |
| `destination` | [APP Address](../0015-APP-addresses/0015-APP-addresses.md) | APP Address of the receiver |
| `data` | Variable-Length Octet String | End-to-end data. Connectors MUST NOT modify this data. Most higher-level protocols will encrypt and authenticate this data, so receivers will reject packets in which the data is modified |

**Note:** There is no `from` or `source` address in the APP Prepare packet. Each hop MUST determine which of their immediate peers or customers the packet came from (to debit the correct account) based on the _authenticated_ communication channel the packet was received through. Also, higher-level protocols MAY communicate the sender's address to the receiver if the use case calls for receivers to send APP Prepare packets to the sender (they can communicate responses to the sender using the `data` field in APP Fulfill and Reject packets).

See the ASN.1 definition [here](../asn1/Agnostic PaymentsProtocol.asn).

#### APP Fulfill

APP Fulfill packets are `type` 13.

| Field | Type | Description |
|---|---|---|
| `data` | Variable-Length Octet String | End-to-end data. Connectors MUST NOT modify this data |

See the ASN.1 definition [here](../asn1/Agnostic PaymentsProtocol.asn).

#### APP Reject

APP Reject packets are `type` 14.

| Field | Type | Description |
|---|---|---|
| `code` | 3-Character IA5String | [APP Error Code](#error-codes) |
| `triggeredBy` | [APP Address](../0015-APP-addresses/0015-APP-addresses.md) | APP Address of the party that created this error |
| `message` | UTF8 String | User-readable error message, primarily intended for debugging purposes |
| `data` | Variable-Length Octet String | End-to-end data. Connectors MUST NOT modify this data |

See the ASN.1 definition [here](../asn1/Agnostic PaymentsProtocol.asn).

### Error Codes

#### F__ - Final Error

Final errors indicate that the payment is invalid and should not be retried unless the details are changed.

| Code | Name | Description | Data Fields |
|---|---|---|---|
| **F00** | **Bad Request** | Generic sender error. | (empty) |
| **F01** | **Invalid Packet** | The APP packet was syntactically invalid. | (empty) |
| **F02** | **Unreachable** | There was no way to forward the payment, because the destination APP address was wrong or the connector does not have a route to the destination. | (empty) |
| **F03** | **Invalid Amount** | The amount is invalid, for example it contains more digits of precision than are available on the destination ledger or the amount is greater than the total amount of the given asset in existence. | (empty) |
| **F04** | **Insufficient Destination Amount** | The receiver deemed the amount insufficient, for example, the effective exchange rate was too low. | (empty) |
| **F05** | **Unexpected Payment** | The receiver was not expecting a payment like this (the data and destination address don't make sense in that combination, for example if the receiver does not understand the transport protocol used) | (empty) |
| **F06** | **Cannot Receive** | The receiver (beneficiary) is unable to accept this payment due to a constraint. For example, the payment would put the receiver above its maximum account balance. | (empty) |
| **F07** | **Amount Too Large** | The packet amount is higher than the maximum a connector is willing to forward. Senders MAY send another packet with a lower amount. Connectors that produce this error SHOULD encode the amount they received and their maximum in the `data` to help senders determine how much lower the packet amount should be. |
| **F08** | *Application Error** | Reserved forapplication layer protocols.Applications MAY use names other than Application Error`. | Determined byApplication |