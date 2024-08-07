---
permalink: /RS-016
---
# PoWeb: Parcel Delivery over HTTP and WebSockets, Version 1
{: .no_toc }

- Id: RS-016.
- Status: Working draft.
- Type: Implementation.
- Reference implementations: [client](https://github.com/relaycorp/relaynet-poweb-js), [server](https://github.com/relaycorp/relaynet-internet-gateway/tree/master/src/services/poweb).

## Abstract
{: .no_toc }

This document describes _PoWeb_, a binding for [Gateway Synchronization Connections (GSC)](rs000-core.md#gateway-synchronization-binding) on top of the HTTP and [WebSockets (RFC-6455)](https://tools.ietf.org/html/rfc6455) protocols.

## Table of contents
{: .no_toc }

1. TOC
{:toc}

## Introduction

As a GSC, the ultimate objective of PoWeb is to enable a gateway to relay parcels for its private endpoints or its private peers. In other words, a private gateway would implement a PoWeb service to exchange parcels with its local endpoints, whilst a Internet gateway would implement a PoWeb service to exchange parcels with its private gateways.

PoWeb owes its name to the fact it uses Web-based, Application Layer protocols with its clients: HTTP for remote procedure calls, and WebSockets to stream messages (e.g., parcels). This specification describes the HTTP and WebSocket endpoints that comprise PoWeb, along with their inputs, outputs and any side effects. The serialisation format of the messages exchanged via this protocol are outside the scope of this document, as they are agnostic of the concrete GSC binding.

The private or Internet gateway implementing the PoWeb service will be referred to as the "server", whilst the private endpoint or private gateway connecting to its will be referred to as the "client".

Clients are required to register their endpoints with the gateway before they can start exchanging messages (e.g., parcels), given that those operations require the client to produce digital signatures using [Awala PKI](rs002-pki.md) certificates previously issued by the server.

## Endpoints

The server MUST expose each PoWeb endpoint under a common URL prefix ending with the path `/v1` (e.g., `https://gateway.com/v1`). Such prefixes MAY include paths and non-standard ports (e.g., `https://gateway.com:1234/path/v1`).

### Private node registration

Per [the core spec](rs000-core.md#private-node-registration),
this process is divided into two steps: pre-registration and registration.

#### Pre-registration endpoint

Servers MUST implement the pre-registration endpoint described in this document, unless the server is part of a private gateway on a mobile platform, in which case it MUST implement a platform-specific method that also authenticates the app that owns the endpoint.

To pre-register a private node, the client MUST make a POST request to `/pre-registration` with the SHA-256 hexadecimal digest of the private node's public key. The request Content-Type MUST be `text/plain`.

The server MUST respond with one of the following:

- A `200 OK` if the pre-registration is authorised. The response body MUST be the [Private Node Registration Authorisation (PNRA)](rs000-core.md#private-node-registration). The Content-Type MUST be `application/vnd.awala.node-registration.authorization`.
- A `400 Bad Request` if the client did not adhere to the requirements above.

Servers SHOULD refuse request bodies larger than 64 octets.

#### Registration endpoint

To complete the registration, the client MUST send the [Private Node Registration Request (PNRR)](rs000-core.md#private-node-registration) in the body of a POST request to `/nodes`, using the Content-Type `application/vnd.awala.node-registration.request`.

The server MUST respond with one of the following:

- `200 OK` if the registration was successfully completed. The response body MUST be a [Private Node Registration (PNR)](rs000-core.md#private-node-registration). The response Content-Type MUST be `application/vnd.awala.node-registration.registration`.
- `400 Bad Request` if the PNRR sent by the client is malformed or its digital signature is invalid.
- `403 Forbidden` if the client is using a PNRA issued for a different private node.

Servers SHOULD refuse request bodies larger than 1 MiB.

### Parcel delivery

To deliver a parcel to the server, the client MUST send the parcel in the body of a POST request to `/parcels` using the Content-Type `application/vnd.awala.parcel`. Additionally, it MUST countersign the parcel with the private node's keys and include the base64 encoding of the resulting digital signature in the `Authorization` header, using the `Awala-Countersignature` type.

The server MUST respond with one of the following:

- `202 Accepted` if the parcel and its countersignature are both valid, and the parcel was successfully stored or forwarded.
- `400 Bad Request` if the parcel is malformed.
- `401 Unauthorized` if the countersignature was missing or malformed.
- `403 Forbidden` if the countersignature was invalid for the parcel.
- `422 Unprocessable Entity` if the parcel was well-formed but invalid; e.g., the [RAMF](rs001-ramf.md) signature verification failed.

### Parcel collection

To collect parcels from the server, the client MUST start a WebSocket connection with the endpoint `/parcel-collection`.

The server MUST send a challenge message to the client as soon as the connection starts. This challenge MUST encapsulate a cryptographic nonce. The client MUST reply to the challenge by sending a response that encapsulates the digital signatures for the nonce using the keys for each private node on whose behalf parcels will be collected. The digital signatures MUST meet the following requirements:

- They MUST be DER-encoded, CMS SignedData values and be validated in accordance to [RFC 5652](https://tools.ietf.org/html/rfc5652).
- The signer's certificate MUST be encapsulated. Other certificates SHOULD NOT be encapsulated.
- The signer's certificate MUST be issued by the underlying gateway behind the server.
- The signed content MUST NOT be encapsulated.

The server MUST close the WebSocket connection with the status code `1003` if there are no signatures or at least one of the signatures does not meet any of the requirements above.

If the handshake is completed successfully, the server MUST send any queued parcels to the client, and the client MUST send an acknowledgement message back to the server for each parcel that is successfully stored or forwarded. Upon reception of each acknowledgement, the server MUST delete the parcel immediately or schedule its deletion.

Finally, the server MUST close the connection with the status code `1000` if the client set the HTTP request header `X-Awala-Streaming-Mode` to `close-upon-completion` and all parcels were send and acknowledged. Otherwise, both peers SHOULD try to keep the connection open indefinitely, and the server MUST send any new parcels as soon as they are received.

### Streaming mode

The client MAY optionally specify whether the server should close the connection as soon as all queued parcels have been collected by setting the HTTP request header `X-Awala-Streaming-Mode`.

If the header is set to `close-upon-completion`, the server MUST close the connection as soon as all parcels have been collected. The server MUST use the close code `1000` (normal closure) if the all parcels were sent. Alternatively, the server MUST use the code `4000` when not all parcels may have been sent, but the connection has to be closed nonetheless.

If the header is to set to `keep-alive`, the server SHOULD keep the connection open until the client closes it. If the server has to close the connection sooner, it MUST end the connection with the WebSocket status code `4000`.

The client MAY attempt another collection immediately, following a `4000` code from the server.

### Pings

The client and the server use WebSocket pings to detect broken connections.

The server MUST send a ping to the client every 5 seconds, to which the client MUST respond with a pong frame as soon as is practical. Additionally:

- The client SHOULD terminate the connection if 7 seconds or more have elapsed since the last ping.
- The server SHOULD terminate the connection if 9 seconds or more have elapsed since the last pong.

### Close codes

If the server initiates the closure of the connection, it SHOULD use one of the following WebSocket codes:

- `1000` to signal that the collection completed successfully and all queued parcels were collected. It only applies to the `close-upon-completion` streaming mode.
- `1003` if the client sent an invalid handshake response or collection acknowledgement.
- `1008` if the client is likely to be a Web browser (i.e., the request included the `Origin` header).
- `1011` if there was an internal error in the server, which prevented the collection request from being fulfilled.
- `4000` if the client may re-attempted to collect parcels immediately.

On the other hand, if the client initiates the closure of the connection, it SHOULD use one of the following codes:

- `1000` if the collection should be interrupted.
- `1003` if the server sent an invalid handshake challenge or collection.

## Undocumented HTTP status codes

The server MAY return any of the following status codes, but their semantics are not within the scope of this document:

- Any other code in the range 400-499 that the implementer or operator of the server deems appropriate but is not explicitly documented in the respective endpoint. The client SHOULD NOT try to make the same request later.
- Any code in the range 500-599 that the implementer or operator of the server deems appropriate. The client SHOULD try to make the same request later.

## Port numbers

If the server is a private gateway, it SHOULD listen on port 276 if it has the appropriate permissions to do so. If not, it SHOULD listen on port 13276.

If the server is a Internet gateway, it SHOULD listen on port 443.

## HTTP Considerations

Clients and servers implementing this specification MUST comply with HTTP version 1.1.

## WebSocket Considerations

Clients and servers implementing this specification MUST comply with the WebSocket specification ([RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)).

## Security Considerations

To prevent web pages from trying to make unauthorised requests to the server of a private gateway, the server MUST NOT support CORS and WebSocket connections MUST be refused with the status code `1008` if the HTTP request included the header `Origin`.

## Internet gateway SRV records

Because this binding uses the Transmission Control Protocol (TCP; [RFC 793](https://tools.ietf.org/html/rfc793)), SRV records for GSC servers implementing this binding MUST use the `tcp` protocol.

For example, a Internet gateway like `example.com` could specify an SRV record as follows:

```
_awala-gsc._tcp.example.com. 300 IN SRV 0 1 443 poweb.example.com.
```

## Relevant Specifications

[Awala Core (RS-000)](rs000-core.md) defines the requirements for [message transport bindings](rs000-core.md#message-transport-bindings) in general and [parcel delivery bindings](rs000-core.md#parcel-delivery-binding) specifically, all of which apply to PoWeb. [Awala PKI (RS-002)](rs002-pki.md) is also particularly relevant to this specification.
