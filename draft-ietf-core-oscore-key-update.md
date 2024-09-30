---
v: 3

title:  Key Update for OSCORE (KUDOS)
abbrev: Key Update for OSCORE (KUDOS)
docname: draft-ietf-core-oscore-key-update-latest

# stand_alone: true

ipr: trust200902
wg: CoRE Working Group
kw: Internet-Draft
cat: std
submissiontype: IETF
updates: 8613

coding: utf-8

author:
      -
        ins: R. Höglund
        name: Rikard Höglund
        org: RISE AB
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: rikard.hoglund@ri.se
      -
        ins: M. Tiloca
        name: Marco Tiloca
        org: RISE AB
        street: Isafjordsgatan 22
        city: Kista
        code: SE-16440 Stockholm
        country: Sweden
        email: marco.tiloca@ri.se

normative:
  RFC5869:
  RFC7252:
  RFC7641:
  RFC8613:
  RFC8949:
  RFC9528:

informative:
  RFC8446:
  RFC7554:
  RFC8180:
  RFC9031:
  RFC9200:
  RFC9203:
  RFC9176:
  RFC8615:
  RFC8724:
  RFC8824:
  RFC7967:
  RFC4086:
  I-D.irtf-cfrg-aead-limits:
  I-D.ietf-core-oscore-key-limits:
  I-D.ietf-ace-edhoc-oscore-profile:
  LwM2M:
    author:
      org: Open Mobile Alliance
    title: Lightweight Machine to Machine Technical Specification - Core, Approved Version 1.2, OMA-TS-LightweightM2M_Core-V1_2-20201110-A
    date: 2020-11
    target: http://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Core-V1_2-20201110-A.pdf
  Symmetric-Security:
    author:
      fullname: John Preuß Mattsson
      org: Ericsson Research
    title: Security of Symmetric Ratchets and Key Chains - Implications for Protocols like TLS 1.3, Signal, and PQ3
    date: 2024-02
    target: https://eprint.iacr.org/2024/220
  LwM2M-Transport:
    author:
      org: Open Mobile Alliance
    title: Lightweight Machine to Machine Technical Specification - Transport Bindings, Approved Version 1.2, OMA-TS-LightweightM2M_Transport-V1_2-20201110-A
    date: 2020-11
    target: http://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Transport-V1_2-20201110-A.pdf

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document defines Key Update for OSCORE (KUDOS), a lightweight procedure that two CoAP endpoints can use to update their keying material by establishing a new OSCORE Security Context. Accordingly, it updates the use of the OSCORE flag bits in the CoAP OSCORE Option as well as the protection of CoAP response messages with OSCORE, and it deprecates the key update procedure specified in Appendix B.2 of RFC 8613. Thus, this document updates RFC 8613. Also, this document defines a procedure that two endpoints can use to update their OSCORE identifiers, run either stand-alone or during a KUDOS execution.

--- middle

# Introduction # {#intro}

Object Security for Constrained RESTful Environments (OSCORE) {{RFC8613}} provides end-to-end protection of CoAP {{RFC7252}} messages at the application-layer, ensuring message confidentiality and integrity, replay protection, as well as binding of response to request between a sender and a recipient.

To ensure secure communication when using OSCORE, peers may need to update their shared keying material. Among other reasons, approaching key usage limits {{I-D.irtf-cfrg-aead-limits}}{{I-D.ietf-core-oscore-key-limits}} requires updating the OSCORE keying material before communications can securely continue.

This document updates {{RFC8613}} as follows.

* It specifies KUDOS, a lightweight key update procedure that the two peers can use in order to update their current keying material and establish a new OSCORE Security Context. This deprecates and replaces the procedure specified in {{Section B.2 of RFC8613}}.

* With reference to the "OSCORE Flag Bits" registry defined in {{Section 13.7 of RFC8613}} as part of the "Constrained RESTful Environments (CoRE) Parameters" registry group, it updates the entries with Bit Position 0 and 1 (see {{sec-iana}}), both originally marked as "Reserved". That is, it defines and registers the usage of the OSCORE flag bit with Bit Position 0, as the one intended to expand the space for the OSCORE flag bits in the OSCORE Option (see {{ssec-oscore-option-extensions}}). Also, it marks the bit with Bit Position of 1 as "Unassigned".

* It updates the protection of CoAP responses with OSCORE originally specified in {{Section 8.3 of RFC8613}}, as defined in {{sec-updated-response-protection}} of this document.

Furthermore, this document specifies a method that two peers can use to update their OSCORE identifiers. This can be run as a stand-alone procedure, or instead integrated in a KUDOS execution.

## Terminology ## {#terminology}

{::boilerplate bcp14-tagged}

Readers are expected to be familiar with the terms and concepts related to CoAP {{RFC7252}}, Observe {{RFC7641}}, CBOR {{RFC8949}}, OSCORE {{RFC8613}}, and EDHOC {{RFC9528}}.

This document additionally defines the following terminology.

* Initiator: the peer starting the KUDOS execution, by sending the first KUDOS message.

* Responder: the peer that receives the first KUDOS message in a KUDOS execution.

* Forward message flow: the KUDOS execution workflow where the initiator acts as CoAP client (see {{ssec-derive-ctx-client-init}}).

* Reverse message flow: the KUDOS execution workflow where the initiator acts as CoAP server (see {{ssec-derive-ctx-server-init}}).

* FS mode: the KUDOS execution mode that achieves forward secrecy (see {{ssec-derive-ctx}}).

* No-FS mode: the KUDOS execution mode that does not achieve forward secrecy (see {{no-fs-mode}}).

# Current Methods for Rekeying OSCORE {#sec-current-methods}

Two peers communicating using OSCORE may choose to renew their shared keying information by establishing a new OSCORE Security Context for a variety of reasons. A particular reason is approaching limits set for safe key usage {{I-D.ietf-core-oscore-key-limits}}. Practically, when the relevant limits have been reached for an OSCORE Security Context, the two peers have to establish a new OSCORE Security Context, in order to continue using OSCORE for secure communication. That is, the two peers have to establish new Sender and Recipient Keys, as the keys actually used by the AEAD algorithm.

In addition to approaching the key usage limits, there may be other reasons for a peer to initiate a key update procedure. These include: the OSCORE Security Context approaching its expiration time; application policies prescribing a regular key rollover; approaching the exhaustion of the Sender Sequence Number space in the OSCORE Sender Context.

It is RECOMMENDED that the peer initiating the key update procedure starts it with some margin, i.e., well before actually experiencing the trigger event forcing to perform a key update, e.g., the OSCORE Security Context expiration or the exhaustion of the Sender Sequence Number space. If the rekeying is not initiated ahead of these events, it may become practically impossible to perform a key update with certain methods, and/or without aborting ongoing message exchanges.

Other specifications define a number of ways for rekeying OSCORE, as summarized below.

* The two peers can run the procedure defined in {{Section B.2 of RFC8613}}. That is, the two peers exchange three or four messages, protected with temporary Security Contexts adding randomness to the ID Context.

   As a result, the two peers establish a new OSCORE Security Context with new ID Context, Sender Key, and Recipient Key, while keeping the same OSCORE Master Secret and OSCORE Master Salt from the old OSCORE Security Context.

   This procedure does not require any additional components to what OSCORE already provides, and it does not provide forward secrecy.

   The procedure defined in {{Section B.2 of RFC8613}} is used in 6TiSCH networks {{RFC7554}}{{RFC8180}} when handling failure events. That is, a node acting as Join Registrar/Coordinator (JRC) assists new devices, namely "pledges", to securely join the network as per the Constrained Join Protocol {{RFC9031}}. In particular, a pledge exchanges OSCORE-protected messages with the JRC, from which it obtains a short identifier, link-layer keying material and other configuration parameters. As per {{Section 8.3.3 of RFC9031}}, a JRC that experiences a failure event may likely lose information about joined nodes, including their assigned identifiers. Then, the reinitialized JRC can establish a new OSCORE Security Context with each pledge, through the procedure defined in {{Section B.2 of RFC8613}}.

* The two peers can run the OSCORE profile {{RFC9203}} of the Authentication and Authorization for Constrained Environments (ACE) Framework {{RFC9200}}.

  When a CoAP client uploads an Access Token to a CoAP server as an access credential, the two peers also exchange two nonces. Then, the two peers use the two nonces together with information provided by the ACE Authorization Server that issued the Access Token, in order to derive an OSCORE Security Context.

  This procedure does not provide forward secrecy.

* The two peers can run the EDHOC key exchange protocol based on Diffie-Hellman and defined in {{RFC9528}}, in order to establish a pseudo-random key in a mutually authenticated way.

   Then, the two peers can use the established pseudo-random key to derive external application keys. This allows the two peers to securely derive an OSCORE Master Secret and an OSCORE Master Salt, from which an OSCORE Security Context can be established.

   This procedure additionally provides forward secrecy.

   EDHOC also specifies an optional function, EDHOC\_KeyUpdate, to perform a key update in a more efficient way than re-running EDHOC. The two communicating peers call EDHOC\_KeyUpdate with equivalent input, which results in derivation of a new shared pseudo-random key. Usage of EDHOC\_KeyUpdate preserves forward secrecy.

   Note that EDHOC may be run standalone or as part of other workflows, such as when using the EDHOC and OSCORE profile of ACE {{I-D.ietf-ace-edhoc-oscore-profile}}.

* If one peer is acting as LwM2M Client and the other peer as LwM2M Server, according to the OMA Lightweight Machine to Machine Core specification {{LwM2M}}, then the LwM2M Client peer may take the initiative to bootstrap again with the LwM2M Bootstrap Server, and receive again an OSCORE Security Context. Alternatively, the LwM2M Server can instruct the LwM2M Client to initiate this procedure.

   If the OSCORE Security Context information on the LwM2M Bootstrap Server has been updated, the LwM2M Client will thus receive a fresh OSCORE Security Context to use with the LwM2M Server.

   In addition to that, the LwM2M Client, the LwM2M Server as well as the LwM2M Bootstrap server are required to use the procedure defined in {{Section B.2 of RFC8613}} and overviewed above, when they use a certain OSCORE Security Context for the first time {{LwM2M-Transport}}.

Manually updating the OSCORE Security Context at the two peers should be a last resort option, and it might often be not practical or feasible.

Even when any of the alternatives mentioned above is available, it is RECOMMENDED that two OSCORE peers update their Security Context by using the KUDOS procedure as defined in {{sec-rekeying-method}} of this document.

# Updated Protection of Responses with OSCORE # {#sec-updated-response-protection}

The protection of CoAP responses with OSCORE is updated, by adding the following text at the end of step 3 of {{Section 8.3 of RFC8613}}.

{:quote}
> If the server is using a different Security Context for the response compared to what was used to verify the request (e.g., due to an occurred key update), then the server MUST take the second alternative. That is, the server MUST include its Sender Sequence Number as Partial IV in the response and use it to build the AEAD nonce to protect the response.
>
> This prevents the server from using the same AEAD (key, nonce) pair for two responses, protected with different OSCORE Security Contexts.
>
> An exception is the procedure in {{Section B.2 of RFC8613}}, which is secure although not complying with the above. The reason is that, in that procedure, the server uses the new OSCORE Security Context only and solely to protect the outgoing response (response #1), and no other message is protected with that OSCORE Security Context. Other procedures where that holds would also remain secure.

# Key Update for OSCORE (KUDOS) # {#sec-rekeying-method}

This section defines KUDOS, a lightweight procedure that two OSCORE peers can use to update their keying material and establish a new OSCORE Security Context.

KUDOS relies on the OSCORE Option defined in {{RFC8613}} and extended as defined in {{ssec-oscore-option-extensions}}, as well as on the support function updateCtx() defined in {{ssec-update-function}}.

In order to run KUDOS, two peers perform a message exchange of OSCORE-protected CoAP messages. This message exchange between the two peers is defined in {{ssec-derive-ctx}}, with particular reference to the stateful FS mode providing forward secrecy. Building on the same message exchange, the possible use of the stateless no-FS mode is defined in {{no-fs-mode}}, as intended to peers that are not able to write in non-volatile memory. Two peers MUST run KUDOS in FS mode if they are both capable to.

The key update procedure has the following properties.

* KUDOS can be initiated by either peer. In particular, the CoAP client or the CoAP server may start KUDOS by sending the first rekeying message, by running KUDOS in the forward message flow {{ssec-derive-ctx}} or reverse message flow {{ssec-derive-ctx-server-init}}, respectively. A peer that supports KUDOS MUST support both the forward message flow and the reverse message flow.

* The new OSCORE Security Context enjoys forward secrecy, unless KUDOS is run in no-FS mode (see {{no-fs-mode}}).

* The same ID Context value used in the old OSCORE Security Context is preserved in the new Security Context. Furthermore, the ID Context value never changes throughout the KUDOS execution.

* KUDOS is robust against a peer rebooting, and it especially avoids the reuse of AEAD (nonce, key) pairs.

* KUDOS completes in one round trip by exchanging two CoAP messages. The two peers achieve mutual key confirmation in the following exchange, which is protected with the newly established OSCORE Security Context.

## Extensions to the OSCORE Option # {#ssec-oscore-option-extensions}

In order to support the message exchange for establishing a new OSCORE Security Context, this document extends the use of the OSCORE Option originally defined in {{RFC8613}} as follows.

* This document defines the usage of the eight least significant bit, called "Extension-1 Flag", in the first byte of the OSCORE Option containing the OSCORE flag bits. The registration of this flag bit in the "OSCORE Flag Bits" registry is specified in {{iana-cons-flag-bits}}.

   When the Extension-1 Flag is set to 1, the second byte of the OSCORE Option MUST include the OSCORE flag bits 8-15.

* This document defines the usage of the least significant bit "Nonce Flag", 'd', in the second byte of the OSCORE Option containing the OSCORE flag bits 8-15. This flag bit is specified in {{iana-cons-flag-bits}}.

   When it is set to 1, the compressed COSE object contains a field 'x' and a field 'nonce', to be used for the steps defined in {{ssec-derive-ctx}}. In particular, the 1 byte 'x' following 'kid context' (if any) encodes the size of the following field 'nonce', together with signaling bits that indicate the specific behavior to adopt during the KUDOS execution.

   Hereafter, a message is referred to as a "KUDOS (request/response) message", if and only if the second byte of flags is present and the 'd' bit is set to 1. If that is not the case, the message is referred to as a "non KUDOS (request/response) message".

   The encoding of 'x' is as follows:

   * The four least significant bits encode the 'nonce' size in bytes minus 1, namely 'm'.

   * The fifth least significant bit is the "No Forward Secrecy" 'p' bit. The sender peer indicates its wish to run KUDOS in FS mode or in no-FS mode, by setting the 'p' bit to 0 or 1, respectively. This makes KUDOS possible to run also for peers that cannot support the FS mode. At the same time, two peers MUST run KUDOS in FS mode if they are both capable to, as per {{ssec-derive-ctx}}. The execution of KUDOS in no-FS mode is defined in {{no-fs-mode}}.

   * The sixth least significant bit is the "Preserve Observations" 'b' bit. The sender peer indicates its wish to preserve ongoing observations beyond the KUDOS execution or not, by setting the 'b' bit to 1 or 0, respectively. The related processing is defined in {{preserving-observe}}.

   * The seventh least significant bit is the 'z' bit. When it is set to 1, the compressed COSE object contains a field 'y' and a field 'old\_nonce', to be used for the steps defined in {{ssec-derive-ctx}}. In particular, the 1 byte 'y' following 'nonce' encodes the size of the following field 'old\_nonce'. This bit SHALL only be set in the second KUDOS message and only if it is a CoAP request. For an example see the execution of KUDOS in the reverse message flow shown in {{fig-message-exchange-server-init}}.

   * The eight least significant bit is reserved for future use. This bit SHALL be set to zero when not in use. According to this specification, if this bit is set to 1: i) if the message is a request, it is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}; ii) if the message is a response, it is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.4 of RFC8613}} and the client SHALL discard the response as specified in item 8 of {{Section 8.4 of RFC8613}}.

   The encoding of 'y' is as follows:

   * The four least significant bits of the 'y' byte encode the 'old\_nonce' size in bytes minus 1, namely 'w'.

   * The fifth to seventh least significant bits SHALL be set to zero when not in use. According to this specification, if these bits are set to 1, the message is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}

   * The eight least significant bit is reserved for future use. This bit SHALL be set to zero when not in use. According to this specification, if this bit is set to 1, the message is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}.

* The second-to-eighth least significant bits in the second byte of the OSCORE Option containing the OSCORE flag bits are reserved for future use. These bits SHALL be set to zero when not in use. According to this specification, if any of these bits are set to 1, the message is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}.

{{fig-oscore-option}} shows extended OSCORE Option value, with the possible presence of 'nonce' and 'old\_nonce'.

~~~~~~~~~~~
 0 1 2 3 4 5 6 7  8   9   10  11  12  13  14  15 <----- n bytes ----->
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+
|1|0|0|h|k|  n  | 0 | 0 | 0 | 0 | 0 | 0 | 0 | d | Partial IV (if any) |
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+


 <- 1 byte -> <----- s bytes ------> <- 1 byte -> <--- m + 1 bytes --->
+------------+----------------------+------------+--------------------+
| s (if any) | kid context (if any) | x (if any) | nonce (if any)     |
+------------+----------------------+------------+--------------------+
                                   /              \____
                                  /                    |
                                 /   0 1 2 3 4 5 6 7   |
                                 |  +-+-+-+-+-+-+-+-+  |
                                 |  |0|z|b|p|   m   |  |
                                 |  +-+-+-+-+-+-+-+-+  |

    <- 1 byte -> <--- w + 1 bytes --->
   +------------+---------------------+------------------+
   | y (if any) | old_nonce (if any)  | kid (if any) ... |
   +------------+---------------------+------------------+
  /              \____
 /                    |
/   0 1 2 3 4 5 6 7   |
|  +-+-+-+-+-+-+-+-+  |
|  |0|0|0|0|   w   |  |
|  +-+-+-+-+-+-+-+-+  |
~~~~~~~~~~~
{: #fig-oscore-option title="The extended OSCORE Option value, with the possible presence of 'nonce' and 'old_nonce'" artwork-align="center"}

## Function for Security Context Update # {#ssec-update-function}

The updateCtx() function shown in {{function-update}} takes as input the three parameters X, N, and CTX\_IN. In particular, X and N are built from the 'x' and 'nonce' fields transported in the OSCORE Option value of the exchanged KUDOS messages (see {{ssec-oscore-option-extensions}}), while CTX\_IN is the OSCORE Security Context to update. The function returns a new OSCORE Security Context CTX\_OUT.

As a first step, the updateCtx() function builds the two CBOR byte strings X\_cbor and N\_cbor, with value the input parameter X and N, respectively. Then, it builds X\_N, as the byte concatenation of X\_cbor and N\_cbor.

After that, the updateCtx() function derives the new values of the Master Secret and Master Salt for CTX\_OUT. In particular, the new Master Secret is derived through a KUDOS-Expand() step, which takes as input the Master Secret value from the Security Context CTX\_IN, the literal string "key update", X\_N, and the length of the Master Secret. Instead, the new Master Salt takes N as value.

   The definition of KUDOS-Expand depends on the key derivation function used for OSCORE by the two peers, as specified in CTX\_IN.
either peer
   If the key derivation function is an HKDF Algorithm (see {{Section 3.1 of RFC8613}}), then KUDOS-Expand is mapped to HKDF-Expand {{RFC5869}}, as shown below. Also, the hash algorithm is the same one used by the HKDF Algorithm specified in CTX\_IN.

~~~~~~~~~~~
KUDOS-Expand(CTX_IN.MasterSecret, ExpandLabel, key_length) =
   HKDF-Expand(CTX_IN.MasterSecret, ExpandLabel, key_length)
~~~~~~~~~~~
{: artwork-align="left"}

   If a future specification updates {{RFC8613}} by admitting different key derivation functions than HKDF Algorithms (e.g., KMAC as based on the SHAKE128 or SHAKE256 hash functions), that specification has to update also the present document in order to define the mapping between such key derivation functions and KUDOS-Expand.

When an HKDF Algorithm is used, the derivation of new values follows the same approach used in TLS 1.3, which is also based on HKDF-Expand (see {{Section 7.1 of RFC8446}}) and used for computing new keying material in case of key update (see {{Section 4.6.3 of RFC8446}}).

After that, the new Master Secret and Master Salt parameters are used to derive a new Security Context CTX\_OUT as per {{Section 3.2 of RFC8613}}. Any other parameter required for the derivation takes the same value as in the Security Context CTX\_IN.

Note that the following holds for the newly derived CTX\_OUT:

* In its Sender Context, the Sender Sequence Number is initialized to 0 as per {{Section 3.2.2 of RFC8613}}.

* If the peer that has derived CTX\_OUT supports CoAP Observe {{RFC7641}}, the Notification Number used for the replay protection of Observe notifications (see {{Section 7.4.1 of RFC8613}}) is left as not initialized.

Finally, the updateCtx() function returns the newly derived Security Context CTX\_OUT.

Since the updateCtx() function also takes X as input, the derivation of CTX\_OUT also considers as input the information from the 'x' field transported in the OSCORE Option value of the exchanged KUDOS messages. In turn, this ensures that, if successfully completed, a KUDOS execution occurs as intended by the two peers.

~~~~~~~~~~~
function updateCtx(X, N, CTX_IN):

  // Output values
  CTX_OUT       // The new Security Context
  MSECRET_NEW   // The new Master Secret
  MSALT_NEW     // The new Master Salt

  // Create CBOR byte strings from X and N
  X_cbor = create_cbor_bstr(X)
  N_cbor = create_cbor_bstr(N)

  // Concatenate the CBOR-encoded X and N
  X_N = X_cbor | N_cbor

  // Determine the length in bytes of the current Master Secret
  oscore_key_length = length(CTX_IN.MasterSecret)

  // Define the label for the key update
  Label = "key update"

  // Create the new Master Secret using KUDOS-Expand-Label
  MSECRET_NEW = KUDOS_Expand_Label(CTX_IN.MasterSecret, Label,
                                   X_N, oscore_key_length)

  // Set the new Master Salt to N
  MSALT_NEW = N

  // Derive the new Security Context CTX_OUT, using
  // the new Master Secret, the new Master Salt,
  // and other parameters from CTX_IN
  CTX_OUT = derive_security_context(MSECRET_NEW, MSALT_NEW, CTX_IN)

  // Return the new Security Context
  return CTX_OUT


function KUDOS_Expand_Label(master_secret, Label, X_N, key_length):

  struct {
      uint16 length = key_length;
      opaque label<7..255> = "oscore " + Label;
      opaque context<0..255> = X_N;
  } ExpandLabel;

  return KUDOS_Expand(master_secret, ExpandLabel, key_length)
~~~~~~~~~~~
{: #function-update title="Functions for deriving a new OSCORE Security Context" artwork-align="left"}

## Key Update with Forward Secrecy # {#ssec-derive-ctx}

This section defines the actual KUDOS procedure performed by two peers to update their OSCORE keying material.

A peer can run KUDOS for active rekeying at any time, or for a variety of more compelling reasons. These include the (approaching) expiration of the OSCORE Security Context, approaching limits for the key usage {{I-D.ietf-core-oscore-key-limits}}, application policies, and imminent exhaustion of the OSCORE Sender Sequence Number space.

The expiration time of an OSCORE Security Context and the key usage limits are hard limits. Once reached them, a peer MUST stop using the keying material in the OSCORE Security Context for conventional communication with the other peer, and has to perform a rekeying before resuming secure communication.

Before starting KUDOS, the two peers share the OSCORE Security Context CTX\_OLD. Once successfully completed the KUDOS execution, the two peers agree on a newly established OSCORE Security Context CTX\_NEW.

In particular, CTX\_OLD is the most recent OSCORE Security Context that a peer has with a given ID Context or without ID Context, before initiating the KUDOS procedure or upon having received and successfully verified the first KUDOS message. In turn, CTX\_NEW is the most recent OSCORE Security Context that a peer has with a given ID Context or without ID Context, before sending the second KUDOS message or upon having received and successfully verified the second KUDOS message.

The following specifically defines how KUDOS is run in its stateful FS mode achieving forward secrecy. That is, in the OSCORE Option value of all the exchanged KUDOS messages, the "No Forward Secrecy" bit is set to 0.

In order to run KUDOS in FS mode, both peers have to be able to write in non-volatile memory. From the newly derived Security Context CTX\_NEW, the peers MUST store to non-volatile memory the immutable parts of the OSCORE Security Context as specified in {{Section 3.1 of RFC8613}}, with the possible exception of the Common IV, Sender Key, and Recipient Key that can be derived again when needed, as specified in {{Section 3.2.1 of RFC8613}}. If the peer is unable to write in non-volatile memory, the two peers have to run KUDOS in its stateless no-FS mode (see {{no-fs-mode}}).

### Nonces and X Bytes {#ssec-nonces-x-bytes}

When running KUDOS, each peer contributes by generating a nonce value N1 or N2, and providing it to the other peer. The size of the nonces N1 and N2 is application specific, and the use of 8 byte nonce values is RECOMMENDED. The nonces N1 and N2 MUST be random values, with the possible exception described later in {{key-material-handling}}. Note that a good amount of randomness is important for the nonce generation. {{RFC4086}} provides guidance on the generation of random values.

Furthermore, X1 and X2 are the value of the 'x' byte specified in the OSCORE Option of the first and second KUDOS message, respectively. The X1 and X2 values are calculated by the sender peer based on: the length of nonce N1 and N2, specified in the 'nonce' field of the OSCORE Option of the first and second KUDOS message, respectively; as well as on the specific settings the peer wishes to run KUDOS with. As defined in {{ssec-derive-ctx-client-init}}, these values are used by the peers to build the input N and X to the updateCtx() function, in order to derive a new OSCORE Security Context. As for any new OSCORE Security Context, the Sender Sequence Number and the Replay Window are re-initialized accordingly (see {{Section 3.2.2 of RFC8613}}).

After a peer has generated or received the value N1, and after a peer has calculated or received the value X1, it shall retain these in memory until it has received and processed the second KUDOS message.

### Handling of OSCORE Security Contexts {#ssec-context-handling}

The peer starting a KUDOS execution is denoted as initiator, while the other peer in the same session is denoted as responder.

The initiator completes the key update process when receiving the second KUDOS message and successfully verifying it with CTX\_NEW. The responder completes the key update process when sending the second KUDOS message, as protected with CTX\_NEW.

KUDOS may run with the initiator acting either as CoAP client or CoAP server. The former case is denoted as the "forward message flow" (see {{ssec-derive-ctx-client-init}}) and the latter as the "reverse message flow" (see {{ssec-derive-ctx-server-init}}).

The following properties hold for both the forward and reverse message flow.

* The initiator always offers the fresh value N1.
* The responder always offers the fresh value N2
* The responder is always the first one deriving CTX\_NEW.
* The initiator is always the first one achieving key confirmation, hence the first one able to safely discard CTX\_OLD.
* Both the initiator and the responder use and preserve the same respective OSCORE Sender ID and Recipient ID.
* If CTX\_OLD specifies an OSCORE ID Context, both peers use and preserve the same OSCORE ID Context.

Once a peer has successfully derived the new OSCORE Security Context CTX\_NEW, the following applies.

* The peer MUST use CTX\_NEW to protect outgoing non KUDOS messages, and MUST NOT use the originally shared OSCORE Security Context CTX\_OLD for protecting outgoing messages.

* The peer MUST delete any OSCORE Security Context CTX\_DEL older than CTX\_OLD, such that both CTX\_DEL and CTX\_OLD have the same ID\_CONTEXT or no ID Context.

  For instance, this can occur while using the forward message flow (see {{ssec-derive-ctx-client-init}}}), when the initiator has just received the second KUDOS message, and immediately starts KUDOS again as initiator before sending a non KUDOS message, thereby not providing the responder with key confirmation and not allowing it to safely discard CTX\_OLD.

* The peer MUST terminate all the ongoing observations {{RFC7641}} that it has with the other peer as protected with the old Security Context CTX\_OLD, unless the two peers have explicitly agreed otherwise as defined in {{preserving-observe}}.

  More specifically, if either or both peers indicate the wish to cancel their observations, those will be all cancelled following a successful KUDOS execution.

  Note that, even though a peer had no real reason to update its OSCORE keying material, running KUDOS can be intentionally exploited as a more efficient way to terminate all the ongoing observations with the other peer, compared to sending one cancellation request per observation (see {{Section 3.6 of RFC7641}}).

Once a peer has successfully decrypted and verified an incoming message protected with CTX\_NEW, that peer MUST discard the old Security Context CTX\_OLD.

### Handling of Messages

If a KUDOS message is a CoAP request, then it can target two different types of resources at the recipient CoAP server:

* The well-known KUDOS resource at /.well-known/kudos, or an alternative KUDOS resource with resource type "core.kudos" (see {{well-known-kudos-desc}} and {{rt-kudos}}). In such a case, no application processing is expected at the CoAP server, and the plain CoAP request composed before OSCORE protection should not include an application payload.

* A non-KUDOS resource, i.e., an actual application resource that a CoAP request can target in order to trigger application processing at the CoAP server. In such a case, the plain CoAP request composed before OSCORE protection may include an application payload, if admitted by the request method.

In either case, the link to the target resource can have the "osc" target attribute to indicate that the resource is only accessible using OSCORE (see {{Section 9 of RFC8613}}).

Similarly, any CoAP response can also be a KUDOS message. If the corresponding CoAP request has targeted a KUDOS resource, then the plain CoAP response composed before OSCORE encryption should not include an application payload. Otherwise, an application payload may be included.

Once a peer acting as initiator (responder) has sent (received) the first KUDOS message, that peer MUST NOT send a non KUDOS message to the other peer, until having aborted or successfully completed the key update process on its side.

In order to prevent two peers from unwittingly running two simultaneous executions of KUDOS, the following applies.

* When a peer P1 receives the first KUDOS message from a peer P2 in a KUDOS execution E1, the peer P1 MUST check whether it has a non completed KUDOS session E2 where P1 acts as initiator with P2.

  To this end, P1 may check whether it is currently acting as initiator in a KUDOS execution E2 different from E1, such that both sessions aim at updating the OSCORE Security Context CTX\_OLD shared with P2. The particular way to achieve this is implementation specific.

* If P1 finds such a session E2, then P1 MUST terminate the KUDOS execution E1, and MUST reply to the first KUDOS message received from P2 with a CoAP Reset message.

  Upon receiving the Reset message above, P2 terminates the KUDOS execution E2 where it acts as initiator.

### Avoiding In-Transit Requests During a Key Update

Before sending the first KUDOS message, the initiator MUST ensure that it has no outstanding interactions with the responder (see {{Section 4.7 of RFC7252}}), with the exception of ongoing observations {{RFC7641}} with the responder.

Before sending the second KUDOS message, the responder MUST ensure that it has no outstanding interactions with the initiator (see {{Section 4.7 of RFC7252}}), with the exception of ongoing observations {{RFC7641}} with the initiator.

If any such outstanding interactions are found, the initiator (responder) MUST NOT initiate (follow up with) the KUDOS execution, before either: i) having all those outstanding interactions cleared; or ii) freeing up the Token values used with those outstanding interactions, with the exception of ongoing observations with the other peer.

Later on, this prevents a non KUDOS response protected with the new Security Context CTX\_NEW from cryptographically matching with both the corresponding request also protected with CTX\_NEW and with an older request protected with CTX\_OLD, in case the two requests were protected using the same OSCORE Partial IV.

During an ongoing KUDOS execution, the peer acting as client MUST NOT send any non-KUDOS requests to the other peer. This could otherwise be possible, if the client is using a value of NSTART greater than 1 (see {{Section 4.7 of RFC7252}}).

### Forward Message Flow {#ssec-derive-ctx-client-init}

{{fig-message-exchange-client-init}} shows an example of KUDOS run in the forward message flow, i.e., with the client acting as KUDOS initiator.

In the example, 'Comb(a,b)' denotes the byte concatenation of two CBOR byte strings, where the first one has value 'a' and the second one has value 'b'. That is, Comb(a,b) = bstr .cbor a \| bstr .cbor b, where \| denotes byte concatenation.

~~~~~~~~~~~ aasvg
                     Client                  Server
                   (initiator)            (responder)
                        |                      |
Generate N1             |                      |
                        |                      |
CTX_1 = updateCtx(      |                      |
        X1,             |                      |
        N1,             |                      |
        CTX_OLD )       |                      |
                        |                      |
                        |      Request #1      |
Protect with CTX_1      +--------------------->| /.well-known/kudos
                        | OSCORE {             |
                        |  ...                 |
                        |  Partial IV: 0       |
                        |  ...                 |
                        |  d flag: 1           | CTX_1 = updateCtx(
                        |  x: X1               |         X1,
                        |  nonce: N1           |         N1,
                        |  ...                 |         CTX_OLD )
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_1
                        |  ...                 |
                        | }                    | Generate N2
                        |                      |
                        |                      | CTX_NEW = updateCtx(
                        |                      |           Comb(X1,X2),
                        |                      |           Comb(N1,N2),
                        |                      |           CTX_OLD )
                        |                      |
                        |      Response #1     |
                        |<---------------------+ Protect with CTX_NEW
                        | OSCORE {             |
                        |  ...                 |
CTX_NEW = updateCtx(    |  Partial IV: 0       |
          Comb(X1,X2),  |  ...                 |
          Comb(N1,N2),  |  d flag: 1           |
          CTX_OLD )     |  x: X2               |
                        |  nonce: N2           |
Verify with CTX_NEW     |  ...                 |
                        | }                    |
Discard CTX_OLD         | Encrypted Payload {  |
                        |  ...                 |
                        | }                    |
                        |                      |

The actual key update process ends here.
The two peers can use the new Security Context CTX_NEW.

                        |                      |
                        |      Request #2      |
Protect with CTX_NEW    +--------------------->| /temp
                        | OSCORE {             |
                        |  ...                 |
                        | }                    | Verify with CTX_NEW
                        | Encrypted Payload {  |
                        |  ...                 | Discard CTX_OLD
                        |  Application Payload |
                        | }                    |
                        |                      |
                        |      Response #2     |
                        |<---------------------+ Protect with CTX_NEW
                        | OSCORE {             |
                        |  ...                 |
Verify with CTX_NEW     | }                    |
                        | Encrypted Payload {  |
                        |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |
~~~~~~~~~~~
{: #fig-message-exchange-client-init title="Example of the KUDOS forward message flow." artwork-align="center"}

First, the client generates a value N1, and uses the nonce N = N1 and X = X1 together with the old Security Context CTX\_OLD, in order to derive a temporary Security Context CTX\_1.

Then, the client prepares a CoAP request targeting the well-known KUDOS resource (see {{well-known-kudos-desc}}) at "/.well-known/kudos". The client protects this CoAP request using CTX\_1 and sends it to the server. When the client protects this request using OSCORE, it MUST use 0 as the value of Partial IV. In particular, the request has the 'd' flag bit set to 1, and specifies X1 as 'x' and N1 as 'nonce' (see {{ssec-oscore-option-extensions}}). After that, the client deletes CTX\_1.

Upon receiving the OSCORE request, the server retrieves the value N1 from the 'nonce' field of the OSCORE Option and the value X1 from the 'x' byte of the OSCORE Option. Then, the server provides the updateCtx() function with the input N = N1, X = X1, and CTX\_OLD, in order to derive the temporary Security Context CTX\_1.

{{fig-kudos-x-n-example-mess-one}} shows an example of how the two peers compute X and N provided as input to the updateCtx() function, and how they compute X\_N within the updateCtx() function, when deriving CTX\_1 (see {{ssec-update-function}}).

~~~~~~~~~~~~~~~~~~~~~~~
   X1 and N1 expressed as raw values
   X1 = 0x07
   N1 = 0x018a278f7faab55a

   updateCtx() is called with
   X = 0x07
   N = 0x018a278f7faab55a

   In updateCtx(), X_cbor and N_cbor are built as CBOR byte strings
   X_cbor = 0x4107               (h'07')
   N_cbor = 0x48018a278f7faab55a (h'018a278f7faab55a')

   In updateCtx(), X_N is the byte concatenation of X_cbor and N_cbor
   X_N = 0x410748018a278f7faab55a
~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-kudos-x-n-example-mess-one title="Example of X, N, and X_N when processing the first KUDOS message"}

Then, the server verifies the request by using the Security Context CTX\_1.

After that, the server generates a value N2, and uses N = Comb(N1, N2) and X = Comb(X1, X2) together with CTX\_OLD, in order to derive the new Security Context CTX\_NEW.

An example of this nonce processing on the server with values for N1, X1, N2, and X2 is presented in {{fig-kudos-x-n-example-mess-two}}.

~~~~~~~~~~~~~~~~~~~~~~~
   X1, X2, N1, and N2 expressed as raw values
   X1 = 0x07
   X2 = 0x07
   N1 = 0x018a278f7faab55a
   N2 = 0x25a8991cd700ac01

   X1, X2, N1, and N2 as CBOR byte strings
   X1 = 0x4107 (h'07')
   X2 = 0x4107 (h'07')
   N1 = 0x48018a278f7faab55a (h'018a278f7faab55a')
   N2 = 0x4825a8991cd700ac01 (h'25a8991cd700ac01')

   updateCtx() is called with
   X = 0x41074107
   N = 0x48018a278f7faab55a4825a8991cd700ac01

   In updateCtx(), X_cbor and N_cbor are built as CBOR byte strings
   X_cbor = 0x4441074107 (h'41074107')
   N_cbor = 0x5248018a278f7faab55a4825a8991cd700ac01
            (h'48018a278f7faab55a4825a8991cd700ac01')

   In updateCtx(), X_N is the byte concatenation of X_cbor and N_cbor
   X_N = 0x44410741075248018a278f7faab55a4825a8991cd700ac01
~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-kudos-x-n-example-mess-two title="Example of X, N, and X_N when processing the second KUDOS message"}

Then, the server sends an OSCORE response to the client, protected with CTX\_NEW. In particular, the response has the 'd' flag bit set to 1 and specifies N2 as 'nonce'. Consistently with {{sec-updated-response-protection}}, the server includes its Sender Sequence Number as Partial IV in the response. After that, the server deletes CTX\_1.

Upon receiving the OSCORE response, the client retrieves the value N2 from the 'nonce' field of the OSCORE Option, and the value X2 from the 'x' byte of the OSCORE Option. Since the client has received a response to an OSCORE request that it made with the 'd' flag bit set to 1, the client provides the updateCtx() function with the input N = Comb(N1, N2), X = Comb(X1, X2), and CTX\_OLD, in order to derive CTX\_NEW. Finally, the client verifies the response by using CTX\_NEW and deletes CTX\_OLD.

From then on, the two peers can protect their message exchanges by using CTX\_NEW. As soon as the server successfully verifies an incoming message protected with CTX\_NEW, the server deletes CTX\_OLD.

In the example in {{fig-message-exchange-client-init}}, the client takes the initiative and sends a new OSCORE request protected with CTX\_NEW.

In case the server does not successfully verify the request, the same error handling specified in {{Section 8.2 of RFC8613}} applies. This does not result in deleting CTX\_NEW. If the server successfully verifies the request using CTX\_NEW, the server deletes CTX\_OLD and can reply with an OSCORE response protected with CTX\_NEW.

Note that the server achieves key confirmation only when receiving and successfully verifying a message from the client as protected with CTX\_NEW. If the server sends a non KUDOS request to the client protected with CTX\_NEW before then, and the server receives a 4.01 (Unauthorized) error response as reply, the server SHOULD delete CTX\_NEW and start a new KUDOS execution acting as CoAP client, i.e., as initiator in the forward message flow.

Also note that, if both peers reboot simultaneously, they will run the KUDOS forward message flow as defined in this section. That is, one of the two peers implementing a CoAP client will send KUDOS Request #1 in {{fig-message-exchange-client-init}}.

In case the KUDOS message Request #1 in Figure 3 targets a non-KUDOS resource and the application at the server requires freshness for the received requests, then the server does not deliver the request to the application even if the request has been succesfully verified, and the following KUDOS message (i.e., Response #1 in Figure 3) MUST be a 4.01 (Unauthorized) error response.

Upon receiving the 4.01 (Unauthorized) error response as the second KUDOS message Response #1, the client processes it like described above. After successfully completing the KUDOS execution, the client can send to the server a non-KUDOS request protected with CTX\_NEW (i.e., Request #2 in Figure 3). Presumably, this request targets the same resource targeted by the previous Request #1, as the same application request or a different one, if the application permits it.
Upon receiving, decrypting, and successfully verifying this request protected with CTX\_NEW, the server asserts the request as fresh, leveraging the recent establishment of CTX\_NEW.

In the example shown in {{fig-message-exchange-client-init}} and discussed in this section, the first KUDOS message is a request and the second one is a response, like typically expected when using the forward message flow. However, KUDOS is not constrained to this request/response model and a KUDOS execution can be performed with any combination of CoAP requests and responses. Related examples using the forward message flow are provided later:

* {{ssec-derive-ctx-client-init-requests-only}} presents an example where both KUDOS messages are CoAP requests.

* {{ssec-derive-ctx-client-init-unrelated}} presents an example where KUDOS Response #1 is a response to a different request from KUDOS Request #1.

  In such a case, if the client knows that KUDOS Response #2 is going to be sent as a response to a different request from KUDOS Request #1, then the client can use the No-Response CoAP Option {{RFC7967}} in KUDOS Request #1 without impairing the successful completion of KUDOS.

* {{ssec-derive-ctx-client-init-normal-resource}} presents an example where KUDOS Request #1 is sent to a non-KUDOS resource.

### Reverse Message Flow {#ssec-derive-ctx-server-init}

{{fig-message-exchange-server-init}} shows an example of KUDOS run in the reverse message flow, i.e., with the server acting as initiator.

The example uses the same notation 'Comb(a,b)' used in {{ssec-derive-ctx-client-init}}.

~~~~~~~~~~~ aasvg
                      Client                 Server
                   (responder)            (initiator)
                        |                      |
                        |      Request #1      |
Protect with CTX_OLD    +--------------------->| /temp
                        | OSCORE {             |
                        |  ...                 |
                        | }                    | Verify with CTX_OLD
                        | Encrypted Payload {  |
                        |  ...                 | Generate N1
                        |  Application Payload |
                        | }                    | CTX_1 = updateCtx(
                        |                      |         X1,
                        |                      |         N1,
                        |                      |         CTX_OLD )
                        |                      |
                        |      Response #1     |
                        |<---------------------+ Protect with CTX_1
                        | OSCORE {             |
                        |  ...                 |
CTX_1 = updateCtx(      |  Partial IV: 0       |
        X1,             |  ...                 |
        N1,             |  d flag: 1           |
        CTX_OLD )       |  x: X1               |
                        |  nonce: N1           |
Verify with CTX_1       |  ...                 |
                        | }                    |
Generate N2             | Encrypted Payload {  |
                        |  ...                 |
CTX_NEW = updateCtx(    | }                    |
          Comb(X1,X2),  |                      |
          Comb(N1,N2),  |                      |
          CTX_OLD )     |                      |
                        |                      |
                        |      Request #2      |
Protect with CTX_NEW    +--------------------->| /.well-known/kudos
                        | OSCORE {             |
                        |  ...                 |
                        |  d flag: 1           | CTX_NEW = updateCtx(
                        |  x: X2               |           Comb(X1,X2),
                        |  nonce: N2           |           Comb(N1,N2),
                        |  y: w                |           CTX_OLD )
                        |  old_nonce: N1       |
                        |  ...                 |
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_NEW
                        |  ...                 |
                        |  Application Payload |
                        | }                    | Discard CTX_OLD
                        |                      |

The actual key update process ends here.
The two peers can use the new Security Context CTX_NEW.

                        |                      |
                        |      Response #2     |
                        |<---------------------+ Protect with CTX_NEW
                        | OSCORE {             |
                        |  ...                 |
Verify with CTX_NEW     | }                    |
                        | Encrypted Payload {  |
Discard CTX_OLD         |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |
~~~~~~~~~~~
{: #fig-message-exchange-server-init title="Example of the KUDOS reverse message flow" artwork-align="center"}

First, the client sends a normal OSCORE request to the server, protected with the old Security Context CTX\_OLD and with the 'd' flag bit set to 0.

Upon receiving the OSCORE request and after having verified it with CTX\_OLD as usual, the server generates a value N1 and provides the updateCtx() function with the input N = N1, X = X1, and CTX\_OLD, in order to derive the temporary Security Context CTX\_1.

Then, the server sends an OSCORE response to the client, protected with CTX\_1. In particular, the response has the 'd' flag bit set to 1 and specifies N1 as 'nonce' (see {{ssec-oscore-option-extensions}}). After that, the server deletes CTX\_1. Consistently with {{sec-updated-response-protection}}, the server includes its Sender Sequence Number as Partial IV in the response. After that, the server deletes CTX\_1.

Upon receiving the OSCORE response, the client retrieves the value N1 from the 'nonce' field of the OSCORE Option and the value X1 from the 'x' byte of the OSCORE Option. Then, the client provides the updateCtx() function with the input N = N1, X = X1, and CTX\_OLD, in order to derive the temporary Security Context CTX\_1.

Then, the client verifies the response by using the Security Context CTX\_1.

After that, the client generates a value N2, and provides the updateCtx() function with the input N = Comb(N1, N2), X = Comb(X1, X2), and CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Then, the client sends an OSCORE request to the server, protected with CTX\_NEW. In particular, the request has the 'd' flag bit set to 1 and specifies N2 as 'nonce' and N1 as 'old\_nonce'. After that, the client deletes CTX\_1.

Upon receiving the OSCORE request, the server retrieves the values N1 from the 'old\_nonce' field of the OSCORE Option, the value N2 from the 'nonce' field of the OSCORE Option, and the value X2 from the 'x' byte of the OSCORE Option. Then, the server verifies that: i) the value N1 is identical to the value N1 specified in a previous OSCORE response with the 'd' flag bit set to 1; and ii) the value N1 \| N2 has not been received before in an OSCORE request with the 'd' flag bit set to 1.

If the verification succeeds, the server provides the updateCtx() function with the input N = Comb(N1, N2), X = Comb(X1, X2), and CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Finally, the server verifies the request by using CTX\_NEW and deletes CTX\_OLD.

From then on, the two peers can protect their message exchanges by using CTX\_NEW. In particular, as shown in the example in {{fig-message-exchange-server-init}}, the server can send an OSCORE response protected with CTX\_NEW.

In case the client does not successfully verify the response, the same error handling specified in {{Section 8.4 of RFC8613}} applies. This does not result in deleting CTX\_NEW. If the client successfully verifies the response using CTX\_NEW, the client deletes CTX\_OLD. Note that, if the verification of the response fails, the client may want to send again the normal OSCORE request to the server it initially sent (to /temp in the example above), in order to ensure the retrieval of the resource representation.

More generally, as soon as the client successfully verifies an incoming message protected with CTX\_NEW, the client deletes CTX\_OLD.

Note that the client achieves key confirmation only when receiving and successfully verifying a message from the server as protected with CTX\_NEW. If the client sends a non KUDOS request to the server protected with CTX\_NEW before then, and the client receives a 4.01 (Unauthorized) error response as reply, the client SHOULD delete CTX\_NEW and start a new KUDOS execution acting again as CoAP client, i.e., as initiator in the forward message flow (see {{ssec-derive-ctx-client-init}}).

It might be the case that the server is only a CoAP server (i.e., it does not implement a CoAP client) and, at the same time, it becomes unable to safely decrypt further incoming messages from the client. For example, this occurs when the server reaches key usage limits for its Recipient Key in the OSCORE Security Context shared with the client (see {{I-D.ietf-core-oscore-key-limits}}). When this happens, the server cannot decrypt Request #1. Consequently, the server replies to the client with an unprotected 4.01 (Unauthorized) response, and is therefore practically unable to execute KUDOS with the client in the reverse message flow. In such a case, the only chance for the server to perform a key update with the client by means of KUDOS relies on the client starting a KUDOS execution using the forward message flow (see {{ssec-derive-ctx-client-init}}).

## Avoiding Deadlocks

This section defines how to avoid a deadlock in different scenarios.

### Scenario 1

In this scenario, an execution of KUDOS fails at PEER\_1 acting as initiator, but successfully completes at PEER\_2 acting as responder. After that, PEER\_1 still stores CTX\_OLD, while PEER\_2 stores CTX\_OLD and the just derived CTX\_NEW.

Then, PEER\_1 starts a new KUDOS execution acting again as initiator, by sending the first KUDOS message as a CoAP request. This is protected with a temporary Security Context CTX\_1, which is newly derived from the retained CTX\_OLD, and from the new values X1 and N1 exchanged in the present KUDOS execution.

Upon receiving the first KUDOS message, PEER\_2, acting again as responder, proceeds as follows.

1. PEER\_2 attempts to verify the first KUDOS message by using a temporary Security Context CTX\_1'. This is derived from the Security Context CTX\_NEW established during the latest successfully completed KUDOS execution.

2. The message verification inevitably fails. If PEER\_2 is acting as CoAP server, it MUST NOT reply with an unprotected 4.01 (Unauthorized) CoAP response yet.

3. PEER\_2 MUST attempt to verify the first KUDOS message by using a temporary Security Context CTX\_1. This is newly derived from the Security Context CTX\_OLD retained after the latest successfully completed KUDOS execution, and from the values X1 and N1 exchanged in the present KUDOS execution.

   If the message verification fails, PEER\_2: i) retains CTX\_OLD and CTX\_NEW from the latest successfully completed KUDOS execution; ii) if acting as CoAP server, replies with an unprotected 4.01 (Unauthorized) CoAP response.

   If the message verification succeeds, PEER\_2: i) retains CTX\_OLD from the latest successfully completed KUDOS execution; ii) replaces CTX\_NEW from the latest successfully completed KUDOS execution with a new Security Context CTX\_NEW', derived from CTX\_OLD and from the values X1, X2, N1, and N2 exchanged in the present KUDOS execution; iii) replies with the second KUDOS message, which is protected with the just derived CTX\_NEW'.

### Scenario 2

In this scenario, an execution of KUDOS fails at PEER\_1 acting as initiator, but successfully completes at PEER\_2 acting as responder. After that, PEER\_1 still stores CTX\_OLD, while PEER\_2 stores CTX\_OLD and the just derived CTX\_NEW.

Then, PEER\_2 starts a new KUDOS execution, this time acting as initiator, by sending the first KUDOS message as a CoAP request. This is protected with a temporary Security Context CTX\_1, which is newly derived from CTX\_NEW established during the latest successfully completed KUDOS execution, as well as from the new values X1 and N1 exchanged in the present KUDOS execution.

Upon receiving the first KUDOS message, PEER\_1, this time acting as responder, proceeds as follows.

1. PEER\_1 attempts to verify the first KUDOS message by using a temporary Security Context CTX\_1', which is derived from the retained Security Context CTX\_OLD and from the values X1 and N1 exchanged in the present KUDOS execution.

2. The message verification inevitably fails. If PEER\_1 is acting as CoAP server, it replies with an unprotected 4.01 (Unauthorized) CoAP response.

3. If PEER\_2 does not receive the second KUDOS message for a pre-defined amount of time, or if it receives a 4.01 (Unauthorized) CoAP response when acting as CoAP client, then PEER\_2 can start a new KUDOS execution for a maximum, pre-defined number of times.

   In this case, PEER\_2 sends a new first KUDOS message protected with a temporary Security Context CTX\_1', which is derived from the retained CTX\_OLD, as well as from the new values X1 and N1 exchanged in the present KUDOS execution.

   During this time, PEER\_2 does not delete CTX\_NEW established during the latest successfully completed KUDOS execution, and does not delete CTX\_OLD unless it successfully verifies an incoming message protected with CTX\_NEW.

4. Upon receiving such a new, first KUDOS message, PEER\_1 verifies it by using the temporary Security Context CTX\_1', which is derived from the Security Context CTX\_OLD, and from the values X1 and N1 exchanged in the present KUDOS execution.

   If the message verification succeeds, PEER\_1 derives an OSCORE Security Context CTX\_NEW' from CTX\_OLD and from the values X1, X2, N1, and N2 exchanged in the present KUDOS execution. Then, it replies with the second KUDOS message, which is protected with the latest, just derived CTX\_NEW'.

5. Upon receiving such second KUDOS message, PEER\_2 derives CTX\_NEW' from the retained CTX\_OLD and from the values X1, X2, N1, and N2 exchanged in the present KUDOS execution. Then, PEER\_2 attempts to verify the KUDOS message using the just derived CTX\_NEW'.

   If the message verification succeeds, PEER\_2 deletes the retained CTX\_OLD as well as the retained CTX\_NEW established during the immediately previously, successfully completed KUDOS execution.

### Scenario 3

When KUDOS is run in the reverse message flow (see {{ssec-derive-ctx-server-init}}), the two peers risk to run into a deadlock, if all the following conditions hold.

* The client is a client-only device, i.e., it does not act as CoAP server and thus does not listen for incoming requests.

* The server needs to execute KUDOS, which, due to the previous point, can only be performed in its reverse message flow. That is, the server has to wait for an incoming non KUDOS request, in order to initiate KUDOS by replying with the first KUDOS message as a response.

* The client sends only Non-confirmable CoAP requests to the server and does not expect responses sent back as reply, hence freeing up a request's Token value once the request is sent.

In such a case, in order to avoid experiencing a deadlock situation where the server needs to execute KUDOS but cannot practically initiate it, a client-only device that supports KUDOS SHOULD intersperse Non-confirmable requests it sends to that server with confirmable requests.

## Key Update with or without Forward Secrecy {#no-fs-mode}

The FS mode of the KUDOS procedure defined in {{ssec-derive-ctx}} ensures forward secrecy of the OSCORE keying material. However, it requires peers executing KUDOS to preserve their state (e.g., across a device reboot), by writing information such as data from the newly derived OSCORE Security Context CTX\_NEW in non-volatile memory.

This can be problematic for devices that cannot dynamically write information to non-volatile memory. For example, some devices may support only a single writing in persistent memory when initial keying material is provided (e.g., at manufacturing or commissioning time), but no further writing after that. Therefore, these devices cannot perform a stateful key update procedure, and thus are not capable to run KUDOS in FS mode to achieve forward secrecy.

In order to address these limitations, KUDOS can be run in its stateless no-FS mode, as defined in the following. This allows two peers to achieve the same results as when running KUDOS in FS mode (see {{ssec-derive-ctx}}), with the difference that no forward secrecy is achieved and no state information is required to be dynamically written in non-volatile memory.

From a practical point of view, the two modes differ as to what exact OSCORE Master Secret and Master Salt are used as part of the OSCORE Security Context CTX\_OLD provided as input to the updateCtx() function (see {{ssec-update-function}}).

If either or both peers are not able to write in non-volatile memory the OSCORE Master Secret and OSCORE Master Salt from the newly derived Security Context CTX\_NEW, then the two peers have to run KUDOS in no-FS mode.

### Handling and Use of Keying Material {#key-material-handling}

In the following, a device is denoted as "CAPABLE" if it is able to store information in non-volatile memory (e.g., on disk), beyond a one-time-only writing occurring at manufacturing or (re-)commissioning time. If that is not the case, the device is denoted as "non-CAPABLE".

The following terms are used to refer to OSCORE keying material.

* Bootstrap Master Secret and Bootstrap Master Salt. If pre-provisioned during manufacturing or (re-)commissioning, these OSCORE Master Secret and Master Salt are initially stored on disk and are never going to be overwritten by the device.

* Latest Master Secret and Latest Master Salt. These OSCORE Master Secret and Master Salt can be dynamically updated by the device. In case of reboot, they are lost unless they have been stored on disk.

Note that:

* A peer running KUDOS can have none of the pairs above associated with another peer, only one, or both.

* A peer that has neither of the pairs above associated with another peer, cannot run KUDOS in any mode with that other peer.

* A peer that has only one of the pairs above associated with another peer can attempt to run KUDOS with that other peer, but the procedure might fail depending on the other peer's capabilities. In particular:

   - In order to run KUDOS in FS mode, a peer must be a CAPABLE device. It follows that two peers have to both be CAPABLE devices in order to be able to run KUDOS in FS mode with one another.

   - In order to run KUDOS in no-FS mode, a peer must have Bootstrap Master Secret and Bootstrap Master Salt available as stored on disk.

* A peer that is a non-CAPABLE device MUST support the no-FS mode.

* A peer that is a CAPABLE device MUST support the FS mode and the no-FS mode.

* As an exception to the nonces being generated as random values (see {{ssec-nonces-x-bytes}}), a peer that is a CAPABLE device MAY use a value obtained from a monotonically incremented counter as nonce N1 or N2. This has privacy implications, which are described in {{sec-cons}}. In such a case, the peer MUST enforce measures to ensure freshness of the nonce values. For example, the peer can use the same procedure described in {{Section B.1.1 of RFC8613}} for handling the OSCORE Sender Sequence Number values. These measures require to regularly store the used counter values in non-volatile memory, which makes non-CAPABLE devices unable to safely use counter values as nonce values.

As a general rule, once successfully generated a new OSCORE Security Context CTX (e.g., CTX is the CTX\_NEW resulting from a KUDOS execution, or it has been established through the EDHOC protocol {{RFC9528}}), a peer considers the Master Secret and Master Salt of CTX as Latest Master Secret and Latest Master Salt. After that:

* If the peer is a CAPABLE device, it MUST store Latest Master Secret and Latest Master Salt on disk, with the exception of possible temporary OSCORE Security Contexts used during a key update procedure, such as CTX\_1 used during the KUDOS execution. That is, the OSCORE Master Secret and Master Salt from such temporary Security Contexts are not stored on disk.

* The peer MUST store Latest Master Secret and Latest Master Salt in volatile memory, thus making them available to OSCORE message processing and possible key update procedures.

#### Actions after Device Reboot

Building on the above, after having experienced a reboot, a peer A checks whether it has stored on disk a pair P1 = (Latest Master Secret, Latest Master Salt) associated with any another peer B.

* If a pair P1 is found, the peer A performs the following actions.

    * The peer A loads the Latest Master Secret and Latest Master Salt to volatile memory, and uses them to derive an OSCORE Security Context CTX\_OLD.

    * The peer A runs KUDOS with the other peer B, acting as initiator. If the peer A is a CAPABLE device, it stores on disk the Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

* If a pair P1 is not found, the peer A checks whether it has stored on disk a pair P2 = (Bootstrap Master Secret, Bootstrap Master Salt) associated with the other peer B.

    * If a pair P2 is found, the peer A performs the following actions.

        * The peer A loads the Bootstrap Master Secret and Bootstrap Master Salt to volatile memory, and uses them to derive an OSCORE Security Context CTX\_OLD.

        * If the peer A is a CAPABLE device, it stores on disk Bootstrap Master Secret and Bootstrap Master Salt as Latest Master Secret and Latest Master Salt, respectively. This supports the situation where A is a CAPABLE device and has never run KUDOS with the other peer B before.

        * The peer A runs KUDOS with the other peer B, acting as initiator. If the peer A is a CAPABLE device, it stores on disk the Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

    * If a pair P2 is not found, the peer A has to use alternative ways to establish a first OSCORE Security Context CTX\_NEW with the other peer B, e.g., by running the EDHOC protocol. After that, if A is a CAPABLE device, it stores on disk the OSCORE Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

Following a state loss (e.g., due to a reboot), a device MUST first complete a successful KUDOS execution (with either of the workflows) before exchanging OSCORE-protected application data with another peer. An exception is a CAPABLE device implementing a functionality for safely reusing old keying material, such as the one defined in {{Section B.1 of RFC8613}}.

### Selection of KUDOS Mode {#no-fs-signaling}

During a KUDOS execution, the two peers agree on whether to perform the key update procedure in FS mode or no-FS mode, by leveraging the "No Forward Secrecy" bit, 'p', in the 'x' byte of the OSCORE Option value of the KUDOS messages (see {{ssec-oscore-option-extensions}}). The 'p' bit practically determines what OSCORE Security Context to use as CTX\_OLD during the KUDOS execution, consistently with the indicated mode.

* If the 'p' bit is set to 0 (FS mode), the updateCtx() function used to derive CTX\_1 or CTX\_NEW considers as input CTX\_OLD the current OSCORE Security Context shared with the other peer as is. In particular, CTX\_OLD includes Latest Master Secret as OSCORE Master Secret and Latest Master Salt as OSCORE Master Salt.

* If the 'p' bit is set to 1 (no-FS mode), the updateCtx() function used to derive CTX\_1 or CTX\_NEW considers as input CTX\_OLD the current OSCORE Security Context shared with the other peer, with the following difference: Bootstrap Master Secret is used as OSCORE Master Secret and Bootstrap Master Salt is used as OSCORE Master Salt. That is, every execution of KUDOS in no-FS mode between these two peers considers the same pair (Master Secret, Master Salt) in the OSCORE Security Context CTX\_OLD provided as input to the updateCtx() function, hence the impossibility to achieve forward secrecy.

A peer determines to run KUDOS either in FS or no-FS mode with another peer as follows.

* If a peer A is a non-CAPABLE device, it MUST run KUDOS only in no-FS mode. That is, when sending a KUDOS message, it MUST set to 1 the 'p' bit of the 'x' byte in the OSCORE Option value.

* If a peer A is a CAPABLE device, it SHOULD run KUDOS only in FS mode. That is, when sending a KUDOS message, it SHOULD set to 0 the 'p' bit of the 'x' byte in the OSCORE Option value. An exception applies in the following cases.

   * The peer A is running KUDOS with another peer B, which A has learned to be a non-CAPABLE device (and hence not able to run KUDOS in FS mode).

      Note that, if the peer A is a CAPABLE device, it is able to store such information about the other peer B on disk and it MUST do so. From then on, the peer A will perform every execution of KUDOS with the peer B in no-FS mode, including after a possible reboot.

   * The peer A is acting as responder and running KUDOS with another peer B without knowing its capabilities, and A receives a KUDOS message where the 'p' bit of the 'x' byte in the OSCORE Option value is set to 1.

* If a peer A is a CAPABLE device and has learned that another peer B is also a CAPABLE device (and hence able to run KUDOS in FS mode), then the peer A MUST NOT run KUDOS with the peer B in no-FS mode. This also means that, if the peer A acts as responder when running KUDOS with the peer B, the peer A MUST terminate the KUDOS execution if it receives a KUDOS message from the peer B where the 'p' bit of the 'x' byte in the OSCORE Option value is set to 1.

   Note that, if the peer A is a CAPABLE device, it is able to store such information about the other peer B on disk and it MUST do so. This ensures that the peer A will perform every execution of KUDOS with the peer B in FS mode. In turn, this prevents a possible downgrading attack, aimed at making A believe that B is a non-CAPABLE device, and thus to run KUDOS in no-FS mode although the FS mode can actually be used by both peers.

Within the limitations above, two peers running KUDOS generate the new OSCORE Security Context CTX\_NEW according to the mode indicated per the bit 'p' set by the responder in the second KUDOS message.

If, after having received the first KUDOS message, the responder can continue performing KUDOS, the bit 'p' in the reply message has the same value as in the bit 'p' set by the initiator, unless such latter value is 0 and the responder is a non-CAPABLE device. More specifically:

* If both peers are CAPABLE devices, they will run KUDOS in FS mode. That is, both initiator and responder sets the 'p' bit to 0 in the respective sent KUDOS message.

* If both peers are non-CAPABLE devices or only the peer acting as initiator is a non-CAPABLE device, they will run KUDOS in no-FS mode. That is, both initiator and responder sets the 'p' bit to 1 in the respective sent KUDOS message.

* If only the peer acting as initiator is a CAPABLE device and it has knowledge of the other peer being a non-CAPABLE device, they will run KUDOS in no-FS mode. That is, both initiator and responder sets the 'p' bit to 1 in the respective sent KUDOS message.

* If only the peer acting as initiator is a CAPABLE device and it has no knowledge of the other peer being a non-CAPABLE device, they will not run KUDOS in FS mode and will rather set to ground for possibly retrying in no-FS mode. In particular, the initiator sets the 'p' bit of its sent KUDOS message to 0. Then:

   * If the responder is a server, it MUST consider the KUDOS execution unsuccessful and MUST reply with a 5.03 (Service Unavailable) error response. The response MUST be protected with the newly derived OSCORE Security Context CTX\_NEW. The diagnostic payload MAY provide additional information. This response is a KUDOS message, and it MUST have the 'd' bit and the 'p' bit set to 1.

      When receiving the error response, the initiator learns that the responder is a non-CAPABLE device (and hence not able to run KUDOS in FS mode), since the 'p' bit in the error response is set to 1, while the 'p' bit in the corresponding request was set to 0. Hence, the initiator MUST consider the KUDOS execution unsuccessful, and MAY try running KUDOS again. If it does so, the initiator MUST set the 'p' bit to 1, when sending a new request as first KUDOS message.

   * If the responder is a client, it MUST consider the KUDOS execution unsuccessful and MUST send to the initiator the second KUDOS message as a new request, which MUST be protected with the newly derived OSCORE Security Context CTX\_NEW. In the newly sent request, the 'p' bit MUST be set to 1.

      When receiving the new request above, the initiator learns that the responder is a non-CAPABLE device (and hence not able to run KUDOS in FS mode), since the 'p' bit in the request is set to 1, while the 'p' bit in the response previously sent as first KUDOS message was set to 0. Also, the initiator SHOULD NOT send any response to such a request, and the responder SHOULD NOT expect any such response.

   In either case, both KUDOS peers delete the OSCORE Security Contexts CTX\_1 and CTX\_NEW. Also, both peers MUST retain CTX\_OLD for use during the next KUDOS execution in the no-FS mode. This is in contrast with the typical behavior where CTX\_OLD is deleted upon reception of a message protected with CTX\_NEW.

## Preserving Observations Across Key Updates # {#preserving-observe}

As defined in {{ssec-derive-ctx}}, once a peer has completed the KUDOS execution and successfully derived the new OSCORE Security Context CTX\_NEW, that peer normally terminates all the ongoing observations it has with the other peer {{RFC7641}}, as protected with the old OSCORE Security Context CTX\_OLD.

This section describes a method that the two peers can use to safely preserve the ongoing observations that they have with one another, beyond the completion of a KUDOS execution. In particular, this method ensures that an Observe notification can never successfully cryptographically match against the Observe requests of two different observations, e.g., against an Observe request protected with CTX\_OLD and an Observe request protected with CTX\_NEW.

The actual preservation of ongoing observations has to be agreed by the two peers at each execution of KUDOS that they run with one another, as defined in {{preserving-observe-management}}. If, at the end of a KUDOS execution, the two peers have not agreed on that, they MUST terminate the ongoing observations that they have with one another, just as defined in {{ssec-context-handling}}.

### Management of Observations {#preserving-observe-management}

As per {{Section 3.1 of RFC7641}}, a client can register its interest in observing a resource at a server, by sending a registration request including the Observe Option with value 0.

If the server registers the observation as ongoing, the server sends back a successful response also including the Observe Option, hence confirming that an entry has been successfully added for that client.

If the client receives back the successful response above from the server, then the client also registers the observation as ongoing.

In case the client can ever consider to preserve ongoing observations beyond a key update as defined below, then the client MUST NOT simply forget about an ongoing observation if not interested in it anymore. Instead, the client MUST send an explicit cancellation request to the server, i.e., a request including the Observe Option with value 1 (see {{Section 3.6 of RFC7641}}). After sending this cancellation request, if the client does not receive back a response confirming that the observation has been terminated, the client MUST NOT consider the observation terminated. The client MAY try again to terminate the observation by sending a new cancellation request.

In case a peer A performs a KUDOS execution with another peer B, and A has ongoing observations with B that it is interested to preserve beyond the key update, then A can explicitly indicate its interest to do so. To this end, the peer A sets to 1 the bit "Preserve Observations", 'b', in the 'x' byte of the OSCORE Option value (see {{ssec-oscore-option-extensions}}), in the KUDOS message it sends to the other peer B.

If a peer acting as responder receives the first KUDOS message with the bit 'b' set to 0, then the peer MUST set to 0 the bit 'b' in the KUDOS message it sends as follow-up, regardless of its wish to preserve ongoing observations with the other peer.

If a peer acting as initiator has sent the first KUDOS message with the bit 'b' set to 0, the peer MUST ignore the bit 'b' in the follow-up KUDOS message that it receives from the other peer.

After successfully completing the KUDOS execution (i.e., after having successfully derived the new OSCORE Security Context CTX\_NEW), both peers have expressed their interest in preserving their common ongoing observations if and only if the bit 'b' was set to 1 in both the exchanged KUDOS messages. In such a case, each peer X performs the following actions.

1. The peer X considers all the still ongoing observations that it has with the other peer, such that X acts as client in those observations. If there are no such observations, the peer X takes no further actions. Otherwise, it moves to step 2.

2. The peer X considers all the OSCORE Partial IV values used in the Observe registration request associated with any of the still ongoing observations determined at step 1.

3. The peer X determines the value PIV\* as the highest OSCORE Partial IV value among those considered at step 2.

4. In the Sender Context of the OSCORE Security Context shared with the other peer, the peer X sets its own Sender Sequence Number to (PIV\* + 1), rather than to 0.

As a result, each peer X will "jump" beyond the OSCORE Partial IV (PIV) values that are occupied and in use for ongoing observations with the other peer where X acts as client.

Note that, each time it runs KUDOS, a peer must determine if it wishes to preserve ongoing observations with the other peer or not, before sending its KUDOS message.

To this end, the peer should also assess the new value that PIV\* would take after a successful completion of KUDOS, in case ongoing observations with the other peer are going to be preserved. If the peer considers such a new value of PIV\* to be too close to or equal to the maximum possible value admitted for the OSCORE Partial IV, then the peer may choose to run KUDOS with no intention to preserve its ongoing observations with the other peer, in order to "start over" from a fresh, entirely unused PIV space.

Application policies can further influence whether attempting to preserve observations beyond a key update is appropriate or not.

## Retention Policies # {#ssec-retention}

Applications MAY define policies that allow a peer to temporarily keep the old Security Context CTX\_OLD beyond having established the new Security Context CTX\_NEW and having achieved key confirmation, rather than simply overwriting CTX\_OLD with CTX\_NEW. This allows the peer to decrypt late, still on-the-fly incoming messages protected with CTX\_OLD.

When enforcing such policies, the following applies.

* Outgoing non KUDOS messages MUST be protected by using only CTX\_NEW.

* Incoming non KUDOS messages MUST first be attempted to decrypt by using CTX\_NEW. If decryption fails, a second attempt can use CTX\_OLD.

* When an amount of time defined by the policy has elapsed since the establishment of CTX\_NEW, the peer deletes CTX\_OLD.

A peer MUST NOT retain CTX\_OLD beyond the establishment of CTX\_NEW and the achievement of key confirmation, if any of the following conditions holds: CTX\_OLD is expired; limits set for safe key usage have been reached {{I-D.ietf-core-oscore-key-limits}}, for the Recipient Key of the Recipient Context of CTX\_OLD.

## Discussion # {#ssec-discussion}

KUDOS is intended to deprecate and replace the procedure defined in {{Section B.2 of RFC8613}}, as fundamentally achieving the same goal, while displaying a number of improvements and advantages.

In particular, it is especially convenient for the handling of failure events concerning the JRC node in 6TiSCH networks (see {{sec-current-methods}}). That is, among its intrinsic advantages compared to the procedure defined in {{Section B.2 of RFC8613}}, KUDOS preserves the same ID Context value, when establishing a new OSCORE Security Context.

Since the JRC uses ID Context values as identifiers of network nodes, namely "pledge identifiers", the above implies that the JRC does not have to perform anymore a mapping between a new, different ID Context value and a certain pledge identifier (see {{Section 8.3.3 of RFC9031}}). It follows that pledge identifiers can remain constant once assigned, and thus ID Context values used as pledge identifiers can be employed in the long-term as originally intended.

### KUDOS Interleaved with Other Message Exchanges

During a KUDOS execution, a peer that is a CoAP Client must be ready to receive CoAP responses that are not KUDOS messages and that are protected with a different OSCORE Security Context than the one that was used to protect the corresponding request.

This can happen, for instance, when a CoAP client sends a request and, shortly after that, it executes KUDOS. In such a case, the CoAP request is protected with CTX\_OLD, while the CoAP response from the server is protected with CTX\_NEW. Another case is when incoming responses are Observe notifications protected with CTX\_NEW, while the corresponding request from the CoAP client that started the observation was protected with CTX\_OLD.

Another case is when running KUDOS in the reverse message flow, if the client uses NSTART > 1 and one of its requests triggers a KUDOS execution, i.e., the server replies with the first KUDOS message by acting as responder. The other requests would be latest served by the server after KUDOS has been completed.

### Communication Overhead

Each of the two KUDOS messages results in communication overhead. This is determined by the following, additional information conveyed in the OSCORE Option (see {{ssec-oscore-option-extensions}}).

* The second byte of the OSCORE Option value.

* The byte 'x' of the OSCORE Option value.

* The nonce conveyed in the 'nonce' field of the OSCORE option. Its size ranges from 1 to 16 bytes as indicated in the 'x' byte, and is typically of 8 bytes.

* The byte 'y' of the OSCORE Option value, if present.

* The nonce conveyed in the 'old_nonce' field of the OSCORE Option value, if present. When present, its size ranges from 1 to 16 bytes as indicated in the ’y’ byte, and is typically of 8 bytes.

* The 'Partial IV' parameter of the OSCORE Option value, in a CoAP response message that is a KUDOS message.

  This takes into account the fact that OSCORE-protected CoAP response messages normally do not include the 'Partial IV' parameter, but they have to when they are KUDOS messages (see Section 3).

* The first byte of the OSCORE Option value (i.e., the first OSCORE flag byte), in a CoAP response message that is a KUDOS message.

  This takes into account the fact that OSCORE-protected CoAP response messages normally convey an OSCORE Option that only consists of the all zero (0x00) flag byte. In turn, this results in the OSCORE Option being encoded as with empty value (see {{Section 2 of RFC8613}}.

* The possible presence of the 1-byte Option Length (extended) field in the OSCORE Option (see {{Section 3.1 of RFC7252}}). This is the case where the length of the OSCORE Option value is between 13 and 255 bytes (see {{Section 2 of RFC8613}}).

The results shown below are the minimum, typical, and maximum communication overhead introduced by KUDOS, when considering a nonce with size 1, 8, and 16 bytes. {{table-overhead-forward}} and {{table-overhead-reverse}} refer to the KUDOS forward message flow and reverse message flow, respectively. All the indicated values are in bytes.

In particular, the shown results build on the following assumptions.

* Both messages of the same KUDOS execution use nonces of the same size, and do not include the 'kid context' parameter in the OSCORE Option value.

* When included in the OSCORE Option value, the 'Partial IV' parameter has size 1 byte.

* CoAP request messages include the 'kid' parameter with size 1 byte in the OSCORE Option value.

* CoAP response messages do not include the 'kid' parameter in the OSCORE Option value.

| Nonce size | First KUDOS message | Second KUDOS message | Total |
| 1          | 3                   | 5                    | 8     |
| 8          | 11                  | 12                   | 23    |
| 16         | 19                  | 21                   | 40    |
{: #table-overhead-forward title="Communication overhead (forward message flow)" align="center"}


| Nonce size | First KUDOS message | Second KUDOS message | Total |
| 1          | 5                   | 5                    | 10    |
| 8          | 12                  | 20                   | 32    |
| 16         | 21                  | 36                   | 57    |
{: #table-overhead-reverse title="Communication overhead (reverse message flow)" align="center"}

### Well-Known KUDOS Resource # {#well-known-kudos-desc}

According to this specification, KUDOS is transferred in POST requests and 2.04 (Changed) responses. If a client wishes to execute the KUDOS procedure as initiator without triggering any application processing on the server, then the request sent as first KUDOS message must target a KUDOS resource, e.g., at the Uri-Path "/.well-known/kudos" (see {{well-known-kudos}}), or at an alternative Uri-Path that can be discovered, e.g., by using a resource directory {{RFC9176}}.

In order to discover a server's well-known KUDOS resource and possible other KUDOS resources, client applications can use the resource type "core.kudos" (see {{rt-kudos}}).

### Rekeying when Using SCHC with OSCORE

In the interest of rekeying, the following points must be taken into account when using the Static Context Header Compression and fragmentation (SCHC) framework {{RFC8724}} for compressing CoAP messages protected with OSCORE, as defined in {{RFC8824}}.

Compression of the OSCORE Partial IV has implications for the frequency of rekeying. That is, if the Partial IV is compressed, the communicating peers must perform rekeying more often, as the available Partial IV space becomes smaller due to the compression. For instance, if only 3 bits of the Partial IV are sent, then the maximum PIV before having to rekey is only 2^3 - 1 = 7.

Furthermore, any time the SCHC context Rules are updated on an OSCORE endpoint, that endpoint must perform a rekeying (see {{Section 9 of RFC8824}}).

That is, the use of SCHC plays a role in triggering KUDOS executions and in affecting their cadence. Hence, the used SCHC Rules and their update policies should ensure that the KUDOS executions occurring as their side effect do not significantly impair the gain from message compression.

## Signaling KUDOS support in EDHOC # {#edhoc-ead-signaling}

The EDHOC protocol defines the transport of additional External Authorization Data (EAD) within an optional EAD field of the EDHOC messages (see {{Section 3.8 of RFC9528}}). An EAD field is composed of one or multiple EAD items, each of which specifies an identifying 'ead\_label' encoded as a CBOR integer, and an optional 'ead\_value' encoded as a CBOR bstr.

This document defines a new EDHOC EAD item KUDOS\_EAD and registers its 'ead\_label' in {{iana-edhoc-aad}}. By including this EAD item in an outgoing EDHOC message, a sender peer can indicate whether it supports KUDOS and in which modes, as well as query the other peer about its support. Note that peers do not have to use this EDHOC EAD item to be able to run KUDOS with each other, irrespective of the modes they support. A KUDOS peer MUST only use the EDHOC EAD item KUDOS_EAD as non-critical. That is, when included in an EDHOC message, its 'ead\_label' MUST be used with positive sign. The possible values of the 'ead\_value' are as follows:

| Name | Value          | Description                                                                                                             |
| ASK  | h'' (0x40)     | Used only in EDHOC message_1. It asks the recipient peer to specify in EDHOC message_2 whether it supports KUDOS.       |
| NONE | h'00' (0x4100) | Used only in EDHOC message_2 and message_3. It specifies that the sender peer does not support KUDOS.                   |
| FULL | h'01' (0x4101) | Used only in EDHOC message_2 and message_3. It specifies that the sender peer supports KUDOS in FS mode and no-FS mode. |
| PART | h'02' (0x4102) | Used only in EDHOC message_2 and message_3. It specifies that the sender peer supports KUDOS in no-FS mode only.        |
{: #table-kudos-ead title="Values for the EDHOC EAD item KUDOS_EAD" align="center"}

When the KUDOS\_EAD item is included in EDHOC message\_1 with 'ead\_value' ASK, a recipient peer that supports the KUDOS\_EAD item MUST specify whether it supports KUDOS in EDHOC message\_2.

When the KUDOS\_EAD item is not included in EDHOC message\_1 with 'ead\_value' ASK, a recipient peer that supports the KUDOS\_EAD item MAY still specify whether it supports KUDOS in EDHOC message\_2.

When the KUDOS\_EAD item is included in EDHOC message\_2 with 'ead\_value' FULL or PART, a recipient peer that supports the KUDOS\_EAD item SHOULD specify whether it supports KUDOS in EDHOC message\_3. An exception applies in case, based on application policies or other context information, the recipient peer that receives EDHOC message\_2 already knows that the sender peer is supposed to have such knowledge.

When the KUDOS\_EAD item is included in EDHOC message\_2 with 'ead\_value' NONE, a recipient peer that supports the KUDOS\_EAD item MUST NOT specify whether it supports KUDOS in EDHOC message\_3.

In the following cases, the recipient peer silently ignores the KUDOS\_EAD item specified in the received EDHOC message, and does not include a KUDOS\_EAD item in the next EDHOC message it sends (if any).

   * The recipient peer does not support the KUDOS\_EAD item.

   * The KUDOS\_EAD item is included in EDHOC message\_1 with 'ead\_value' different than ASK

   * The KUDOS\_EAD item is included in EDHOC message\_2 or message\_3 with 'ead\_value' ASK.

   * The KUDOS\_EAD item is included in EDHOC message\_4.

That is, by specifying 'ead\_value' ASK in EDHOC message\_1, a peer A can indicate to the other peer B that it wishes to know if B supports KUDOS and in what mode(s). In the following EDHOC message\_2, B indicates whether it supports KUDOS and in what mode(s), by specifying either NONE, FULL, or PART as 'ead\_value'. Specifying the 'ead\_value' FULL or PART in EDHOC message\_2 also asks A to indicate whether it supports KUDOS in EDHOC message\_3.

To further illustrate the functionality, two examples are presented below as EDHOC executions where only the new KUDOS\_EAD item is shown when present, and assuming that no other EAD items are used by the two peers.

~~~~~~~~~~~ aasvg
EDHOC                                                 EDHOC
Initiator                                         Responder
|                                                         |
|                EAD_1: (TBD_LABEL, ASK)                  |
+-------------------------------------------------------->|
|                        message_1                        |
|                                                         |
|                EAD_2: (TBD_LABEL, FULL)                 |
|<--------------------------------------------------------+
|                        message_2                        |
|                                                         |
|                EAD_3: (TBD_LABEL, FULL)                 |
+-------------------------------------------------------->|
|                        message_3                        |
|                                                         |
~~~~~~~~~~~
{: artwork-align="center"}

In the example above, the Initiator asks the EDHOC Responder about its support for KUDOS ('ead\_value' = ASK). In EDHOC message\_2, the Responder indicates that it supports both the FS and no-FS mode of KUDOS ('ead\_value' = FULL). Finally, in EDHOC message\_3, the Initiator indicates that it also supports both the FS and no-FS mode of KUDOS ('ead\_value' = FULL). After the EDHOC execution has successfully finished, both peers are aware that they both support KUDOS, in the FS and no-FS modes.

~~~~~~~~~~~ aasvg
EDHOC                                                 EDHOC
Initiator                                         Responder
|                                                         |
|                EAD_1: (TBD_LABEL, ASK)                  |
+-------------------------------------------------------->|
|                        message_1                        |
|                                                         |
|                EAD_2: (TBD_LABEL, NONE)                 |
|<--------------------------------------------------------+
|                        message_2                        |
|                                                         |
+-------------------------------------------------------->|
|                        message_3                        |
|                                                         |
~~~~~~~~~~~
{: artwork-align="center"}

In this second example, the Initiator asks the EDHOC Responder about its support for KUDOS ('ead\_value' = ASK). In EDHOC message\_2, the Responder indicates that it does not support KUDOS at all ('ead\_value' = NONE). Finally, in EDHOC message\_3, the Initiator does not include the KUDOS\_EAD item, since it already knows that using KUDOS with the other peer will not be possible. After the EDHOC execution has successfully finished, the Initiator is aware that the Responder does not support KUDOS, which the two peers are not going to use with each other.

# Security Considerations {#sec-cons}

Depending on the specific key update procedure used to establish a new OSCORE Security Context, the related security considerations also apply.

As mentioned in {{ssec-nonces-x-bytes}}, it is RECOMMENDED that the size for nonces N1 and N2 is 8 bytes. The application needs to set the size of each nonce such that the probability of its value being repeated is negligible. Note that the probability of collision of nonce values is heightened by the birthday paradox. However, considering a nonce size of 8 bytes there will be a collision on average after approximately 2^32 instances of Response #1 messages.

Overall, the size of the nonces N1 and N2 should be set such that the security level is harmonized with other components of the deployment. Considering the constraints of embedded implementations, there might be a need for allowing N1 and N2 values that are smaller in size. This is acceptable, provided that safety, reliability, and robustness within the system can still be assured. Although using nonces that are smaller in size means that there will be a collision on average after fewer KUDOS messages have been sent, this should not pose significant problems even for a constrained server operating at a capacity of one request per second.

The nonces exchanged in the KUDOS messages are sent in the clear, so using random nonces is preferable for maintaining privacy. If instead a counter value is used, this can leak some information about the peers. Specifically, using counters will reveal the frequency of rekeying procedures performed.

As discussed in {{Symmetric-Security}}, key update methods built on symmetric key exchange have weaker security properties compared to methods built on ephemeral Diffie-Hellman key exchange. In fact, while the two approaches can co-exist, rekeying with symmetric key exchange is not intended as a substitute for ephemeral Diffie-Hellman key exchange. Peers should periodically perform a key update based on ephemeral Diffie-Hellman key exchange (e.g., by running the EDHOC protocol {{RFC9528}}). The cadence of such periodic key updates should be determined based on how much the two peers and their network environment are constrained, as well as on the maximum amount of time and of exchanged data that are acceptable between two consecutive key updates.

# IANA Considerations # {#sec-iana}

This document has the following actions for IANA.

Note to RFC Editor: Please replace all occurrences of "{{&SELF}}" with the RFC number of this specification and delete this paragraph.

## OSCORE Flag Bits Registry {#iana-cons-flag-bits}

IANA is asked to add the following entries to the "OSCORE Flag Bits" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

| Bit Position | Name             | Description                                                                                       | Reference |
| 0            | Extension-1 Flag | Set to 1 if the OSCORE Option specifies a second byte, which includes the OSCORE flag bits 8-15   | {{&SELF}} |
| 8            | Extension-2 Flag | Set to 1 if the OSCORE Option specifies a third byte, which includes the OSCORE flag bits 16-23   | {{&SELF}} |
| 15           | Nonce Flag       | Set to 1 if nonce is present in the compressed COSE object                                        | {{&SELF}} |
| 16           | Extension-3 Flag | Set to 1 if the OSCORE Option specifies a fourth byte, which includes the OSCORE flag bits 24-31  | {{&SELF}} |
| 24           | Extension-4 Flag | Set to 1 if the OSCORE Option specifies a fifth byte, which includes the OSCORE flag bits 32-39   | {{&SELF}} |
| 32           | Extension-5 Flag | Set to 1 if the OSCORE Option specifies a sixth byte, which includes the OSCORE flag bits 40-47   | {{&SELF}} |
| 40           | Extension-6 Flag | Set to 1 if the OSCORE Option specifies a seventh byte, which includes the OSCORE flag bits 48-55 | {{&SELF}} |
| 48           | Extension-7 Flag | Set to 1 if the OSCORE Option specifies an eigth byte, which includes the OSCORE flag bits 56-63  | {{&SELF}} |
{: #table-iana-oscore-flag-bits title="Registrations in the OSCORE Flag Bits Registry" align="center"}

In the same registry, IANA is asked to mark as 'Unassigned' the entry with Bit Position of 1, i.e., to update the entry as follows.

| Bit Position | Name       | Description | Reference |
| 1            | Unassigned |             |           |
{: #table-iana-oscore-flag-bits-2 title="Update in the OSCORE Flag Bits Registry" align="center"}

## CoAP Option Numbers Registry

IANA is asked to add this document as a reference for the OSCORE Option in the "CoAP Option Numbers" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

## EDHOC External Authorization Data Registry {#iana-edhoc-aad}

IANA is asked to add the following entries to the "EDHOC External Authorization Data" registry defined in {{Section 10.5 of RFC9528}} within the "Ephemeral Diffie-Hellman Over COSE (EDHOC)" registry group.

| Label   | Description                                                     | Reference |
| TBD1    | Indicates whether this peer supports KUDOS and in which mode(s) | {{&SELF}} |
{: #table-iana-edhoc-ead title="Registrations in the EDHOC External Authorization Data Registry" align="center"}

## The Well-Known URI Registry {#well-known-kudos}

IANA is asked to add the 'kudos' well-known URI to the Well-Known URIs registry as defined by {{RFC8615}}.

- URI suffix: kudos

- Change controller: IETF

- Specification document(s): {{&SELF}}

- Related information: None

## Resource Type (rt=) Link Target Attribute Values Registry {#rt-kudos}

IANA is requested to add the resource type "core.kudos" to the "Resource Type (rt=) Link Target Attribute Values" registry under the registry group "Constrained RESTful Environments (CoRE) Parameters".

-  Value: "core.kudos"

-  Description: KUDOS resource.

-  Reference: {{&SELF}}

--- back

# Forward Message Flow using two CoAP Requests {#ssec-derive-ctx-client-init-requests-only}

This section presents an example of KUDOS run in the forward message flow, with the client acting as KUDOS initiator, and both KUDOS messages being CoAP requests.

The example uses the same notation 'Comb(a,b)' used in {{ssec-derive-ctx-client-init}}.

~~~~~~~~~~~ aasvg
                  Client/Server          Client/Server
                   (initiator)            (responder)
                        |                      |
Generate N1             |                      |
                        |                      |
CTX_1 = updateCtx(      |                      |
        X1,             |                      |
        N1,             |                      |
        CTX_OLD )       |                      |
                        |                      |
                        |      Request #1      |
Protect with CTX_1      +--------------------->| /.well-known/kudos
                        | Token: 0x4a          |
                        | OSCORE {             |
                        |  ...                 |
                        |  Partial IV: 0       |
                        |  ...                 |
                        |  d flag: 1           | CTX_1 = updateCtx(
                        |  x: X1               |         X1,
                        |  nonce: N1           |         N1,
                        |  ...                 |         CTX_OLD )
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_1
                        |  ...                 |
                        | }                    | Generate N2
                        |                      |
                        |                      | CTX_NEW = updateCtx(
                        |                      |           Comb(X1,X2),
                        |                      |           Comb(N1,N2),
                        |                      |           CTX_OLD )
                        |                      |
                        |      Request #2      |
     /.well-known/kudos |<---------------------+ Protect with CTX_NEW
                        | Token: 0x7c          |
                        | OSCORE {             |
                        |  ...                 |
CTX_NEW = updateCtx(    |  Partial IV: 0       |
          Comb(X1,X2),  |  ...                 |
          Comb(N1,N2),  |  d flag: 1           |
          CTX_OLD )     |  x: X2               |
                        |  nonce: N2           |
Verify with CTX_NEW     |  ...                 |
                        | }                    |
Discard CTX_OLD         | Encrypted Payload {  |
                        |  ...                 |
                        | }                    |
                        |                      |

The actual key update process ends here.
The two peers can use the new Security Context CTX_NEW.

                        |                      |
                        |      Response #1     |
Protect with CTX_NEW    +--------------------->|
                        | Token: 0x7c          |
                        | OSCORE {             |
                        |  ...                 |
                        | }                    | Verify with CTX_NEW
                        | Encrypted Payload {  |
                        |  ...                 | Discard CTX_OLD
                        |  Application Payload |
                        | }                    |
                        |                      |
                        |      Response #2     |
                        |<---------------------+ Protect with CTX_NEW
                        | Token: 0x4a          |
                        | OSCORE {             |
                        |  ...                 |
Verify with CTX_NEW     | }                    |
                        | Encrypted Payload {  |
                        |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |
~~~~~~~~~~~
{: #fig-message-exchange-client-init-requests-only title="Example of the KUDOS forward message flow where both KUDOS messages are requests." artwork-align="center"}

# Forward Message Flow with Response #1 unrelated to Request #1 {#ssec-derive-ctx-client-init-unrelated}

This section presents an example of KUDOS run in the forward message flow, with the client acting as KUDOS initiator, and where the second KUDOS message Response #1 is not a response to the first KUDOS message Request #2, but rather an unrelated Observe notification as a response to the non-KUDOS message Request #1

The example uses the same notation 'Comb(a,b)' used in {{ssec-derive-ctx-client-init}}.

~~~~~~~~~~~ aasvg
                     Client                  Server
                   (initiator)            (responder)
                        |                      |
                        |      Request #1      |
                        |    (Registration)    |
Protect with CTX_OLD    +--------------------->| /obs
                        | Token: 0x4a          |
                        | Observe: 0           |
                        | OSCORE {             |
                        |  ...                 |
                        |  Partial IV: 4324    |
                        |  ...                 |
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_OLD
                        |  Observe: -          |
                        |  ...                 |
                        | }                    |
                        |                      |
Generate N1             |                      |
                        |                      |
CTX_1 = updateCtx(      |                      |
        X1,             |                      |
        N1,             |                      |
        CTX_OLD )       |                      |
                        |                      |
                        |      Request #2      |
Protect with CTX_1      +--------------------->| /.well-known/kudos
                        | Token: 0x7c          |
                        | OSCORE {             |
                        |  ...                 |
                        |  Partial IV: 0       |
                        |  ...                 |
                        |  d flag: 1           | CTX_1 = updateCtx(
                        |  x: X1               |         X1,
                        |  nonce: N1           |         N1,
                        |  ...                 |         CTX_OLD )
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_1
                        |  ...                 |
                        | }                    | Generate N2
                        |                      |
                        |                      | CTX_NEW = updateCtx(
                        |                      |           Comb(X1,X2),
                        |                      |           Comb(N1,N2),
                        |                      |           CTX_OLD )
                        |                      |
                        |      Response #1     |
                        |    (Notification)    |
                        |<---------------------+ Protect with CTX_NEW
                        | Token: 0x4a          |
                        | Observe: 1           |
                        | OSCORE {             |
                        |  ...                 |
CTX_NEW = updateCtx(    |  Partial IV: 0       |
          Comb(X1,X2),  |  ...                 |
          Comb(N1,N2),  |  d flag: 1           |
          CTX_OLD )     |  x: X2               |
                        |  nonce: N2           |
Verify with CTX_NEW     |  ...                 |
                        | }                    |
Discard CTX_OLD         | Encrypted Payload {  |
                        | Observe: -           |
                        |  ...                 |
                        | }                    |
                        |                      |

The actual key update process ends here.
The two peers can use the new Security Context CTX_NEW.

                        |      Response #2     |
                        |<---------------------+ Protect with CTX_NEW
                        | Token: 0x7c          |
                        | OSCORE {             |
                        |  ...                 |
Verify with CTX_NEW     | }                    |
                        | Encrypted Payload {  |
                        |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |
~~~~~~~~~~~
{: #fig-message-exchange-client-init-unrelated-response title="Example of the KUDOS forward message flow where the second KUDOS message Response #1 is not a response to Request #1." artwork-align="center"}

# Forward Message Flow Targeting a non-KUDOS Resource at Server {#ssec-derive-ctx-client-init-normal-resource}

This section presents an example of KUDOS run in the forward message flow, with the client acting as KUDOS initiator, and with the KUDOS message Request #1 targeting a non-KUDOS resource at the Uri-Path "/temp". The server application has freshness requirements on the requests targeting the resource at "/temp".

Note the presence of an application payload in the KUDOS message Request #1 and in the non-KUDOS message Request #2, both of which are composed as PUT requests. That request method is part of the encrypted payload, since it is protected by OSCORE.

Also note the fact that the KUDOS message Response #1 is composed as a 4.01 (Unauthorized) response, while the non-KUDOS message Response #2 is composed as a 2.04 (Changed) repsonse. Those response codes are part of the encrypted payload, since they are protected by OSCORE.

The example uses the same notation 'Comb(a,b)' used in {{ssec-derive-ctx-client-init}}.

~~~~~~~~~~~ aasvg
                     Client                  Server
                   (initiator)            (responder)
                        |                      |
Generate N1             |                      |
                        |                      |
CTX_1 = updateCtx(      |                      |
        X1,             |                      |
        N1,             |                      |
        CTX_OLD )       |                      |
                        |                      |
                        |      Request #1      |
Protect with CTX_1      +--------------------->| /temp
                        | OSCORE {             |
                        |  ...                 |
                        |  Partial IV: 0       |
                        |  ...                 |
                        |  d flag: 1           | CTX_1 = updateCtx(
                        |  x: X1               |         X1,
                        |  nonce: N1           |         N1,
                        |  ...                 |         CTX_OLD )
                        | }                    |
                        | Encrypted Payload {  | Verify with CTX_1
                        |  0.03 (PUT)          |
                        |  ...                 |
                        |  Application Payload |
                        | }                    | Generate N2
                        |                      |
                        |                      | CTX_NEW = updateCtx(
                        |                      |           Comb(X1,X2),
                        |                      |           Comb(N1,N2),
                        |                      |           CTX_OLD )
                        |                      |
                        |      Response #1     |
                        |<---------------------+ Protect with CTX_NEW
                        | OSCORE {             |
                        |  ...                 |
CTX_NEW = updateCtx(    |  Partial IV: 0       |
          Comb(X1,X2),  |  ...                 |
          Comb(N1,N2),  |  d flag: 1           |
          CTX_OLD )     |  x: X2               |
                        |  nonce: N2           |
Verify with CTX_NEW     |  ...                 |
                        | }                    |
Discard CTX_OLD         | Encrypted Payload {  |
                        |  4.01 (Unauthorized) |
                        |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |

The actual key update process ends here.
The two peers can use the new Security Context CTX_NEW.

                        |                      |
                        |      Request #2      |
Protect with CTX_NEW    +--------------------->| /temp
                        | OSCORE {             |
                        |  ...                 |
                        | }                    | Verify with CTX_NEW
                        | Encrypted Payload {  |
                        |  0.03 (PUT)          |
                        |  ...                 | Discard CTX_OLD
                        |  Application Payload |
                        | }                    |
                        |                      |
                        |      Response #2     |
                        |<---------------------+ Protect with CTX_NEW
                        | OSCORE {             |
                        |  ...                 |
Verify with CTX_NEW     | }                    |
                        | Encrypted Payload {  |
                        |  2.04 (Changed)      |
                        |  ...                 |
                        |  Application Payload |
                        | }                    |
                        |                      |
~~~~~~~~~~~
{: #fig-message-exchange-client-init-normal-resource title="Example of the KUDOS forward message flow where the KUDOS message Request #1 targets a non-KUDOS resource." artwork-align="center"}

# Document Updates # {#sec-document-updates}
{:removeinrfc}

## Version -08 to -09 ## {#sec-08-09}

* Merge text about avoiding in-transit requests during a key update into a single subsection.

* Improved error handling.

* Editorial improvements and clarifications.

* State that the EDHOC EAD item must be used as non-critical.

* Extended description and updates values for KUDOS communication overhead.

## Version -07 to -08 ## {#sec-07-08}

* Add note about usage of the CoAP No-Response Option.

* Avoid problems for two simultaneously started key updates.

* Set Notification Number to be uninitialized for new OSCORE Security Contexts.

* Handle corner case for responder that reached its key usage limits.

* Re-organizing main section about Forward Secrecy mode into subsections.

* IANA considerations for CoAP Option Numbers Registry to refer to this draft for the OSCORE option.

* Use AASVG in diagrams.

* Use actual tables instead of figures.

* Clarifications and editorial improvements.

* Extended security considerations with reference to relevant paper.

## Version -06 to -07 ## {#sec-06-07}

* Removed material about the ID update procedure, which has been split out into a separate draft.

* Allow non-random nonces for CAPABLE devices.

* Editorial improvements.

* Permit flexible message flow with KUDOS messages as any request/response.

* Enable sending KUDOS messages as regular application messages.

## Version -05 to -06 ## {#sec-05-06}

* Mandate support for both the forward and reverse message flow.

* Mention the EDHOC and OSCORE profile of ACE as method for rekeying.

* Clarify definition of KUDOS (request/response) message.

* Further extend the OSCORE option to transport N1 in the second KUDOS message as a request.

* Mandate support for the no-FS mode on CAPABLE devices.

* Explain when KUDOS fails during selection of mode.

* Explicitly forbid using old keying material after reboot.

* Editorial improvements.

## Version -04 to -05 ## {#sec-04-05}

* Note on client retransmissions if KUDOS execution fails in reverse message flow.

* Specify what information needs to be written to non-volatile memory to handle reboots.

* Extended recommendations and considerations on minimum size of nonces N1 & N2.

* Arbitrary maximum size of the Recipient-ID Option.

* Detailed lifecycle of the OSCORE IDs update procedure.

* Described examples of OSCORE IDs update procedure.

* Examples of OSCORE IDs update procedure integrated in KUDOS.

* Considerations about using SCHC for CoAP with OSCORE.

* Clarifications and editorial improvements.

## Version -03 to -04 ## {#sec-03-04}

* Removed content about key usage limits.

* Use of "forward message flow" and "reverse message flow".

* Update to RFC 8613 extended to include protection of responses.

* Include EDHOC_KeyUpdate() in the methods for rekeying.

* Describe reasons for using the OSCORE ID update procedure.

* Clarifications on deletion of CTX\_OLD and CTX\_NEW.

* Added new section on preventing deadlocks.

* Clarified that peers can decide to run KUDOS at any point.

* Defined preservation of observations beyond OSCORE ID updates.

* Revised discussion section, including also communication overhead.

* Defined a well-known KUDOS resource and a KUDOS resource type.

* Editorial improvements.

## Version -02 to -03 ## {#sec-02-03}

* Use of the OSCORE flag bit 0 to signal more flag bits.

* In UpdateCtx(), open for future key derivation different than HKDF.

* Simplified updateCtx() to use only Expand(); used to be METHOD 2.

* Included the Partial IV if the second KUDOS message is a response.

* Added signaling of support for KUDOS in EDHOC.

* Clarifications on terminology and reasons for rekeying.

* Updated IANA considerations.

* Editorial improvements.

## Version -01 to -02 ## {#sec-01-02}

* Extended terminology.

* Moved procedure for preserving observations across key updates to main body.

* Moved procedure to update OSCORE Sender/Recipient IDs to main body.

* Moved key update without forward secrecy section to main body.

* Define signaling bits present in the 'x' byte.

* Modifications and alignment of updateCtx() with EDHOC.

* Rules for deletion of old EDHOC keys PRK\_out and PRK\_exporter.

* Describe CBOR wrapping of involved nonces with examples.

* Renamed 'id detail' to 'nonce'.

* Editorial improvements.

## Version -00 to -01 ## {#sec-00-01}

* Recommendation on limits for CCM_8. Details in Appendix.

* Improved message processing, also covering corner cases.

* Example of method to estimate and not store 'count_q'.

* Added procedure to update OSCORE Sender/Recipient IDs.

* Added method for preserving observations across key updates.

* Added key update without forward secrecy.

# Acknowledgments # {#acknowledgments}
{:numbered="false"}

The authors sincerely thank {{{Christian Amsüss}}}, {{{Carsten Bormann}}}, {{{Simon Bouget}}}, {{{Rafa Marin-Lopez}}}, {{{John Preuß Mattsson}}}, and {{{Göran Selander}}} for their feedback and comments.

The work on this document has been partly supported by the Sweden's Innovation Agency VINNOVA and the Celtic-Next projects CRITISEC and CYPRESS; and by the H2020 projects SIFIS-Home (Grant agreement 952652) and ARCADIAN-IoT (Grant agreement 101020259).
