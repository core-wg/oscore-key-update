---
title:  Key Update for OSCORE (KUDOS)
abbrev: Key Update for OSCORE (KUDOS)
docname: draft-ietf-core-oscore-key-update-latest

# stand_alone: true

ipr: trust200902
area: Internet
wg: CoRE Working Group
kw: Internet-Draft
cat: std
updates: 8613

coding: utf-8
pi:    # can use array (if all yes) or hash here

  toc: yes
  sortrefs:   # defaults to yes
  symrefs: yes

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
  RFC2119:
  RFC7252:
  RFC7641:
  RFC8174:
  RFC8613:
  RFC8949:
  I-D.ietf-lake-edhoc:

informative:
  RFC8446:
  RFC7519:
  RFC7554:
  RFC8180:
  RFC9031:
  I-D.ietf-ace-oauth-authz:
  I-D.ietf-ace-oscore-profile:
  I-D.irtf-cfrg-aead-limits:
  LwM2M:
    author:
      org: Open Mobile Alliance
    title: Lightweight Machine to Machine Technical Specification - Core, Approved Version 1.2, OMA-TS-LightweightM2M_Core-V1_2-20201110-A
    date: 2020-11
    target: http://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Core-V1_2-20201110-A.pdf
  LwM2M-Transport:
    author:
      org: Open Mobile Alliance
    title: Lightweight Machine to Machine Technical Specification - Transport Bindings, Approved Version 1.2, OMA-TS-LightweightM2M_Transport-V1_2-20201110-A
    date: 2020-11
    target: http://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/OMA-TS-LightweightM2M_Transport-V1_2-20201110-A.pdf

--- abstract

Object Security for Constrained RESTful Environments (OSCORE) uses AEAD algorithms to ensure confidentiality and integrity of exchanged messages. Due to known issues allowing forgery attacks against AEAD algorithms, limits should be followed on the number of times a specific key is used for encryption or decryption. This document defines how two OSCORE peers must follow these limits and what steps they must take to preserve the security of their communications. Therefore, this document updates RFC8613. Furthermore, this document specifies Key Update for OSCORE (KUDOS), a lightweight procedure that two peers can use to update their keying material and establish a new OSCORE Security Context.

--- middle

# Introduction # {#intro}

Object Security for Constrained RESTful Environments (OSCORE) {{RFC8613}} provides end-to-end protection of CoAP {{RFC7252}} messages at the application-layer, ensuring message confidentiality and integrity, replay protection, as well as binding of response to request between a sender and a recipient.

In particular, OSCORE uses AEAD algorithms to provide confidentiality and integrity of messages exchanged between two peers. Due to known issues allowing forgery attacks against AEAD algorithms, limits should be followed on the number of times a specific key is used to perform encryption or decryption {{I-D.irtf-cfrg-aead-limits}}.

Should these limits be exceeded, an adversary may break the security properties of the AEAD algorithm, such as message confidentiality and integrity, e.g. by performing a message forgery attack. The original OSCORE specification {{RFC8613}} does not consider such limits.

This document updates {{RFC8613}} as follows.

* It defines when a peer must stop using an OSCORE Security Context shared with another peer, due to the reached key usage limits. When this happens, the two peers have to establish a new Security Context with new keying material, in order to continue their secure communication with OSCORE.

* It specifies KUDOS, a lightweight key update procedure that the two peers can use in order to update their current keying material and establish a new OSCORE Security Context. This deprecates and replaces the procedure specified in {{Section B.2 of RFC8613}}.

## Terminology ## {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Readers are expected to be familiar with the terms and concepts related to the CoAP {{RFC7252}} and OSCORE {{RFC8613}} protocols.

# AEAD Key Usage Limits in OSCORE

The following sections details how key usage limits for AEAD algorithms must be considered when using OSCORE. It covers specific limits for common AEAD algorithms used with OSCORE; necessary additions to the OSCORE Security Context, updates to the OSCORE message processing, and existing methods for rekeying OSCORE.

## Problem Overview {#problem-overview}

The OSCORE security protocol {{RFC8613}} uses AEAD algorithms to provide integrity and confidentiality of messages, as exchanged between two peers sharing an OSCORE Security Context.

When processing messages with OSCORE, each peer should follow specific limits as to the number of times it uses a specific key. This applies separately to the Sender Key used to encrypt outgoing messages, and to the Recipient Key used to decrypt and verify incoming protected messages.

Exceeding these limits may allow an adversary to break the security properties of the AEAD algorithm, such as message confidentiality and integrity, e.g. by performing a message forgery attack.

The following refers to the two parameters 'q' and 'v' introduced in {{I-D.irtf-cfrg-aead-limits}}, to use when deploying an AEAD algorithm.

* 'q': this parameter has as value the number of messages protected with a specific key, i.e. the number of times the AEAD algorithm has been invoked to encrypt data with that key.

* 'v': this parameter has as value the number of alleged forgery attempts that have been made against a specific key, i.e. the amount of failed decryptions that has been done with the AEAD algorithm for that key.

When a peer uses OSCORE:

* The key used to protect outgoing messages is its Sender Key, in its Sender Context.

* The key used to decrypt and verify incoming messages is its Recipient Key, in its Recipient Context.

Both keys are derived as part of the establishment of the OSCORE Security Context, as defined in {{Section 3.2 of RFC8613}}.

As mentioned above, exceeding specific limits for the 'q' or 'v' value can weaken the security properties of the AEAD algorithm used, thus compromising secure communication requirements.

Therefore, in order to preserve the security of the used AEAD algorithm, OSCORE has to observe limits for the 'q' and 'v' values, throughout the lifetime of the used AEAD keys.

### Limits for 'q' and 'v' {#limits}

Formulas for calculating the security levels as Integrity Advantage (IA) and Confidentiality Advantage (CA) probabilities, are presented in {{I-D.irtf-cfrg-aead-limits}}. These formulas take as input specific values for 'q' and 'v' (see section {{problem-overview}}) and for 'l', i.e., the maximum length of each message (in cipher blocks).

For the algorithms that can be used as AEAD Algorithm for OSCORE shows in {{algorithm-limits}}, the key property to achieve is having IA and CA values which are no larger than p = 2^-64, which will ensure a safe security level for the AEAD Algorithm. This can be entailed by using the values q = 2^20, v = 2^20, and l = 2^10, that this document recommends to use for these algorithms.

{{algorithm-limits}} shows the resulting IA and CA probabilities enjoyed by the considered algorithms, when taking the value of 'q', 'v' and 'l' above as input to the formulas defined in {{I-D.irtf-cfrg-aead-limits}}.

~~~~~~~~~~~
+------------------------+----------------+----------------+
| Algorithm name         | IA probability | CA probability |
|------------------------+----------------+----------------|
| AEAD_AES_128_CCM       | 2^-64          | 2^-66          |
| AEAD_AES_128_GCM       | 2^-97          | 2^-89          |
| AEAD_AES_256_GCM       | 2^-97          | 2^-89          |
| AEAD_CHACHA20_POLY1305 | 2^-73          | -              |
+------------------------+----------------+----------------+
~~~~~~~~~~~
{: #algorithm-limits title="Probabilities for algorithms based on chosen q, v and l values." artwork-align="center"}

When AEAD_AES_128_CCM_8 is used as AEAD Algorithm for OSCORE, the triplet (q, v, l) considered above yields larger values of IA and CA. Hence, specifically for AEAD_AES_128_CCM_8, this document recommends using the triplet (q, v, l) = (2^20, 2^14, 2^8). This is appropriate since the resulting CA and IA values are not greater than the threshold value of 2^-50 defined in {{I-D.irtf-cfrg-aead-limits}}, and thus yields an acceptable security level. Achieving smaller values of CA and IA would require to inconveniently reduce 'q', 'v' or 'l', with no corresponding increase in terms of security. This is further elaborated in {{aead-aes-128-ccm-8-details}}.

~~~~~~~~~~~
+------------------------+----------+----------+-----------+
| Algorithm name         | l=2^6 in | l=2^8 in | l=2^10 in |
|                        | bytes    | bytes    | bytes     |
|------------------------+----------+----------|-----------|
| AEAD_AES_128_CCM       | 1024     | 4096     | 16384     |
| AEAD_AES_128_GCM       | 1024     | 4096     | 16384     |
| AEAD_AES_256_GCM       | 1024     | 4096     | 16384     |
| AEAD_AES_128_CCM_8     | 1024     | 4096     | 16384     |
| AEAD_CHACHA20_POLY1305 | 4096     | 16384    | 65536     |
+------------------------+----------+----------+-----------+
~~~~~~~~~~~
{: #l-values-as-bytes title="Maximum length of each message (in bytes)" artwork-align="center"}


## Additional Information in the Security Context # {#context}

In addition to what defined in {{Section 3.1 of RFC8613}}, the OSCORE Security Context MUST also include the following information.

### Common Context # {#common-context}

The Common Context is extended to include the following parameter.

* 'exp': with value the expiration time of the OSCORE Security Context, as a non-negative integer. The parameter contains a numeric value representing the number of seconds from 1970-01-01T00:00:00Z UTC until the specified UTC date/time, ignoring leap seconds, analogous to what specified for NumericDate in {{Section 2 of RFC7519}}.

   At the time indicated in this field, a peer MUST stop using this Security Context to process any incoming or outgoing message, and is required to establish a new Security Context to continue OSCORE-protected communications with the other peer.

### Sender Context # {#sender-context}

The Sender Context is extended to include the following parameters.

* 'count_q': a non-negative integer counter, keeping track of the current 'q' value for the Sender Key. At any time, 'count_q' has as value the number of messages that have been encrypted using the Sender Key. The value of 'count_q' is set to 0 when establishing the Sender Context.

* 'limit_q': a non-negative integer, which specifies the highest value that 'count_q' is allowed to reach, before stopping using the Sender Key to process outgoing messages.

   The value of 'limit_q' depends on the AEAD algorithm specified in the Common Context, considering the properties of that algorithm. The value of 'limit_q' is determined according to {{limits}}.

Note for implementation: it is possible to avoid storing and maintaining the counter 'count_q'. Rather, an estimated value to be compared against 'limit_q' can be computed, by leveraging the Sender Sequence Number of the peer and (an estimate of) the other peer's. A possible method to achieve this is described in {{estimation-count-q}}. While this relieves peers from storing and maintaining the precise 'count_q' value, it results in overestimating the number of encryptions performed with a Sender Key. This in turn results in approaching 'limit_q' sooner and performing a key update procedure more frequently.

### Recipient Context # {#recipient-context}

The Recipient Context is extended to include the following parameters.

* 'count_v': a non-negative integer counter, keeping track of the current 'v' value for the Recipient Key. At any time, 'count_v' has as value the number of failed decryptions occurred on incoming messages using the Recipient Key. The value of 'count_v' is set to 0 when establishing the Recipient Context.

* 'limit_v': a non-negative integer, which specifies the highest value that 'count_v' is allowed to reach, before stopping using the Recipient Key to process incoming messages.

   The value of 'limit_v' depends on the AEAD algorithm specified in the Common Context, considering the properties of that algorithm. The value of 'limit_v' is determined according to {{limits}}.

## OSCORE Messages Processing #

In order to keep track of the 'q' and 'v' values and ensure that AEAD keys are not used beyond reaching their limits, the processing of OSCORE messages is extended as defined in this section. A limitation that is introduced is that, in order to not exceed the selected value for 'l', the total size of the COSE plaintext, authentication Tag, and possible cipher padding for a message may not exceed the block size for the selected algorithm multiplied with 'l‘.

In particular, the processing of OSCORE messages follows the steps outlined in {{Section 8 of RFC8613}}, with the additions defined below.

### Protecting a Request or a Response ## {#protecting-req-resp}

Before encrypting the COSE object using the Sender Key, the 'count_q' counter MUST be incremented.

If 'count_q' exceeds the 'limit_q' limit, the message processing MUST be aborted. From then on, the Sender Key MUST NOT be used to encrypt further messages.

### Verifying a Request or a Response ##

If an incoming message is detected to be a replay (see {{Section 7.4 of RFC8613}}), the 'count_v' counter MUST NOT be incremented.

If the decryption and verification of the COSE object using the Recipient Key fails, the 'count_v' counter MUST be incremented.

After 'count_v' has exceeded the 'limit_v' limit, incoming messages MUST NOT be decrypted and verified using the Recipient Key, and their processing MUST be aborted.

# Current methods for Rekeying OSCORE {#sec-current-methods}

Before the limit of 'q' or 'v' defined in {{limits}} has been reached for an OSCORE Security Context, the two peers have to establish a new OSCORE Security Context, in order to continue using OSCORE for secure communication.

In practice, the two peers have to establish new Sender and Recipient Keys, as the keys actually used by the AEAD algorithm. When this happens, both peers reset their 'count_q' and 'count_v' values to 0 (see {{context}}).

Other specifications define a number of ways to accomplish this, as summarized below.

* The two peers can run the procedure defined in {{Section B.2 of RFC8613}}. That is, the two peers exchange three or four messages, protected with temporary Security Contexts adding randomness to the ID Context.

   As a result, the two peers establish a new OSCORE Security Context with new ID Context, Sender Key and Recipient Key, while keeping the same OSCORE Master Secret and OSCORE Master Salt from the old OSCORE Security Context.

   This procedure does not require any additional components to what OSCORE already provides, and it does not provide perfect forward secrecy.

   The procedure defined in {{Section B.2 of RFC8613}} is used in 6TiSCH networks {{RFC7554}}{{RFC8180}} when handling failure events. That is, a node acting as Join Registrar/Coordinator (JRC) assists new devices, namely "pledges", to securely join the network as per the Constrained Join Protocol {{RFC9031}}. In particular, a pledge exchanges OSCORE-protected messages with the JRC, from which it obtains a short identifier, link-layer keying material and other configuration parameters. As per {{Section 8.3.3 of RFC9031}}, a JRC that experiences a failure event may likely lose information about joined nodes, including their assigned identifiers. Then, the reinitialized JRC can establish a new OSCORE Security Context with each pledge, through the procedure defined in {{Section B.2 of RFC8613}}.

* The two peers can run the OSCORE profile {{I-D.ietf-ace-oscore-profile}} of the Authentication and Authorization for Constrained Environments (ACE) Framework {{I-D.ietf-ace-oauth-authz}}.

  When a CoAP client uploads an Access Token to a CoAP server as an access credential, the two peers also exchange two nonces. Then, the two peers use the two nonces together with information provided by the ACE Authorization Server that issued the Access Token, in order to derive an OSCORE Security Context.

  This procedure does not provide perfect forward secrecy.

* The two peers can run the EDHOC key exchange protocol based on Diffie-Hellman and defined in {{I-D.ietf-lake-edhoc}}, in order to establish a pseudo-random key in a mutually authenticated way.

   Then, the two peers can use the established pseudo-random key to derive external application keys. This allows the two peers to securely derive especially an OSCORE Master Secret and an OSCORE Master Salt, from which an OSCORE Security Context can be established.

   This procedure additionally provides perfect forward secrecy.

* If one peer is acting as LwM2M Client and the other peer as LwM2M Server, according to the OMA Lightweight Machine to Machine Core specification {{LwM2M}}, then the LwM2M Client peer may take the initiative to bootstrap again with the LwM2M Bootstrap Server, and receive again an OSCORE Security Context. Alternatively, the LwM2M Server can instruct the LwM2M Client to initiate this procedure.

   If the OSCORE Security Context information on the LwM2M Bootstrap Server has been updated, the LwM2M Client will thus receive a fresh OSCORE Security Context to use with the LwM2M Server.

   In addition to that, the LwM2M Client, the LwM2M Server as well as the LwM2M Bootstrap server are required to use the procedure defined in {{Section B.2 of RFC8613}} and overviewed above, when they use a certain OSCORE Security Context for the first time {{LwM2M-Transport}}.

Manually updating the OSCORE Security Context at the two peers should be a last resort option, and it might often be not practical or feasible.

Even when any of the alternatives mentioned above is available, it is RECOMMENDED that two OSCORE peers update their Security Context by using the KUDOS procedure as defined in {{sec-rekeying-method}} of this document.

It is RECOMMENDED that the peer initiating the key update procedure starts it before reaching the 'q' or 'v' limits. Otherwise, the AEAD keys possibly to be used during the key update procedure itself may already be or become invalid before the rekeying is completed, which may prevent a successful establishment of the new OSCORE Security Context altogether.

# Key Update for OSCORE (KUDOS) # {#sec-rekeying-method}

This section defines KUDOS, a lightweight procedure that two OSCORE peers can use to update their keying material and establish a new OSCORE Security Context.

KUDOS relies on the support function updateCtx() defined in {{ssec-update-function}} and the message exchange defined in {{ssec-derive-ctx}}. The following properties are fulfilled.

* KUDOS can be initiated by either peer. In particular, the client or the server may start KUDOS by sending the first rekeying message.

* The new OSCORE Security Context enjoys Perfect Forward Secrecy.

* The same ID Context value used in the old OSCORE Security Context is preserved in the new Security Context. Furthermore, the ID Context value never changes throughout the KUDOS execution.

* KUDOS is robust against a peer rebooting, and it especially avoids the reuse of AEAD (nonce, key) pairs.

* KUDOS completes in one round trip. The two peers achieve mutual proof-of-possession in the following exchange, which is protected with the newly established OSCORE Security Context.

## Extensions to the OSCORE Option # {#ssec-oscore-option-extensions}

In order to support the message exchange for establishing a new OSCORE Security Context as defined in {{ssec-derive-ctx}}, this document extends the use of the OSCORE option originally defined in {{RFC8613}} as follows.

* This document defines the usage of the seventh least significant bit, called "Extension-1 Flag", in the first byte of the OSCORE option containing the OSCORE flag bits. This flag bit is specified in {{iana-cons-flag-bits}}.

   When the Extension-1 Flag is set to 1, the second byte of the OSCORE option MUST include the set of OSCORE flag bits 8-15.

* This document defines the usage of the first least significant bit "ID Detail Flag", 'd', in the second byte of the OSCORE option containing the OSCORE flag bits. This flag bit is specified in {{iana-cons-flag-bits}}.

   When it is set to 1, the compressed COSE object contains an 'id detail', to be used for the steps defined in {{ssec-derive-ctx}}. In particular, the 1 byte following 'kid context' (if any) encodes the length x of 'id detail', and the following x bytes encode 'id detail'.

   Hereafter, this document refers to a message where the 'd' flag is set to 0 as "non KUDOS (request/response) message", and to a message where the 'd' flag is set to 1 as "KUDOS (request/response) message".

* The second-to-eighth least significant bits in the second byte of the OSCORE option containing the OSCORE flag bits are reserved for future use. These bits SHALL be set to zero when not in use. According to this specification, if any of these bits are set to 1, the message is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}.

{{fig-oscore-option}} shows the OSCORE option value including also 'id detail'.

~~~~~~~~~~~
 0 1 2 3 4 5 6 7  8   9   10  11  12  13  14  15 <----- n bytes ----->
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+
|0|1|0|h|k|  n  | 0 | 0 | 0 | 0 | 0 | 0 | 0 | d | Partial IV (if any) |
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+

 <- 1 byte -> <----- s bytes ------> <- 1 byte -> <----- x bytes ---->
+------------+----------------------+---------------------------------+
| s (if any) | kid context (if any) | x (if any) | id detail (if any) |
+------------+----------------------+------------+--------------------+

+------------------+
| kid (if any) ... |
+------------------+
~~~~~~~~~~~
{: #fig-oscore-option title="The OSCORE option value, including 'id detail'" artwork-align="center"}

## Function for Security Context Update # {#ssec-update-function}

The updateCtx() function shown in {{function-update}} takes as input a nonce N as well as an OSCORE Security Context CTX\_IN, and returns as output a new OSCORE Security Context CTX\_OUT.

As a first step, the updateCtx() function derives the new values of the Master Secret and Master Salt for CTX\_OUT, according to one of the two following methods. The used method depends on how the two peers established their original Security Context, i.e., the Security Context that they shared before performing KUDOS with one another for the first time.

* If the original Security Context was established by running the EDHOC protocol {{I-D.ietf-lake-edhoc}}, the following applies.

   First, the EDHOC key PRK_4x3m shared by the two peers is updated using the EDHOC-KeyUpdate() function defined in {{Section 4.4 of I-D.ietf-lake-edhoc}}, which takes the nonce N as input.

   After that, the EDHOC-Exporter() function defined in {{Section 4.3 of I-D.ietf-lake-edhoc}} is used to derive the new values for the Master Secret and Master Salt, consistently with what is defined in {{Section A.2 of I-D.ietf-lake-edhoc}}. In particular, the context parameter provided as second argument to the EDHOC-Exporter() function is the empty CBOR byte string (0x40) {{RFC8949}}, which is denoted as h''.

   Note that, compared to the compliance requirements in {{Section 7 of I-D.ietf-lake-edhoc}}, a peer MUST support the EDHOC-KeyUpdate() function, in case it establishes an original Security Context through the EDHOC protocol and intends to perform KUDOS.

* If the original Security Context was established through other means than the EDHOC protocol, the new Master Secret is derived through an HKDF-Expand() step, which takes as input N as well as the Master Secret value from the Security Context CTX\_IN. Instead, the new Master Salt takes N as value.

In either case, the derivation of new values follows the same approach used in TLS 1.3, which is also based on HKDF-Expand (see {{Section 7.1 of RFC8446}}) and used for computing new keying material in case of key update (see {{Section 4.6.3 of RFC8446}}).

After that, the new Master Secret and Master Salt parameters are used to derive a new Security Context CTX\_OUT as per {{Section 3.2 of RFC8613}}. Any other parameter required for the derivation takes the same value as in the Security Context CTX\_IN. Finally, the function returns the newly derived Security Context CTX\_OUT.

~~~~~~~~~~~
updateCtx(N, CTX_IN) {

  CTX_OUT       // The new Security Context
  MSECRET_NEW   // The new Master Secret
  MSALT_NEW     // The new Master Salt

  if <the original Security Context was established through EDHOC> {

    EDHOC-KeyUpdate(N)
    // This results in updating the key PRK_4x3m of the
    // EDHOC session, i.e., PRK_4x3m = Extract(N, PRK_4x3m)

    MSECRET_NEW = EDHOC-Exporter("OSCORE_Master_Secret",
                                 h'', key_length)
      = EDHOC-KDF(PRK_4x3m, TH_4,
                  "OSCORE_Master_Secret", h'', key_length)

    MSALT_NEW = EDHOC-Exporter("OSCORE_Master_Salt",
                               h'', salt_length)
      = EDHOC-KDF(PRK_4x3m, TH_4,
                  "OSCORE_Master_Salt", h'', salt_length)

  }
  else {
    Master Secret Length = < Size of CTX_IN.MasterSecret in bytes >

    MSECRET_NEW = HKDF-Expand-Label(CTX_IN.MasterSecret, Label,
                                    N, Master Secret Length)
                = HKDF-Expand(CTX_IN.MasterSecret, HkdfLabel,
                              Master Secret Length)

    MSALT_NEW = N;
  }

  < Derive CTX_OUT using MSECRET_NEW and MSALT_NEW,
    together with other parameters from CTX_IN >

  Return CTX_OUT;

}

Where HkdfLabel is defined as

struct {
    uint16 length = Length;
    opaque label<7..255> = "oscore " + Label;
    opaque context<0..255> = Context;
} HkdfLabel;
~~~~~~~~~~~
{: #function-update title="Function for deriving a new OSCORE Security Context" artwork-align="center"}

## Establishment of the New OSCORE Security Context # {#ssec-derive-ctx}

This section defines the actual KUDOS procedure performed by two peers to update their OSCORE keying material. Before starting KUDOS, the two peers share the OSCORE Security Context CTX\_OLD. Once completed the KUDOS execution, the two peers agree on a newly established OSCORE Security Context CTX\_NEW.

In particular, each peer contributes by generating a fresh value R1 or R2, and providing it to the other peer. The byte string concatenation of the two values, hereafter denoted as R1 \| R2, is used as input N by the updateCtx() function, in order to derive the new OSCORE Security Context CTX\_NEW. As for any new OSCORE Security Context, the Sender Sequence Number and the replay window are re-initialized accordingly (see {{Section 3.2.2 of RFC8613}}).

Once a peer has successfully derived the new OSCORE Security Context CTX\_NEW, that peer MUST use CTX\_NEW to protect outgoing non KUDOS messages.

Also, that peer MUST terminate all the ongoing observations {{RFC7641}} that it has with the other peer as protected with the old Security Context CTX\_OLD, unless the two peers have explicitly agreed otherwise as defined in {{preserving-observe}}.

Once a peer has successfully decrypted and verified an incoming message protected with CTX\_NEW, that peer MUST discard the old Security Context CTX\_OLD.

KUDOS can be started by the client or the server, as defined in {{ssec-derive-ctx-client-init}} and {{ssec-derive-ctx-server-init}}, respectively. The following properties hold for both the client- and server-initiated version of KUDOS.

* The initiator always offers the fresh value R1.
* The responder always offers the fresh value R2.
* The responder is always the first one deriving the new OSCORE Security Context CTX\_NEW.
* The initiator is always the first one achieving key confirmation, hence able to safely discard the old OSCORE Security Context CTX\_OLD.
* Both the initiator and the responder use the same respective OSCORE Sender ID and Recipient ID. Also, they both preserve and use the same OSCORE ID Context from CTX\_OLD.

The length of the nonces R1 and R2 is application specific. The application needs to set the length of each nonce such that the probability of its value being repeated is negligible. To this end, each nonce is typically at least 8 bytes long.

Once a peer acting as initiator (responder) has sent (received) the first KUDOS message, that peer MUST NOT send a non KUDOS message to the other peer, until having completed the key update process on its side. The initiator completes the key update process when receiving the second KUDOS message and successfully verifying it with the new OSCORE Security Context CTX\_NEW. The responder completes the key update process when sending the second KUDOS message, as protected with the new OSCORE Security Context CTX\_NEW.

<!--
TEXT OBSOLETED BY SAFER HANDLING IN LATER SECTIONS

A client must be ready to receive a non KUDOS response protected with keying material different than that used to protect the corresponding non KUDOS request. This is the case if KUDOS is run between the transmission of a non KUDOS request and the transmission of the corresponding non KUDOS response.
-->

### Client-Initiated Key Update {#ssec-derive-ctx-client-init}

{{fig-message-exchange-client-init}} shows the KUDOS workflow with the client acting as initiator.

~~~~~~~~~~~
                   Client               Server
                (initiator)          (responder)
                     |                    |
Generate R1          |                    |
                     |                    |
CTX_1 =              |                    |
  updateCtx(R1,      |                    |
            CTX_OLD) |                    |
                     |                    |
                     |     Request #1     |
Protect with CTX_1   |------------------->|
                     | OSCORE Option:     | CTX_1 =
                     |   ...              |   updateCtx(R1,
                     |   d flag: 1        |             CTX_OLD)
                     |   ...              |
                     |   ID Detail: R1    | Verify with CTX_1
                     |   ...              |
                     |                    | Generate R2
                     |                    |
                     |                    | CTX_NEW =
                     |                    |   updateCtx(R1|R2,
                     |                    |             CTX_OLD)
                     |                    |
                     |     Response #1    |
                     |<-------------------| Protect with CTX_NEW
CTX_NEW =            | OSCORE Option:     |
  updateCtx(R1|R2,   |   ...              |
            CTX_OLD) |   d flag: 1        |
                     |   ...              |
Verify with CTX_NEW  |   ID Detail: R2    |
                     |   ...              |
Discard CTX_OLD      |                    |
                     |                    |

// The actual key update process ends here.
// The two peers can use the new Security Context CTX_NEW.

                     |                    |
                     |     Request #2     |
Protect with CTX_NEW |------------------->|
                     |                    | Verify with CTX_NEW
                     |                    |
                     |                    | Discard CTX_OLD
                     |                    |
                     |     Response #2    |
                     |<-------------------| Protect with CTX_NEW
Verify with CTX_NEW  |                    |
                     |                    |
~~~~~~~~~~~
{: #fig-message-exchange-client-init title="Client-Initiated KUDOS Workflow" artwork-align="center"}

First, the client generates a random value R1, and uses the nonce N = R1 together with the old Security Context CTX\_OLD, in order to derive a temporary Security Context CTX\_1. Then, the client sends an OSCORE request to the server, protected with the Security Context CTX\_1. In particular, the request has the 'd' flag bit set to 1 and specifies R1 as 'id detail' (see {{ssec-oscore-option-extensions}}).

Upon receiving the OSCORE request, the server retrieves the value R1 from the 'id detail' of the request, and uses the nonce N = R1 together with the old Security Context CTX\_OLD, in order to derive the temporary Security Context CTX\_1. Then, the server verifies the request by using the Security Context CTX\_1.

After that, the server generates a random value R2, and uses the nonce N = R1 \| R2 together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Then, the server sends an OSCORE response to the client, protected with the new Security Context CTX\_NEW. In particular, the response has the 'd' flag bit set to 1 and specifies R2 as 'id detail'.

Upon receiving the OSCORE response, the client retrieves the value R2 from the 'id detail' of the response. Since the client has received a response to an OSCORE request it made with the 'd' flag bit set to 1, the client uses the nonce N = R1 \| R2 together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Finally, the client verifies the response by using the Security Context CTX\_NEW and deletes the old Security Context CTX\_OLD.

After that, the client can send a new OSCORE request protected with the new Security Context CTX\_NEW. When successfully verifying the request using the Security Context CTX\_NEW, the server deletes the old Security Context CTX\_OLD and can reply with an OSCORE response protected with the new Security Context CTX\_NEW.

From then on, the two peers can protect their message exchanges by using the new Security Context CTX\_NEW.

Note that the server achieves key confirmation only when receiving a message from the client as protected with the new Security Context CTX\_NEW. If the server sends a non KUDOS request to the client protected with CTX\_NEW before then, and the server receives a 4.01 (Unauthorized) error response as reply, the server SHOULD delete the new Security Context CTX\_NEW and start a new client-initiated key update process, by taking the role of initiator as per {{fig-message-exchange-client-init}}.

#### Avoiding In-Transit Requests During a Key Update

Before sending the KUDOS message Request #1 in {{fig-message-exchange-client-init}}, the client MUST ensure that it has no ouststanding interactions with the server (see {{Section 4.7 of RFC7252}}), with the exception of ongoing observations {{RFC7641}} with that server.

If there are any, the client MUST NOT initiate the KUDOS execution, before either: i) having all those outstanding interactions cleared; or ii) freeing up the Token values used with those outstanding interactions, with the exception of ongoing observations with the server.

Later on, this prevents a non KUDOS response protected with the new Security Context CTX\_NEW to cryptographically match with both the corresponding request also protected with CTX\_NEW and with an older request protected with CTX\_OLD, in case the two requests were protected using the same OSCORE Partial IV.

<!--
TEXT OBSOLETED BY SAFER HANDLING ABOVE

As mentioned in {{ssec-derive-ctx}}, a client must be ready to receive a non KUDOS response protected with keying material different than that used to protect the corresponding non KUDOS request. When running the client-initiated version of KUDOS as per {{fig-message-exchange-client-init}}, this can happen if the client uses NSTART > 1 (see {{Section 4.7 of RFC7252}}), and the client has outstanding interactions with the server (see {{Section 4.7 of RFC7252}}) when sending the first KUDOS message.

\[NOTE: Another case would be where the client has an observation already ongoing at the server when KUDOS starts. However, this assumes that it is possible to safely preserve observations across a key update, which is not the case at the moment although under consideration.\]
-->

### Server-Initiated Key Update {#ssec-derive-ctx-server-init}

{{fig-message-exchange-server-init}} shows the KUDOS workflow with the server acting as initiator.

~~~~~~~~~~~
                   Client               Server
                (responder)          (initiator)
                     |                    |
                     |     Request #1     |
Protect with CTX_OLD |------------------->|
                     |                    | Verify with CTX_OLD
                     |                    |
                     |                    | Generate R1
                     |                    |
                     |                    | CTX_1 =
                     |                    |   updateCtx(R1,
                     |                    |             CTX_OLD)
                     |                    |
                     |     Response #1    |
                     |<-------------------| Protect with CTX_1
CTX_1 =              | OSCORE Option:     |
  updateCtx(R1,      |   ...              |
            CTX_OLD) |   d flag: 1        |
                     |   ...              |
Verify with CTX_1    |   ID Detail: R1    |
                     |   ...              |
Generate R2          |                    |
                     |                    |
CTX_NEW =            |                    |
  updateCtx(R1|R2,   |                    |
            CTX_OLD) |                    |
                     |                    |
                     |     Request #2     |
Protect with CTX_NEW |------------------->|
                     | OSCORE Option:     | CTX_NEW =
                     |   ...              |   updateCtx(R1|R2,
                     |   d flag: 1        |             CTX_OLD)
                     |   ...              |
                     |   ID Detail: R1|R2 | Verify with CTX_NEW
                     |   ...              |
                     |                    | Discard CTX_OLD
                     |                    |

// The actual key update process ends here.
// The two peers can use the new Security Context CTX_NEW.

                     |     Response #2    |
                     |<-------------------| Protect with CTX_NEW
Verify with CTX_NEW  |                    |
                     |                    |
Discard CTX_OLD      |                    |
                     |                    |

~~~~~~~~~~~
{: #fig-message-exchange-server-init title="Server-Initiated KUDOS Workflow" artwork-align="center"}

First, the client sends a normal OSCORE request to the server, protected with the old Security Context CTX\_OLD and with the 'd' flag bit set to 0.

Upon receiving the OSCORE request and after having verified it with the old Security Context CTX\_OLD as usual, the server generates a random value R1 and uses the nonce N = R1 together with the old Security Context CTX\_OLD, in order to derive a temporary Security Context CTX\_1. Then, the server sends an OSCORE response to the client, protected with the Security Context CTX\_1. In particular, the response has the 'd' flag bit set to 1 and specifies R1 as 'id detail' (see {{ssec-oscore-option-extensions}}).

Upon receiving the OSCORE response, the client retrieves the value R1 from the 'id detail' of the response, and uses the nonce N = R1 together with the old Security Context CTX\_OLD, in order to derive the temporary Security Context CTX\_1. Then, the client verifies the response by using the Security Context CTX\_1.

After that, the client generates a random value R2, and uses the nonce N = R1 \| R2 together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Then, the client sends an OSCORE request to the server, protected with the new Security Context CTX\_NEW. In particular, the request has the 'd' flag bit set to 1 and specifies R1 \| R2 as 'id detail'.

Upon receiving the OSCORE request, the server retrieves the value R1 \| R2 from the request. Then, the server verifies that: i) the value R1 is identical to the value R1 specified in a previous OSCORE response with the 'd' flag bit set to 1; and ii) the value R1 \| R2 has not been received before in an OSCORE request with the 'd' flag bit set to 1. If the verification succeeds, the server uses the nonce N = R1 \| R2 together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Finally, the server verifies the request by using the Security Context CTX\_NEW and deletes the old Security Context CTX\_OLD.

After that, the server can send an OSCORE response protected with the new Security Context CTX\_NEW. When successfully verifying the response using the Security Context CTX\_NEW, the client deletes the old Security Context CTX\_OLD.

From then on, the two peers can protect their message exchanges by using the new Security Context CTX\_NEW.

Note that the client achieves key confirmation only when receiving a message from the server as protected with the new Security Context CTX\_NEW. If the client sends a non KUDOS request to the server protected with CTX\_NEW before then, and the client receives a 4.01 (Unauthorized) error response as reply, the client SHOULD delete the new Security Context CTX\_NEW and start a new client-initiated key update process, by taking the role of initiator as per {{fig-message-exchange-client-init}} in {{ssec-derive-ctx-client-init}}.

#### Avoiding In-Transit Requests During a Key Update

Before sending the KUDOS message Request #2 in {{fig-message-exchange-server-init}}, the client MUST ensure that it has no ouststanding interactions with the server (see {{Section 4.7 of RFC7252}}), with the exception of ongoing observations {{RFC7641}} with that server.

If there are any, the client MUST NOT initiate the KUDOS execution, before either: i) having all those outstanding interactions cleared; or ii) freeing up the Token values used with those outstanding interactions, with the exception of ongoing observations with the server.

Later on, this prevents a non KUDOS response protected with the new Security Context CTX\_NEW to cryptographically match with both the corresponding request also protected with CTX\_NEW and with an older request protected with CTX\_OLD, in case the two requests were protected using the same OSCORE Partial IV.

<!--
TEXT OBSOLETED BY THE SAFER HANDLING ABOVE

As mentioned in {{ssec-derive-ctx}}, a client must be ready to receive a non KUDOS response protected with keying material different than that used to protect the corresponding non KUDOS request. When running the server-initiated version of KUDOS as per {{fig-message-exchange-server-init}}, this can happen if the client uses NSTART > 1 (see {{Section 4.7 of RFC7252}}), and one of the non KUDOS requests results in the server initiating KUDOS (i.e., yielding the first KUDOS message as response). In such a case, the other non KUDOS requests representing oustanding interactions with the server (see {{Section 4.7 of RFC7252}}) would be replied to later on, once the server has finished executing KUDOS (i.e., when the server receives the second KUDOS message, successfully verifies it, and derives the new OSCORE Security Context CTX\_NEW).
-->

#### Preventing Deadlock Situations

When the server-initiated version of KUDOS is used, the two peers risk to run into a deadlock, if all the following conditions hold.

* The client is a client-only device, i.e., it is not capable to act as CoAP server and thus does not listen for incoming requests.

* The server needs to execute KUDOS, which, due to the previous point, can only be performed in its server-initiated version as per {{fig-message-exchange-server-init}}. That is, the server has to wait for an incoming non KUDOS request, in order to initiate KUDOS by replying with the first KUDOS message as a response.

* The client sends only Non-confirmable CoAP requests to the server and does not expect responses sent back as reply, hence freeing up a request's Token value once the request is sent.

In such a case, in order to avoid experiencing a deadlock situation where the server needs to execute KUDOS but cannot practically initiate it, a client-only device that supports KUDOS SHOULD intersperse Non-confirmable requests it sends to that server with confirmable requests.

## Retention Policies # {#ssec-retention}

Applications MAY define policies that allows a peer to also temporarily keep the old Security Context CTX\_OLD, rather than simply overwriting it to become CTX\_NEW. This allows the peer to decrypt late, still on-the-fly incoming messages protected with CTX\_OLD.

When enforcing such policies, the following applies.

* Outgoing non KUDOS messages MUST be protected by using only CTX\_NEW.

* Incoming non KUDOS messages MUST first be attempted to decrypt by using CTX\_NEW. If decryption fails, a second attempt can use CTX\_OLD.

* When an amount of time defined by the policy has elapsed since the establishment of CTX\_NEW, the peer deletes CTX\_OLD.

## Discussion # {#ssec-discussion}

KUDOS is intended to deprecate and replace the procedure defined in {{Section B.2 of RFC8613}}, as fundamentally achieving the same goal, while displaying a number of improvements and advantages.

In particular, it is especially convenient for the handling of failure events concerning the JRC node in 6TiSCH networks (see {{sec-current-methods}}). That is, among its intrinsic advantages compared to the procedure defined in {{Section B.2 of RFC8613}}, KUDOS preserves the same ID Context value, when establishing a new OSCORE Security Context.

Since the JRC uses ID Context values as identifiers of network nodes, namely "pledge identifiers", the above implies that the JRC does not have anymore to perform a mapping between a new, different ID Context value and a certain pledge identifier (see {{Section 8.3.3 of RFC9031}}). It follows that pledge identifiers can remain constant once assigned, and thus ID Context values used as pledge identifiers can be employed in the long-term as originally intended.

# Security Considerations

This document mainly covers security considerations about using AEAD keys in OSCORE and their usage limits, in addition to the security considerations of {{RFC8613}}.

Depending on the specific key update procedure used to establish a new OSCORE Security Context, the related security considerations also apply.

\[TODO: Add more considerations.\]

# IANA Considerations

RFC Editor: Please replace "\[this document\]" with the RFC number of this document and delete this paragraph.

This document has the following actions for IANA.

## CoAP Option Numbers Registry ## {#iana-coap-options}

IANA is asked to enter the following option number to the "CoAP Option Numbers" registry within the "CoRE Parameters" registry group.

~~~~~~~~~~~
+--------+--------------+-----------------+
| Number |     Name     |    Reference    |
+--------+--------------+-----------------+
|  TBD   | Recipient-ID | [this document] |
+--------+--------------+-----------------+
~~~~~~~~~~~
{: artwork-align="center"}

The number suggested to IANA for the Recipient-ID option is 24.

## OSCORE Flag Bits Registry {#iana-cons-flag-bits}

IANA is asked to add the following entries to the "OSCORE Flag Bits" registry within the "Constrained RESTful Environments (CoRE) Parameters" registry group.

~~~~~~~~~~~
+----------+------------------+------------------------+-----------+
| Bit      |       Name       |      Description       | Reference |
| Position |                  |                        |           |
+----------+------------------+------------------------+-----------+
|    1     | Extension-1 Flag | Set to 1 if the OSCORE | [this     |
|          |                  | Option specifies a     | document] |
|          |                  | second byte of OSCORE  |           |
|          |                  | flag bits              |           |
+----------+------------------+------------------------+-----------+
|    15    |  ID Detail Flag  | Set to 1 if the        | [this     |
|          |                  | compressed COSE object | document] |
|          |                  | contains 'id detail'   |           |
+----------+------------------+------------------------+-----------+
~~~~~~~~~~~

--- back

# Detailed considerations for AEAD_AES_128_CCM_8 # {#aead-aes-128-ccm-8-details}

For the AEAD_AES_128_CCM_8 algorithm when used as AEAD Algorithm for OSCORE, larger IA and CA values are achieved, depending on the value of 'q', 'v' and 'l'. {{algorithm-limits-ccm8}} shows the resulting IA and CA probabilities enjoyed by AEAD_AES_128_CCM_8, when taking different values of 'q', 'v' and 'l' as input to the formulas defined in {{I-D.irtf-cfrg-aead-limits}}.

As shown in {{algorithm-limits-ccm8}}, it is especially possible to achieve the lowest IA = 2^-50 and a good CA = 2^-70 by considering the largest possible value of the (q, v, l) triplet equal to (2^20, 2^10, 2^8), while still keeping a good security level. Note that the value of 'l' does not impact on IA, while CA displays good values for every considered value of 'l'.

~~~~~~~~~~~
+-----------------------+----------------+----------------+
| 'q', 'v' and 'l'      | IA probability | CA probability |
|-----------------------+----------------+----------------|
| q=2^20, v=2^20, l=2^8 | 2^-44          | 2^-70          |
| q=2^15, v=2^20, l=2^8 | 2^-44          | 2^-80          |
| q=2^10, v=2^20, l=2^8 | 2^-44          | 2^-90          |
| q=2^20, v=2^15, l=2^8 | 2^-49          | 2^-70          |
| q=2^15, v=2^15, l=2^8 | 2^-49          | 2^-80          |
| q=2^10, v=2^15, l=2^8 | 2^-49          | 2^-90          |
| q=2^20, v=2^14, l=2^8 | 2^-50          | 2^-70          |
| q=2^15, v=2^14, l=2^8 | 2^-50          | 2^-80          |
| q=2^10, v=2^14, l=2^8 | 2^-50          | 2^-90          |
| q=2^20, v=2^10, l=2^8 | 2^-54          | 2^-70          |
| q=2^15, v=2^10, l=2^8 | 2^-54          | 2^-80          |
| q=2^10, v=2^10, l=2^8 | 2^-54          | 2^-90          |
|-----------------------+----------------+----------------|
| q=2^20, v=2^20, l=2^6 | 2^-44          | 2^-74          |
| q=2^15, v=2^20, l=2^6 | 2^-44          | 2^-84          |
| q=2^10, v=2^20, l=2^6 | 2^-44          | 2^-94          |
| q=2^20, v=2^15, l=2^6 | 2^-49          | 2^-74          |
| q=2^15, v=2^15, l=2^6 | 2^-49          | 2^-84          |
| q=2^10, v=2^15, l=2^6 | 2^-49          | 2^-94          |
| q=2^20, v=2^14, l=2^6 | 2^-50          | 2^-74          |
| q=2^15, v=2^14, l=2^6 | 2^-50          | 2^-84          |
| q=2^10, v=2^14, l=2^6 | 2^-50          | 2^-94          |
| q=2^20, v=2^10, l=2^6 | 2^-54          | 2^-74          |
| q=2^15, v=2^10, l=2^6 | 2^-54          | 2^-84          |
| q=2^10, v=2^10, l=2^6 | 2^-54          | 2^-94          |
+-----------------------+----------------+----------------+
~~~~~~~~~~~
{: #algorithm-limits-ccm8 title="Probabilities for AEAD_AES_128_CCM_8 based on chosen q, v and l values." artwork-align="center"}

# Estimation of 'count_q' # {#estimation-count-q}

This section defines a method to compute an estimate of the counter 'count_q' (see {{sender-context}}), hence not requiring a peer to store it in its own Sender Context.

This method relies on the fact that, at any point in time, a peer has performed _at most_ ENC = (SSN + SSN\*) encryptions using its own Sender Key, where:

* SSN is the current value of this peer's Sender Sequence Number.

* SSN\* is the current value of other peer's Sender Sequence Number. That is, SSN\* is an overestimation of the responses without Partial IV that this peer has sent.

Thus, when protecting an outgoing message (see {{protecting-req-resp}}), the peer aborts the message processing if the estimated est\_q > limit\_q, where est\_q = (SSN + X) and X is determined as follows.

* If the outgoing message is a response, X is the Partial IV specified in the corresponding request that this peer is responding to. Note that X < SSN\* always holds.

* If the outgoing message is a request, X is the highest Partial IV value marked as received in this peer's Replay Window plus 1, or 0 if it has not accepted any protected message from the other peer yet. That is, X is the highest Partial IV specified in message received from the other peer, i.e., the highest seen Sender Sequence Number of the other peer. Note that, also in this case, X < SSN\* always holds.

# Preserving Observations across Key Updates # {#preserving-observe}

As defined in {{ssec-derive-ctx}}, once a peer has completed the KUDOS execution and successfully derived the new OSCORE Security Context CTX\_NEW, that peer normally terminates all the ongoing observations it has with the other peer {{RFC7641}}, as protected with the old Security Context CTX\_OLD.

This section describes a method that the two peers can use to safely preserve the ongoing observations that they have with one another, after having completed a KUDOS execution. In particular, this method ensures that an Observe notification can never successfully match against the Observe requests of two different observations, i.e., an Observe request protected with CTX\_OLD and an Observe request protected with CTX\_NEW.

The actual preservation of ongoing observations has to be agreed by the two peers at each execution of KUDOS that they run with one another, as defined in {{preserving-observe-signaling}}. If, at the end of a KUDOS execution, the two peers have not agreed on that, they MUST terminate the ongoing observations that they have with one another, as defined in {{ssec-derive-ctx}}.

If a peer supporting KUDOS is generally interested in preserving ongoing observations across a key update, the peer maintains a counter EPOCH for each ongoing observation it participates in. At any point in time, (EPOCH + 1) is the number of KUDOS execution performed by the peer since the sucessful registration of the associated observation. That is, EPOCH indicates the lifetime of an observation measured in keying material epochs, and is bounded by the configuration parameter MAX\_EPOCH.

\[NOTE: MAX\_EPOCH really has to be the same for any two peers. As a start, it can be assumed that a TBD default value applies, unless a different one is provided. It is possible to unable an actual negotiation between two peers running KUDOS, see {{preserving-observe-signaling}}.\]

The following sections specify the different actions taken by the peer depending on whether it acts as client or server in an ongoing observation, as well as the signaling method used in KUDOS to agree on preserving the ongoing observations beyond the current KUDOS execution.

\[NOTE: This method may be of more general applicability, i.e, also in case an update of the OSCORE keying material is performed through a different means than KUDOS.\]

## Management of Observations

As per {{Section 3.1 of RFC7641}}, a client can register its interest in observing a resource at a server, by sending a registration request including the Observe option with value 0.

If the server sends back a successful response also including the Observe option, hence confirming that the observation has been registered, then the server initializes to 0 the counter EPOCH associated with the just confirmed observation.

If the client receives back the successful response from the server, then the client initializes to 0 the counter EPOCH associated with the just confirmed observation.

If, later on, the client is not interested in the observation anymore, it MUST NOT simply forget about it. Rather, the client MUST send an explicit cancellation request to the server, i.e., a request including the Observe option with value 1 (see {{Section 3.6 of RFC7641}}). After sending this cancellation request, if the client does not receive back a response confirming that the observation has been terminated, the client A MUST NOT consider the observation terminated. The client MAY try again to terminate the observation by sending a new cancellation request.

In case a peer A performs a KUDOS execution with another peer B, and A has ongoing observations with B that it is interested to preserve across the key update, then A explicitly opts-in by using the signaling approach embedded in KUDOS and defined in {{preserving-observe-signaling}}.

After having successfully completed the KUDOS execution (i.e., after having successfully derived the new OSCORE Security Context CTX\_NEW), if the other peer B has confirmed its interest in preserving those ongoing observations also by using the signaling approach defined in {{preserving-observe-signaling}}, then the peer A performs the following actions.

1. For each ongoing observation X that A has with B:

   a. The peer A increments the counter EPOCH associated with X.

   b. If the updated value of EPOCH associated with X has reached MAX\_EPOCH, then the peer A MUST terminate the observation.

      If the peer A was acting as client in the observation X, then it SHOULD send an explicit cancellation request to the other peer B, i.e., a request including the Observe option with value 1 (see {{Section 3.6 of RFC7641}}).

2. For each still ongoing observation X that A has with B after the previous step, such that A acts as client in X:

   a. The peer A considers all the OSCORE Partial IV values used in the Observe registration request associated with any of the still ongoing observations with the other peer B.

   b. The peer A determines the value PIV\* as the highest OSCORE Partial IV among those considered at the previous step.

   c. In the Sender Context of the OSCORE Security Context shared with the other peer B, the peer A sets its own Sender Sequence Number to (PIV\* + 1), rather than to 0.

## Signaling to Preserve Observations # {#preserving-observe-signaling}

When performing KUDOS, a peer can indicate to the other peer its interest in  preserving the ongoing observations that they have with one another. To this end, the OSCORE option shown in {{fig-oscore-option}} and included in a KUDOS message is further extended as follows.

\[
NOTE: This is an early proposal with many details to be refined.
\]

An additional bit "Extend Observations", 'b', is set to 1 by the sender peer to indicate that it wishes to preserve ongoing observations with the other peer.

While 'b' can be a bit in the second byte of the OSCORE option containing the OSCORE flag bits, 'b' can rather be one bit in the 1 byte 'x' following 'kid context' (if any) and originally encoding the size of 'id detail'. Since, the recommended size of 'id detail' is 8 bytes, the number of bits left available in the 'x' bit is amply sufficient to still indicate the size of 'id detail'.

If is fundamental to integrity-protect the value of the bit 'b' set in the two KUDOS messages. This can be achieved by taking also the whole byte 'x' including the the bit 'b' as input in the derivation of the new OSCORE Security Context CTX\_NEW.

That is, the updateCtx() function defined in {{function-update}} would be invoked as follows:

* CTX\_1 = updateCtx(X1\|R1, CTX\_OLD), when deriving CTX\_1 for processing the first KUDOS message in the KUDOS execution.

* CTX\_NEW = updateCtx(X1\|X2\|R1\|R2, CTX\_OLD), when deriving CTX\_NEW for processing the second KUDOS message in the KUDOS execution.

where X1 and X2 are the values of the 'x' byte specified in the OSCORE option of the the first and second KUDOS message in the KUDOS execution, respectively.

\[

NOTE: The single bit 'b' can actually be replaced by three bits 'b1', 'b2' and 'b3' still within the byte 'x'. These can be used by the two peers performing KUDOS to negotiate the value of MAX\_EPOCH (see {{preserving-observe}}. Then, the two peers agree to use as MAX\_EPOCH the smallest of the two values exchanged during the execution of KUDOS.

The final encoding of the 'x' byte specified in {{ssec-oscore-option-extensions}} will be affected.

\]

# Update of OSCORE Sender/Recipient IDs # {#update-oscore-ids}

This section defines an optional procedure that two peers can execute to update the OSCORE Sender/Recipient IDs that they use in their shared OSCORE Security Context.

This procedure can be initiated by either peer. In particular, the client or the server may start it by sending the first OSCORE ID update message. When sending an OSCORE ID update message, a peer provides its new intended OSCORE Recipient ID to the other peer.

Furthermore, this procedure can be executed stand-alone, or rather seamlessly integrated in an execution of KUDOS (see {{sec-rekeying-method}}).

* In the former stand-alone case, updating the OSCORE Sender/Recipient IDs effectively results in updating part of the current OSCORE Security Context.

   That is, a new Sender Key, Recipient Key and Common IV are derived as defined in {{Section 3.2 of RFC8613}}. Also, the Sender Sequence Number and the replay window are re-initialized accordingly, as defined in {{Section 3.2.2 of RFC8613}}. Since the same Master Secret is preserved, perfect forward secrecy is not achieved.

   Finally, as defined in {{id-update-additional-actions}}, the two peers must take additional actions to ensure a safe execution of the OSCORE IDs update procedure.

* In the latter integrated case, the KUDOS initiator (responder) also acts as initiator (responder) for the ID update procedure.

\[TODO: think about the possibility of safely preserving ongoing observations following an update of OSCORE IDs alone.\]

## The Recipient-ID Option # {#sec-recipient-id-option}

The Recipient ID Option defined in this section has the properties summarized in {{fig-recipient-id-option}}, which extends Table 4 of {{RFC7252}}. That is, the option is elective, safe to forward, part of the cache key and non repeatable.

~~~~~~~~~~~
+------+---+---+---+---+--------------+--------+--------+---------+
| No.  | C | U | N | R | Name         | Format | Length | Default |
+------+---+---+---+---+--------------+--------+--------+---------+
|      |   |   |   |   |              |        |        |         |
| TBD1 |   |   |   |   | Recipient-ID | opaque |  0-7   | (none)  |
|      |   |   |   |   |              |        |        |         |
+------+---+---+---+---+--------------+--------+--------+---------+
         C=Critical, U=Unsafe, N=NoCacheKey, R=Repeatable
~~~~~~~~~~~
{: #fig-recipient-id-option title="The Recipient-ID Option." artwork-align="center"}

This document particularly defines how this option is used in messages protected with OSCORE. That is, when the option is included in an outgoing message, the option value specifies the new OSCORE Recipient ID that the sender endpoint intends to use with the other endpoint sharing the OSCORE Security Context.

The Recipient-ID Option is of class E in terms of OSCORE processing (see {{Section 4.1 of RFC8613}}).

### Client-Initiated OSCORE IDs Update {#example-client-initiated-id-update}

{{fig-id-update-client-init}} shows the stand-alone OSCORE IDs update workflow, with the client acting as initiator.

On each peer, SID and RID denote the OSCORE Sender ID and Recipient ID of that peer, respectively.

~~~~~~~~~~~
          Client                             Server
       (initiator)                         (responder)
            |                                   |
CTX_A {     |                                   | CTX_A {
 SID = 1    |                                   |  SID = 0
 RID = 0    |                                   |  RID = 1
}           |                                   | }
            |                                   |
            |            Request #1             |
Protect     |---------------------------------->|
with CTX_A  | OSCORE Option: ..., kid:1         | Verify
            | Encrypted_Payload {               | with CTX_A
            |    ...                            |
            |    RecipientID: 42                |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |

          // When embedded in KUDOS, CTX_1 is CTX_A,
          // and there cannot be application payload.

            |                                   |
            |            Response #1            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_A
with CTX_A  | Encrypted_Payload {               |
            |    ...                            |
            |    Recipient-ID: 78               |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |

           // When embedded in KUDOS, this message
           // is protected using CTX_NEW, and there
           // there cannot be application payload.
           //
           // Then, CTX_B builds on CTX_NEW by updating
           // the new Sender/Recipient IDs

            |                                   |
CTX_B {     |                                   | CTX_B {
 SID = 78   |                                   |  SID = 42
 RID = 42   |                                   |  RID = 78
}           |                                   | }
            |                                   |
            |            Request #2             |
Protect     |---------------------------------->|
with CTX_B  | OSCORE Option: ..., kid:78        | Verify
            | Encrypted_Payload {               | with  CTX_B
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
            |            Response #2            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_B
with CTX_B  | Encrypted_Payload {               |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
Discard     |                                   |
CTX_A       |                                   |
            |                                   |
            |            Request #3             |
Protect     |---------------------------------->|
with CTX_B  | OSCORE Option: ..., kid:78        | Verify
            | Encrypted_Payload {               | with CTX_B
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   | Discard
            |                                   | CTX_A
            |                                   |
~~~~~~~~~~~
{: #fig-id-update-client-init title="Client-Initiated OSCORE IDs Update Workflow" artwork-align="center"}

\[TODO: discuss the example\]

### Server-Initiated OSCORE IDs Update {#example-server-initiated-id-update}

{{fig-id-update-server-init}} shows the stand-alone OSCORE IDs update workflow, with the server acting as initiator.

On each peer, SID and RID denote the OSCORE Sender ID and Recipient ID of that peer, respectively.

~~~~~~~~~~~
          Client                             Server
       (responder)                         (initiator)
            |                                   |
CTX_A {     |                                   | CTX_A {
 SID = 1    |                                   |  SID = 0
 RID = 0    |                                   |  RID = 1
}           |                                   | }
            |                                   |
            |            Request #1             |
Protect     |---------------------------------->|
with CTX_A  | OSCORE Option: ..., kid:1         | Verify
            | Encrypted_Payload {               | with CTX_A
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |

          // When (to be) embedded in KUDOS,
          // CTX_OLD is CTX_A

            |                                   |
            |            Response #1            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_A
with CTX_A  | Encrypted_Payload {               |
            |    ...                            |
            |    Recipient-ID: 78               |
            |    Application Payload            |
            | }                                 |

          // When embedded in KUDOS, this message is
          // protected with CTX_1 instead, and
          // there cannot be application payload.

            |                                   |
CTX_A {     |                                   | CTX_A {
 SID = 1    |                                   |  SID = 0
 RID = 0    |                                   |  RID = 1
}           |                                   | }
            |                                   |
            |            Request #2             |
Protect     |---------------------------------->|
with CTX_A  | OSCORE Option: ..., kid:1         | Verify
            | Encrypted_Payload {               | with CTX_A
            |    ...                            |
            |    Recipient-ID: 42               |
            |    Application Payload            |
            | }                                 |
            |                                   |

          // When embedded in KUDOS, this message is
          // protected with CTX_NEW instead, and
          // there cannot be application payload.

            |                                   |
            |            Response #2            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_A
with CTX_A  | Encrypted_Payload {               |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |

          // When embedded in KUDOS, this message is
          // protected with CTX_NEW instead, and
          // there cannot be application payload.

            |                                   |
CTX_B {     |                                   | CTX_B {
 SID = 78   |                                   |  SID = 42
 RID = 42   |                                   |  RID = 78
}           |                                   | }
            |                                   |
            |            Request #3             |
Protect     |---------------------------------->|
with CTX_B  | OSCORE Option: ..., kid:78        | Verify
            | Encrypted_Payload {               | with CTX_B
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
            |            Response #3            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_B
with CTX_B  | Encrypted_Payload {               |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
Discard     |                                   |
CTX_A       |                                   |
            |                                   |
            |            Request #4             |
Protect     |---------------------------------->|
with CTX_B  | OSCORE Option: ..., kid:78        | Verify
            | Encrypted_Payload {               | with CTX_B
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
            |                                   | Discard
            |                                   | CTX_A
            |                                   |
            |            Response #4            |
            |<----------------------------------| Protect
Verify      | OSCORE Option: ...                | with CTX_B
with CTX_B  | Encrypted_Payload {               |
            |    ...                            |
            |    Application Payload            |
            | }                                 |
            |                                   |
~~~~~~~~~~~
{: #fig-id-update-server-init title="Server-Initiated OSCORE IDs Update Workflow" artwork-align="center"}

\[TODO: discuss the example\]

### Additional Actions for Stand-Alone Execution {#id-update-additional-actions}

After having experienced a loss of state, a peer MUST NOT participate in a stand-alone OSCORE IDs update procedure with another peer, until having performed a full-fledged establishment/renewal of an OSCORE Security Context with the other peer (e.g., through KUDOS or EDHOC {{I-D.ietf-lake-edhoc}}).

More precisely, a peer has experienced a loss of state if it cannot access the latest snapshot of the latest OSCORE Security Context CTX\_OLD or the whole set of OSCORE Sender/Recipient IDs that have been used with the triplet (Master Secret, Master Salt ID Context) of CTX\_OLD. This can happen, for instance, following a device reboot.

Furthermore, when participating in a stand-alone OSCORE IDs update procedure, a peer perform the following additional steps.

* When sending an OSCORE ID update message, the peer MUST specify its new intended OSCORE Recipient ID as value of the Recipient-ID option only if such a Recipient ID is not only available (see {{Section 3.3 of RFC8613}}, but it has also never been used as Recipient ID with the current triplet (Master Secret, Master Salt ID Context).

* When receiving an OSCORE ID update message, the peer MUST abort the procedure if it has already used the identifier specified in the Recipient-ID Option as its own Sender ID with current triplet (Master Secret, Master Salt ID Context).

In order to fulfill the conditions above, a peer has to keep track of the OSCORE Sender/Recipient IDs that it has used with the current triplet (Master Secret, Master Salt ID Context), since the latest update of OSCORE Master Secret (e.g, performed through KUDOS).

# KUDOS procedure without Forward Secrecy {#no-fs-mode}

The KUDOS procedure as defined in section {{sec-rekeying-method}} ensures Forward Secrecy of the keying material after the procedure has completed. However, this original version of KUDOS can be problematic for devices that can not dynamically write information to non-volatile memory. That is, they can afford only a single writing in persistent memory when initial key material is provided (e.g., at manufacturing), but not more after that. These devices cannot perform a stateful key update procedure, which practically prevents guaranteeing Forward Secrecy, and running the original version of KUDOS (as defined with Forward Secrecy).

Due to the need to support also devices that cannot write to non-volatile memory, it becomes neccessary to define a way of running KUDOS that no longer guarantees Forward Secrecy, but allows devices which are not capable of storing information to persistant storage to nontheless use KUDOS. This is an alternative execution of KUDOS, which sacrifices FS but allows devices to perform a stateless key update, i.e., without writing on disk (which is possible using the current OSCORE Appendix B.2). This section defines such a method which is called the non-FS mode of KUDOS.

Formally the requirements for running KUDOS with Forward Secrecy are the following. After KUDOS has successfully completed:
- The Master Secret and Master Salt are updated (in CTX_NEW), and keys derived from the "original" one (in CTX_OLD) are not used anymore
- The new Master Secret and Master Salt are stored in non-volatile memory, for retrieval after loss off state (e.g. rebooting)

If both peers do not fulfill the above requirements the non-FS mode of KUDOS must be used.

## Concepts

This section introduces a number of concepts that are used when describing how the non-FS mode of KUDOS operates.

Devices which are able to store information to non-volatible memory are CAPABLE whichs implies the following: The device is generally capable of writing to disk (non-volatile memory). This excludes any one-time only writing in non-volatile memory happening at manufacturing time or (re-)commissioning time, e.g., to write the Bootstrap Master Secret and Bootstrap Master Salt.

Furthermore the following types of key material is defined:
- Bootstrap Master Secret and Bootstrap Master Salt
    - If provisioned they are stored on disk, and they are never changed by the device.
- Latest Master Secret and Latest Master Salt
    - Can be dynamically updated by the device; they are lost upon reboot unless stored on disk.

Note that:
- A device can have none of the pairs above, only one or both.
- A device that has neither of the above pairs, cannot run KUDOS.
- A device that has only one of the above pairs can attempt to run KUDOS, but that can fail due to the other peer's capabilities. (Practically, in order to use the FS mode of KUDOS both peers must be CAPABLE).

## Workflow after reboot

This section describes the overall procedure a device should follow after it has lost state (e.g. due to a reboot). As a general rule, when generating a new Security Context, the corresponding Latest Master Secret and Latest Master Salt:
- should be stored on disk if the device is CAPABLE;
- must always be stored in volatile memory for practical use with OSCORE

This is independent of how exactly such Master Secret and Master Salt have been obtained (e.g. KUDOS or EDHOC). An exception to the above is the temporary KUDOS context CTX_1 which must not be stored on disk.

This enables the following sequence of event in case of rebooting:

- The device checks if it has a (Latest Master Secret, and Latest Master Salt) on disk

- If yes:
    - Load it to volatile memory, and use its content to derive an OSCORE context CTX_OLD
    - Run KUDOS as initiator
        - If CAPABLE store on disk the Master Secret and Master Salt from CTX_NEW as Latest Master Secret, and Latest Master Salt

- If no, the device checks if it has a (Bootstrap Master Secret, and Bootstrap Master Salt) on disk

    - If yes:
        - Load it to volatile memory, and use its content to derive an OSCORE context CTX_OLD
        - If CAPABLE, the device stores (Bootstrap Master Secret, and Bootstrap Master Salt) on disk as (Latest Master Secret, and Latest Master Salt). (This is to support  the case of a CAPABLE device that has not run KUDOS with the other peer yet.)
        - Run KUDOS as initiator
            - If CAPABLE, store on disk the Master Secret and Master Salt from CTX_New as (Latest Master Secret, Latest Master Salt).

    - If no, use alternative ways to establish a first OSCORE context CTX_NEW, e.g., EDHOC.
        - If CAPABLE, store on disk the Master Secret and Master Salt from CTX_NEW as (Latest Master Secret, Latest Master Salt).

## Signaling

In order for the devices to signal whether the FS or non-FS mode of KUDOS is being used for a specific execution, a method for signaling is needed. This section defines such a signaling method by utilizing a bit 'p' which when set to 0 indicates usage of the original version of KUDOS (with FS), and when set to 1 indicates usage of the non-FS mode of KUDOS.

That is, the 'p' bit is defined and used as follows:

- The 'p' bit to indicate FS or no-FS mode is the left-most bit of the 'x' field intended to signal the size of the 'id detail' field. Specifically 1 bit of the 8 bits in the 'x' field is reserved for the signaling bit 'p'. (Leaving 7 bits to indicate the size of the 'id detail' field, which still ensures the possibility of more than large (secure) enough nonces R1 and R2.)
- The bit must be set to 0 when using the original version of KUDOS. The bit must be set to 0 if the second byte of flag bits is present but the 'd' flag is set to 0 (the message is not a KUDOS message). The bit must be set to 1 when using the non-FS mode of KUDOS.
- In a KUDOS message (i.e., the 'd' bit is set to 1), the 'p' bit indicates what material to use for CTX_OLD which is used as input to updateCtx():
    - If the 'p' bit is set to 0, KUDOS is run in PFS mode. That is, the current Security Context CTX_OLD is used and the goal is to preserve FS. That is, the Security Context CTX_OLD to use is the current one where the following changes apply: Master Secret = Latest Master Secret, and Master Salt = Latest Master Salt. In order to use this mode of KUDOS, a device must be CAPABLE.
    - If the 'p' bit is set to 1, KUDOS is run in no-FS mode, meaning that FS is sacrificed as a stateful execution is not possible. That is, the Security Context CTX_OLD to use is the current one where the following changes apply: Master Secret = Bootstrap Master Secret, and Master Salt = Bootstrap Master Salt. Due to this every execution of KUDOS between these peers will always consider this same Master Secret/Master Salt pair.
        - In order to use this mode of KUDOS a peer must have Bootstrap Master Secret and Bootstrap Master Salt.

Note that in this manner the 'x' field will also be an input to the updateCtx() method, which ensures that the content of the bit is used for deriving key material. Through these means the bit 'p' will be not be possible to modify in transit successfully. Specifically, to avoid inconsistencies (e.g., N1 and N2 have different sizes), updateCtx() takes as input parameters, in addition to CTX_OLD, both the 'x' byte from the first KUDOS message and the 'x' byte from the second KUDOS message. That is, for a client-initiated execution  Request #1 and Response #1, and for a server-initiated exectution Response#1 and Request #2.

## Selection of KUDOS mode

The following section describes instructions for how devices should choose a mode of KUDOS to use.

If a device is non CAPABLE, it MUST NOT run KUDOS in FS mode and MUST run KUDOS in non-FS mode.

If a device is CAPABLE, it SHOULD run KUDOS in FS mode as initiator and SHOULD NOT run KUDOS in no-PFS mode as initiator. An exception to this is a second attempt with a responding peer that has made evident to not support the FS mode. Note that such a CAPABLE device is able to store the knowledge that its peer can only run the non-FS mode of KUDOS, thus it will perform following executions of KUDOS with this peer with the 'p' bit set to 1, including after a possible reboot.

If a peer A has learned that the other peer B does support running KUDOS in FS-mode it should never run KUDOS with that peer B in non-FS mode (if the other peer B initiates KUDOS with p = 1 it should be rejected). If A is a CAPABLE device, it MUST store the information that peer B supporst the FS mode on disk, hence preventing malevolent downgrading to no-PFS mode is case of simultaneous rebooting where B is non capable. Since peer A has learnt that peer B is capable of running KUDOS in FS mode, it will never run KUDOS with peer B using the non-FS mode of KUDOS, and thus such a downgrading attack is not possible.

Note that if both peers reboot simultanously, the client initiated variant of KUDOS would end up being run. This is because the client would first send KUDOS Request #1 which would initiate the procedure and induce the server to respond with Response #1. There is no opportunity for the server to initiate the procedure as the client acts first.

If able to run KUDOS as specified in the 'p' flag by the initiator, the responder MUST comply and do so.

## Negotiation and errors due to mismatched mode

When running KUDOS, it must be ensured that if both peers are CAPABLE, KUDOS is ran in its original FS mode. If both peers are not CAPABLE devices, the initiator will use KUDOS in non-FS mode and the KUDOS execution will be ran in non-FS mode. However, if one device is CAPABLE, and the other devices is not, specific steps are required to be taken. The following section describes what steps a device should take in case the responding devices is non-CAPABLE, and the initiating devices is CAPABLE and initiates KUDOS with the FS mode.

That is, if the initiator sends the first KUDOS message in the procedure (Request #1 for the client-initiated procedure or Response #1 for the server-initiated procedure), with the 'p' bit set to 0 and the responder is non-CAPABLE:
* If the responder is the server,

    * It MUST return a protected 5.03 error response to Request #1 (protected with CTX_NEW), with an explanatory diagnostic payload. The 'p' bit in this response MUST be set to 1. When the initiating client receives this, if 'p' was 0 in the first Request #1, the client learns that the server can run only the no-FS mode and MAY try again, setting the 'p' bit to 1 in the new Request #1.

* If the responder is the client,

   * After receiving Response #1, the client sends a Request #2 protected with CTX_NEW as usual. The 'p' bit in this response MUST be set to 1. Then, the client aborts the current KUDOS execution, and deletes both CTX_1 and CTX_NEW.

   * After receiving the Request #2 above (i.e., it having the 'p' bit set to 1 as a follow-up to Response #1 having the 'p' bit set to 0), the server aborts the KUDOS execution, deletes both CTX_1 and CTX_NEW, and thus learns that the client can run only the no-PFS mode.


# Document Updates # {#sec-document-updates}

RFC EDITOR: PLEASE REMOVE THIS SECTION.

## Version -00 to -01 ## {#sec-00-01}

* Recommendation on limits for CCM_8. Details in Appendix.

* Improved message processing, also covering corner cases.

* Example of method to estimate and not store 'count_q'.

* Procedure to update OSCORE Sender/Recipient IDs.

# Acknowledgments # {#acknowledgments}
{: numbered="no"}

The authors sincerely thank Christian Amsüss, John Mattsson and Göran Selander for their feedback and comments.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
