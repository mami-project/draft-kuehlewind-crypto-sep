---
title: Separating Crypto Negotiation and Communication
abbrev: crypto separation
docname: draft-kuehlewind-crypto-sep-00
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

informative:
   RFC5246:
   I-D.ietf-quic-tls:
   I-D.moskowitz-sse:


--- abstract

Based on the increasing deployment of session resumption mechanisms where cryptographic context
can be resumed to transmit application data with the first packet without delay for connection setup
and negotiation, this draft proposes a split to separate connections used to set up encryption
context and negotiate capabilities from connections used to transmit application data.
While cryptographic context and endpoint capabilities need to be be known before encrypted
application data can be sent, there is otherwise no technical constraint that the crypto handshake
has to be performed on the same transport connection. This document discusses requirements on the
cryptographic protocol to establish medium- to long-lived association that can be used by different
transport protocols that implement different transport services.

--- middle

# Introduction

Secure transport protocols are generally composed of three pieces:

1. A transport protocol to control the transfer of data.
2. A record protocol to encode and, optionally, encrypt packets or datagrams.
3. A handshake protocol to negotiate cryptographic secrets.

For ease of deployment and standardization, among other reasons, these constituents are often tightly
coupled. For example, in TLS {{RFC5246}}, the handshake protocol depends on the record protocol,
and vice versa. However, more recent transport protocols such as QUIC {{I-D.ietf-quic-tls}} keep
these pieces separate. QUIC uses TLS to negotiate secrets, and *exports* those secrets
to encrypt packets directly.

~~~
+---------------+                +---------------+
|               +---------------->               |
|   Transport   |                |   Handshake   |
|               <----------------+               |
+-+-----^-------+                +-----+-----^---+
  |     |                              |     |
  |     |                              |     |
  |     |                              |     |
  |     |        +---------------+     |     |
  |     +--------+               <-----+     |
  |              |    Record     |           |
  +-------------->               +-----------+
                 +---------------+
~~~

This separation is important, as new secure transport protocols increasingly rely on
session resumption mechanisms where cryptographic context can be resumed to transmit
application data with the first packet without delay for connection setup and negotiation.
In the case where there is no cryptographic context available when an application
expressed the wish to transmit data to a certain endpoint, the connection for cryptographic
negotiation must be established first, immediately before the actual payload connection
will be used. In this case, as today for approaches that integrate both the cryptographic
handshake and the payload transmission, the application data transmission is delayed until
the needed cryptographic context is available. If the handshake and transport components
are separate, then the handshake protocol can use a separate transport connection to
establish secrets without blocking the application's main transport connection. Moreover,
this negotiation could be performed a priori and out of band before the application expresses
a desire to send data. For example, an integrated or independent software
system could maintain knowledge about endpoints that are likely to be communication points and set up
or refresh state any time triggered by external events such as the start up of this system or periodically.

This document discusses high-level interface requirements for separating the three
primary features of a secure transport protocol.

{{I-D.moskowitz-sse}} proposes a similar approach. However while {{I-D.moskowitz-sse}} proposes
a new protocol to negotiate and maintain long-term cryptographic sessions,
this document relies on the use of existing protocols and only discusses requirements for the evolution
of these protocols and exchange of information within one endpoint locally.

# Minimal Transport Security Features

Transport functionality aside, there are several critical features common to most
transport security protocols. Some of these features belong to the handshake
piece of the protocol, whereas others belong to the record piece. We highlight
these features in this section.

## Mandatory Features

### Handshake

- Forward-secure segment encryption and authentication: Transit data must be protected with an
authenticated encryption algorithm.

- Private key interface or injection: Authentication based on public key signatures is commonplace for
many transport security protocols.

- Endpoint authentication: The endpoint (receiver) of a new connection must be authenticated before any
data is sent to said party.

- Source validation: Source validation must be provided to mitigate server-targeted DoS attacks. This can
be done with puzzles or cookies.

### Record

- Pre-shared key support: A record protocol must be able to use a pre-shared key established
out-of-band to encrypt individual messages, packets, or datagrams.

## Optional Features

### Handshake

- Mutual authentication: Transport security protocols should allow both endpoints to authenticate one another if needed.

- Application-layer feature negotiation: The type of application using a transport security protocol often requires
features configured at the connection establishment layer, e.g., ALPN {{RFC7301}}. Moreover, application-layer features may often be used to
offload the session to another server which can better handle the request. (The TLS SNI is one example of such a feature.)
As such, transport security protocols should provide a generic mechanism to allow for such application-specific features
and options to be configured or otherwise negotiated.

- Configuration extensions: The protocol negotiation should be extensible with addition of new configuration options.

- Session caching and management: Sessions should be cacheable to enable reuse and amortize the cost of performing
session establishment handshakes.

### Record

- Connection mobility: Sessions should not be bound to a network connection (or 5 tuple). This allows cryptographic
key material and other state information to be reused in the event of a connection change. Examples of this include
a NAT rebinding that occurs without a client's knowledge.

# Crypto-Transport Interface

There are two basic approaches: either the transport protocol can provide data to the crypto engine
and get back an encrypted version of the data to be sent, or the crypto protocol can provide keying
material and inform the transport about the negotiated capabilities of the far end and the transport
is responsible to perform the encryption set. In both cases, we seek to decouple the crypto layer
from the transport layer. Thus, we require some interface between the transport
and record protocols. Based on the minimum set of features described in the previous
section, we now describe a minimal interface for these components.
Here, a minimal interface defines the set of calls that an application interface must
expose in order to generically use a particular feature. The mandatory interfaces are
required to functionally use the protocols, while the optional interfaces allow the
application to constrain the protocol or retrieve extra information.

## Mandatory Interfaces

- Start negotiation: The interface MUST provide an interface to start the protocol handshake for key negotiation, and
have a way to be notified when the handshake is complete.

- State changes: The interface MUST provide a way for the application to be notified of important state changes during
the protocol execution and session lifetime, e.g., when the handshake begins, ends, or when a key update occurs.

- Identity constraints: The interface MUST allow the application to constrain the identities that it will accept
a connection to, such as the hostname it expects to be provided in certificate SAN.

- Local identities: The interface MUST allow the local identity to be set via a raw private key or interface to one
to perform cryptographic operations such as signing and decryption.

- Validation: The interface MUST provide a way for the application to participate in the endpoint authentication and validation,
which can either be specified as parameters to define how the peer's authentication can be validated, or when the protocol
provides the authentication information for the application to inspect directly.

- Key lifetime and rotation: The interface MUST provide a way for the application to set the key lifetime bounds in terms
of *time* or *bytes encrypted* and, additionally, provide a way to forcefully update cryptographic session keys at will.
The protocol should default to reasonable lifetimes barring any application input.

- Key export: The interface MUST either provide a way to export keying material with well-defined cryptographic properties,
e.g., "forward-secure" or "perfectly forward secure", or should provide an interface to keying material for cryptographic operations.

## Optional Interfaces

- Caching domain and lifetime: The application SHOULD be able to specify the instances of the protocol that can share
cached keys, as well as the lifetime of cached resources.

- The protocol SHOULD allow applications to negotiate application protocols and related information.

- The protocol SHOULD allow applications to specify negotiable cryptographic algorithm suites.

- The protocol SHOULD expose the peer's identity information.

# Existing Mappings

In this section we document existing mappings between common transport security
protocols and the three components described in Section I.

## TLS/DTLS

XXX

## QUIC

XXX

## IKEv2 + ESP

XXX

# IANA Considerations

This document has on request to IANA.

# Security Considerations

[editor's note: this section will be added later. However, this document discusses the use of
cryptograohic context for transport connections and as such it has security relevant consideration
within the whole document.]

# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply endorsement.
