[![Gitter chat](https://badges.gitter.im/relaynet/community.png)](https://gitter.im/relaynet/community)

# Relaynet Protocol Suite Specifications

This repository contains all the specifications part of [Relaynet](https://relaynet.link/).

The main specs are:

- [RS-000 (Relaynet Core)](rs000-core.md) defines the foundation of the protocol suite.
- [RS-001 (RAMF)](rs001-ramf.md) defines the _Relaynet Abstract Message Format_, an efficient binary format used to serialize messages.
- [RS-002 (Relaynet PKI)](rs002-pki.md) defines how to use the certificates for endpoints and gateways.
- [RS-003 (Key Agreement)](rs003-key-agreement.md) defines the key agreement protocol to establish and protect sessions between two gateways or two endpoints.
- [RS-012 (Service Integration Scale)](rs012-service-integration.md) categorizes the degrees to which Relaynet can be integrated in a service.
- [RS-014 (Ping)](rs014-ping.md) defines a trivial service to test the implementation and integration of Relaynet components.

_Cargo relay connections_ (CRCs) can be established with one of the following _bindings_. They are the transport medium for gateways to exchange _cargoes_ through a _relayer_.

- [RS-004 (CoSocket)](rs004-cosocket.md): Cargo relay over TCP/Unix Sockets. This is a purpose-built Layer 7 protocol, and the most efficient way to establish a CRC.
- [RS-006 (CoHTTP)](rs006-cohttp.md): Cargo relay over HTTP. An alternative to CoSocket, to lower the barrier to adopt Relaynet.
- [RS-008 (CogRPC)](rs008-cogrpc.md): Cargo relay over gRPC. Another alternative to CoSocket, to lower the barrier to adopt Relaynet.

_Parcel delivery connections_ (PDCs) can be established with one of the following _bindings_. They are the transport medium for an endpoint and a gateway to exchange _parcels_.

- [RS-005 (PoSocket)](rs005-posocket.md): Parcel delivery over TCP/Unix Sockets. This is a purpose-built Layer 7 protocol, and the most efficient way to establish a PDC.
- [RS-007 (PoHTTP)](rs007-pohttp.md): Parcel delivery over HTTP, for external PDCs.
- [RS-009 (PogRPC)](rs009-pogrpc.md): Parcel delivery over gRPC, for external PDCs.

Other specs:

- [RS-010](rs010-pdc-browser.md) defines a JavaScript interface that browsers or browser extensions can expose to make it easier and safer for client-side apps to send and receive parcels.
- [RS-011 (AsyncRPC)](rs011-asyncrpc.md) defines a service that encapsulates RPCs in Relaynet messages. Only meant as a steppingstone until the actual service supports Relaynet.
- [RS-013 (Message Broadcast)](rs013-pubsub.md) adds support for the [Publish-Subscribe pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) in Relaynet.
