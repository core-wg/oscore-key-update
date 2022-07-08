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

The term "KUDOS mode" is used to denote two different modes of running KUDOS, that is preserving Forward Secrecy (FS mode) (see {{ssec-update-function}}) or not preserving Forward Secrecy (no-FS mode) (see {{no-fs-mode}}).

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

   This procedure does not require any additional components to what OSCORE already provides, and it does not provide forward secrecy.

   The procedure defined in {{Section B.2 of RFC8613}} is used in 6TiSCH networks {{RFC7554}}{{RFC8180}} when handling failure events. That is, a node acting as Join Registrar/Coordinator (JRC) assists new devices, namely "pledges", to securely join the network as per the Constrained Join Protocol {{RFC9031}}. In particular, a pledge exchanges OSCORE-protected messages with the JRC, from which it obtains a short identifier, link-layer keying material and other configuration parameters. As per {{Section 8.3.3 of RFC9031}}, a JRC that experiences a failure event may likely lose information about joined nodes, including their assigned identifiers. Then, the reinitialized JRC can establish a new OSCORE Security Context with each pledge, through the procedure defined in {{Section B.2 of RFC8613}}.

* The two peers can run the OSCORE profile {{I-D.ietf-ace-oscore-profile}} of the Authentication and Authorization for Constrained Environments (ACE) Framework {{I-D.ietf-ace-oauth-authz}}.

  When a CoAP client uploads an Access Token to a CoAP server as an access credential, the two peers also exchange two nonces. Then, the two peers use the two nonces together with information provided by the ACE Authorization Server that issued the Access Token, in order to derive an OSCORE Security Context.

  This procedure does not provide forward secrecy.

* The two peers can run the EDHOC key exchange protocol based on Diffie-Hellman and defined in {{I-D.ietf-lake-edhoc}}, in order to establish a pseudo-random key in a mutually authenticated way.

   Then, the two peers can use the established pseudo-random key to derive external application keys. This allows the two peers to securely derive especially an OSCORE Master Secret and an OSCORE Master Salt, from which an OSCORE Security Context can be established.

   This procedure additionally provides forward secrecy.

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

* The new OSCORE Security Context enjoys forward secrecy.

* The same ID Context value used in the old OSCORE Security Context is preserved in the new Security Context. Furthermore, the ID Context value never changes throughout the KUDOS execution.

* KUDOS is robust against a peer rebooting, and it especially avoids the reuse of AEAD (nonce, key) pairs.

* KUDOS completes in one round trip. The two peers achieve mutual proof-of-possession in the following exchange, which is protected with the newly established OSCORE Security Context.

## Extensions to the OSCORE Option # {#ssec-oscore-option-extensions}

In order to support the message exchange for establishing a new OSCORE Security Context as defined in {{ssec-derive-ctx}}, this document extends the use of the OSCORE option originally defined in {{RFC8613}} as follows.

* This document defines the usage of the seventh least significant bit, called "Extension-1 Flag", in the first byte of the OSCORE option containing the OSCORE flag bits. This flag bit is specified in {{iana-cons-flag-bits}}.

   When the Extension-1 Flag is set to 1, the second byte of the OSCORE option MUST include the set of OSCORE flag bits 8-15.

* This document defines the usage of the first least significant bit "Nonce Flag", 'd', in the second byte of the OSCORE option containing the OSCORE flag bits. This flag bit is specified in {{iana-cons-flag-bits}}.

   When it is set to 1, the compressed COSE object contains an 'nonce', to be used for the steps defined in {{ssec-derive-ctx}}. The 1 byte 'x' following 'kid context' (if any) encodes the length of 'nonce', and signaling bits for specific behaviour during the KUDOS execution. Specifically the encoding of 'x' is as follows:

   * The four least significant bits encode the length of the 'nonce' in bytes minus 1, as an integer.

   * The fifth least significant bit is the "No Forward Secrecy" 'p' bit, see {{no-fs-signaling}}. The sender peer indicates its wish to run KUDOS in FS mode or in no-FS mode, by setting the 'p' bit to 0 or 1, respectively. This makes KUDOS possible to run also to devices that do not support its FS mode. At the same time, two devices MUST run KUDOS in FS mode if they both support it, as per {{ssec-update-function}}. The execution of KUDOS in no-FS mode is defined in {{no-fs-mode}}.

   * The sixth least significant bit is the "Preserve Observations" 'b' bit, see {{preserving-observe-signaling}}. The sender peer indicates its wish to preserve ongoing observations or not beyond the KUDOS execution, by setting the 'b' bit to 1 or 0, respectively. The related processing is defined in {{preserving-observe}}.

   * The seventh least significant bit is reserved and SHALL be set to zero.

   * The eight least significant bit is reserved for future use and SHALL be set to zero.

   Hereafter, this document refers to a message where the 'd' flag is set to 0 as "non KUDOS (request/response) message", and to a message where the 'd' flag is set to 1 as "KUDOS (request/response) message".

* The second-to-eighth least significant bits in the second byte of the OSCORE option containing the OSCORE flag bits are reserved for future use. These bits SHALL be set to zero when not in use. According to this specification, if any of these bits are set to 1, the message is considered to be malformed and decompression fails as specified in item 2 of {{Section 8.2 of RFC8613}}.

{{fig-oscore-option}} shows the OSCORE option value including also 'nonce'.

~~~~~~~~~~~
 0 1 2 3 4 5 6 7  8   9   10  11  12  13  14  15 <----- n bytes ----->
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+
|0|1|0|h|k|  n  | 0 | 0 | 0 | 0 | 0 | 0 | 0 | d | Partial IV (if any) |
+-+-+-+-+-+-+-+-+---+---+---+---+---+---+---+---+---------------------+

 <- 1 byte -> <----- s bytes ------> <- 1 byte -> <----- x bytes ---->
+------------+----------------------+---------------------------------+
| s (if any) | kid context (if any) | x (if any) | nonce (if any)     |
+------------+----------------------+------------+--------------------+

+------------------+
| kid (if any) ... |
+------------------+
~~~~~~~~~~~
{: #fig-oscore-option title="The OSCORE option value, including 'nonce'" artwork-align="center"}

## Function for Security Context Update # {#ssec-update-function}

The updateCtx() function shown in {{function-update}} takes as input a nonce N, the 'x' parameter of the OSCORE option (signaling bits and length of the 'nonce') as X, as well as an OSCORE Security Context CTX\_IN, and returns as output a new OSCORE Security Context CTX\_OUT.

As a first step, the updateCtx() function derives the new values of the Master Secret and Master Salt for CTX\_OUT, according to one of the two following methods. The used method depends on how the two peers established their original Security Context, i.e., the Security Context that they shared before performing KUDOS with one another for the first time.

* If the original Security Context was established by running the EDHOC protocol {{I-D.ietf-lake-edhoc}}, the following applies.

   First, the EDHOC keys PRK_out and PRK_exporter shared by the two peers is updated using the EDHOC-KeyUpdate() function defined in {{Section 4.4 of I-D.ietf-lake-edhoc}}, which takes the nonce N as input.

   After that, the EDHOC-Exporter() function defined in {{Section 4.3 of I-D.ietf-lake-edhoc}} is used to derive the new values for the Master Secret and Master Salt, consistently with what is defined in {{Section A.2 of I-D.ietf-lake-edhoc}}. In particular, the context parameter provided as second argument to the EDHOC-Exporter() function is the empty CBOR byte string (0x40) {{RFC8949}}, which is denoted as h''.

   Note that, compared to the compliance requirements in {{Section 7 of I-D.ietf-lake-edhoc}}, a peer MUST support the EDHOC-KeyUpdate() function, in case it establishes an original Security Context through the EDHOC protocol and intends to perform KUDOS.

* If the original Security Context was established through other means than the EDHOC protocol, the new Master Secret is derived through an HKDF-Expand() step, which takes as input N, X, as well as the Master Secret value from the Security Context CTX\_IN. Instead, the new Master Salt takes N as value.

The X parameter, the whole byte 'x' following 'kid context' (if any), is used as input in the derivation of the new OSCORE Security Contexts during the procedure. This is because the 'x' byte encodes both the length of 'nonce', but also contains signaling bits which are fundamental to integrity-protect.

In either case, the derivation of new values follows the same approach used in TLS 1.3, which is also based on HKDF-Expand (see {{Section 7.1 of RFC8446}}) and used for computing new keying material in case of key update (see {{Section 4.6.3 of RFC8446}}).

After that, the new Master Secret and Master Salt parameters are used to derive a new Security Context CTX\_OUT as per {{Section 3.2 of RFC8613}}. Any other parameter required for the derivation takes the same value as in the Security Context CTX\_IN. Finally, the function returns the newly derived Security Context CTX\_OUT.

\[ NOTE:

Note that in the case a peer has lost its EDHOC session it may be unable to utilize the EDHOC-KeyUpdate for deriving a new Security Context. In such case yet another bit in the X1 and X2 bytes can be utilized for signaling to the other peer how the new Security Context should be derived, that is relying on the EDHOC-KeyUpdate or using HKDF-Expand. Furthermore, currently the updateCtx function is defined as a single function, both able to generate a new context with EDHOC-KeyUpdate or HKDF-Expand. An alternative design would be to split this functionality into two separate functions. Or, alternatively keep updateCtx() and create two additional functions to be called from within updateCtx(). This would allow calling the appropriate function depending on the desired functionality. Such a design can be more convenient as the X input does not have to be parsed inside updateCtx, rather the parsing can happen beforehand and the correct function called.

\]

~~~~~~~~~~~
updateCtx(X, N, CTX_IN) {

  CTX_OUT       // The new Security Context
  MSECRET_NEW   // The new Master Secret
  MSALT_NEW     // The new Master Salt

  X_cbor = bstr .cbor X // CBOR bstr wrapping of X
  N_cbor = bstr .cbor N // CBOR bstr wrapping of N
  X_N = bstr .cbor (X_cbor | N_cbor) // CBOR bstr wrapping of above

  oscore_key_length = < Size of CTX_IN.MasterSecret in bytes >
  oscore_salt_length = < Size of CTX_IN.MasterSalt in bytes >

  if <the original Security Context was established through EDHOC> {

    EDHOC-KeyUpdate(X_N)
    // This results in updating the key PRK_out of the
    // EDHOC session, i.e., PRK_out = EDHOC-KDF(PRK_out, 11,
    // N, hash_length)
    // It also results in updating the key PRK_exporter, i.e.,
    // PRK_exporter = EDHOC-KDF(PRK_out, 10, h'', hash_length)

    MSECRET_NEW = EDHOC-Exporter(0, h'', oscore_key_length)

      = EDHOC-KDF(PRK_exporter, 0, h'', oscore_key_length)


    MSALT_NEW = EDHOC-Exporter(1, h'', oscore_salt_length)

      = EDHOC-KDF(PRK_exporter, 1, h'', oscore_salt_length)

  }
  else {

    MSECRET_NEW = HKDF-Expand-Label(CTX_IN.MasterSecret, Label,
                                    X_N, oscore_key_length)
                = HKDF-Expand(CTX_IN.MasterSecret, HkdfLabel,
                              oscore_key_length)

    MSALT_NEW = N;
  }

  < Derive CTX_OUT using MSECRET_NEW and MSALT_NEW,
    together with other parameters from CTX_IN >

  Return CTX_OUT;

}

Where HkdfLabel is defined as

struct {
    uint16 length = oscore_key_length;
    opaque label<7..255> = "oscore " + Label;
    opaque context<0..255> = X_N;
} HkdfLabel;
~~~~~~~~~~~
{: #function-update title="Function for deriving a new OSCORE Security Context" artwork-align="center"}

## Establishment of the New OSCORE Security Context # {#ssec-derive-ctx}

This section defines the actual KUDOS procedure performed by two peers to update their OSCORE keying material. Before starting KUDOS, the two peers share the OSCORE Security Context CTX\_OLD. Once completed the KUDOS execution, the two peers agree on a newly established OSCORE Security Context CTX\_NEW.

In particular, each peer contributes by generating a fresh value N1 or N2, and providing it to the other peer. Furthermore, X1 and X2 are the values of the 'x' byte specified in the OSCORE option of the first and second KUDOS message, respectively. These values are used by the peers as input to the updateCtx() function in order to derive a new OSCORE Security Context. As for any new OSCORE Security Context, the Sender Sequence Number and the replay window are re-initialized accordingly (see {{Section 3.2.2 of RFC8613}}).

When starting KUDOS the initiator determines if it wants to preserve observations. In such case, the "Preserve Observations" 'b' bit in the message is set to 1 (see {{ssec-oscore-option-extensions}}). Only if the equivalent bit from the responder is set to 1 observatons are preserved (see {{preserving-observe}}). Note that this may be exploited as a general method for terminating multiple ongoing observations. That is, a client wishing to terminate all its ongoing observations can execute the KUDOS procedure and indicate to NOT keep ongoing observations. Thus, all its observations will be terminated after the KUDOS execution has finished. Note that in this scenario the client may in fact not be interested primarily in running KUDOS to update the key material but rather specifically for terminating observations. This solution may be cheaper than explicitly sending observaton cancellation requests for all ongoing observations.

Once a peer has successfully derived the new OSCORE Security Context CTX\_NEW, that peer MUST use CTX\_NEW to protect outgoing non KUDOS messages.

Also, that peer MUST terminate all the ongoing observations {{RFC7641}} that it has with the other peer as protected with the old Security Context CTX\_OLD, unless the two peers have explicitly agreed otherwise as defined in {{preserving-observe}}.

Once a peer has successfully decrypted and verified an incoming message protected with CTX\_NEW, that peer MUST discard the old Security Context CTX\_OLD.

KUDOS can be started by the client or the server, as defined in {{ssec-derive-ctx-client-init}} and {{ssec-derive-ctx-server-init}}, respectively. The following properties hold for both the client- and server-initiated version of KUDOS.

* The initiator always offers the fresh value N1.
* The responder always offers the fresh value N2
* The initiator and responder encodes the 'x' byte with the length of 'nonce' and signaling bits
* The responder is always the first one deriving the new OSCORE Security Context CTX\_NEW.
* The initiator is always the first one achieving key confirmation, hence able to safely discard the old OSCORE Security Context CTX\_OLD.
* Both the initiator and the responder use the same respective OSCORE Sender ID and Recipient ID. Also, they both preserve and use the same OSCORE ID Context from CTX\_OLD.

The length of the nonces N1 and N2 is application specific. The application needs to set the length of each nonce such that the probability of its value being repeated is negligible. To this end, each nonce is typically at least 8 bytes long.

Once a peer acting as initiator (responder) has sent (received) the first KUDOS message, that peer MUST NOT send a non KUDOS message to the other peer, until having completed the key update process on its side. The initiator completes the key update process when receiving the second KUDOS message and successfully verifying it with the new OSCORE Security Context CTX\_NEW. The responder completes the key update process when sending the second KUDOS message, as protected with the new OSCORE Security Context CTX\_NEW.

In the following sections 'b(c,d)' denotes the byte concatenation of two elements. The first element is the CBOR byte string with value 'c'. The second element is the CBOR byte string with value 'd'. That is, b(c,d) = bstr .cbor c \| bstr .cbor d, where \| denotes byte concatenation.

<!-- Add example with real values, and for updateCtx processing -->

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
Generate N1          |                    |
                     |                    |
CTX_1 =              |                    |
  updateCtx(X1, N1,  |                    |
            CTX_OLD) |                    |
                     |                    |
                     |     Request #1     |
Protect with CTX_1   |------------------->|
                     | OSCORE Option:     | CTX_1 =
                     |   ...              |  updateCtx(X1, N1,
                     |   d flag: 1        |            CTX_OLD)
                     |   X1               |
                     |   Nonce: N1        | Verify with CTX_1
                     |   ...              |
                     |                    | Generate N2
                     |                    |
                     |                    | CTX_NEW =
                     |                    |  updateCtx(b(X1,X2), b(N1,N2),
                     |                    |            CTX_OLD)
                     |                    |
                     |     Response #1    |
                     |<-------------------| Protect with CTX_NEW
CTX_NEW =            | OSCORE Option:     |
 updateCtx(b(X1,X2), |   ...              |
           b(N1,N2), |                    |
           CTX_OLD)  |   d flag: 1        |
                     |   X2               |
Verify with CTX_NEW  |   Nonce: N2        |
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

First, the client generates a random value N1, and uses the nonce N = N1 and X = X1 together with the old Security Context CTX\_OLD, in order to derive a temporary Security Context CTX\_1. Then, the client sends an OSCORE request to the server, protected with the Security Context CTX\_1. In particular, the request has the 'd' flag bit set to 1, specifies N1 as 'nonce' (see {{ssec-oscore-option-extensions}}).

An example of this nonce processing on the client with values for N1 and X1 is presented in {{fig-kudos-nonce-mess-one}}.

~~~~~~~~~~~~~~~~~~~~~~~
   N1 and X1 expressed a raw values
   N1 = 0x018a278f7faab55a
   X1 = 0x80

   updateCtx() is called with
   N = 0x018a278f7faab55a
   X = 0x80

   In updateCtx() N_cbor and X_cbor are built as CBOR encoded byte strings
   N_cbor = 0x48018a278f7faab55a (h'018a278f7faab55a')
   X_cbor = 0x4180               (h'80')

   Finally X_N is built from N_cbor and X_cbor
   X_N = 0x4b418048018A278F7FAAB55A (h'4180 48018a278f7faab55a')
~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-kudos-nonce-mess-one title="Example of nonce processing for KUDOS message 1"}

Upon receiving the OSCORE request, the server retrieves the value N1 from the 'nonce' of the request, the value X1 from the 'x' byte of the OSCORE option, and uses the nonce N = N1 and X = X1 together with the old Security Context CTX\_OLD, in order to derive the temporary Security Context CTX\_1. Then, the server verifies the request by using the Security Context CTX\_1.

After that, the server generates a random value N2, and uses the nonce N = b(N1, N2) and X = b(X1, X2) together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Then, the server sends an OSCORE response to the client, protected with the new Security Context CTX\_NEW. In particular, the response has the 'd' flag bit set to 1 and specifies N2 as 'nonce'.


An example of this nonce processing on the server with values for N1, X1, N2 and X2 is presented in {{fig-kudos-nonce-mess-two}}.

~~~~~~~~~~~~~~~~~~~~~~~
   N1, X1, N2 and X2 expressed a raw values
   N1 = 0x018a278f7faab55a
   X1 = 0x80
   N2 = 0x25a8991cd700ac01
   X2 = 0x80

   N1, X1, N2, and X2 as CBOR encoded byte strings
   N1 = 0x48018A278F7FAAB55A
   X1 = 0x4180
   N2 = 0x4825a8991cd700ac01
   X2 = 0x4180

   N and X are built
   N = 0x48018A278F7FAAB55A4825a8991cd700ac01
   X = 0x41804180

   In updateCtx() N_cbor and X_cbor are built as CBOR encoded byte strings
   N_cbor = 0x5248018A278F7FAAB55A4825A8991CD700AC01
   X_cbor = 0x4441804180

   Finally X_N is built from N_cbor and X_cbor
   X_N = 0x581844418041805248018A278F7FAAB55A4825A8991CD700AC01
   (h'4441804180 5248018A278F7FAAB55A4825A8991CD700AC01')
~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-kudos-nonce-mess-two title="Example of nonce processing for KUDOS message 2"}

Upon receiving the OSCORE response, the client retrieves the value N2 from the 'nonce' of the response, and the value X2 from the 'x' byte of the OSCORE option. Since the client has received a response to an OSCORE request it made with the 'd' flag bit set to 1, the client uses the nonce N = b(N1, N2) and X = b(X1, X2) together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Finally, the client verifies the response by using the Security Context CTX\_NEW and deletes the old Security Context CTX\_OLD.

After that, the client can send a new OSCORE request protected with the new Security Context CTX\_NEW. When successfully verifying the request using the Security Context CTX\_NEW, the server deletes the old Security Context CTX\_OLD and can reply with an OSCORE response protected with the new Security Context CTX\_NEW.

From then on, the two peers can protect their message exchanges by using the new Security Context CTX\_NEW.

Note that the server achieves key confirmation only when receiving a message from the client as protected with the new Security Context CTX\_NEW. If the server sends a non KUDOS request to the client protected with CTX\_NEW before then, and the server receives a 4.01 (Unauthorized) error response as reply, the server SHOULD delete the new Security Context CTX\_NEW and start a new client-initiated key update process, by taking the role of initiator as per {{fig-message-exchange-client-init}}.

Also note that, if both peers reboot simultaneously, they will run the client-initiated version of KUDOS defined in this section. That is, one of the two peers implementing a CoAP client will send KUDOS Request #1 in {{fig-message-exchange-client-init}}.

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
                     |                    | Generate N1
                     |                    |
                     |                    | CTX_1 =
                     |                    |  updateCtx(X1, N1,
                     |                    |            CTX_OLD)
                     |                    |
                     |     Response #1    |
                     |<-------------------| Protect with CTX_1
CTX_1 =              | OSCORE Option:     |
  updateCtx(X1, N1,  |   ...              |
            CTX_OLD) |   d flag: 1        |
                     |   X1               |
Verify with CTX_1    |   Nonce: N1        |
                     |   ...              |
Generate N2          |                    |
                     |                    |
CTX_NEW =            |                    |
 updateCtx(b(X1,X2), |                    |
           b(N1,N2   |                    |
           CTX_OLD)  |                    |
                     |                    |
                     |     Request #2     |
Protect with CTX_NEW |------------------->|
                     | OSCORE Option:     | CTX_NEW =
                     |   ...              |  updateCtx(b(X1,X2), b(N1,N2),
                     |   d flag: 1        |            CTX_OLD)
                     |   X2               |
                     |   Nonce: N1|N2     | Verify with CTX_NEW
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

Upon receiving the OSCORE request and after having verified it with the old Security Context CTX\_OLD as usual, the server generates a random value N1 and uses the nonce N = N1 and X = X1 together with the old Security Context CTX\_OLD, in order to derive a temporary Security Context CTX\_1. Then, the server sends an OSCORE response to the client, protected with the Security Context CTX\_1. In particular, the response has the 'd' flag bit set to 1 and specifies N1 as 'nonce' (see {{ssec-oscore-option-extensions}}).

Upon receiving the OSCORE response, the client retrieves the value N1 from the 'nonce' of the response, the value X1 from the 'x' byte of the OSCORE option, and uses the nonce N = N1 and X = X1 together with the old Security Context CTX\_OLD, in order to derive the temporary Security Context CTX\_1. Then, the client verifies the response by using the Security Context CTX\_1.

After that, the client generates a random value N2, and uses the nonce N = b(N1, N2) and X = b(X1, X2) together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Then, the client sends an OSCORE request to the server, protected with the new Security Context CTX\_NEW. In particular, the request has the 'd' flag bit set to 1 and specifies N1 \| N2 as 'nonce'.

Upon receiving the OSCORE request, the server retrieves the value N1 \| N2 from the request and the value X2 from the 'x' byte of the OSCORE option. Then, the server verifies that: i) the value N1 is identical to the value N1 specified in a previous OSCORE response with the 'd' flag bit set to 1; and ii) the value N1 \| N2 has not been received before in an OSCORE request with the 'd' flag bit set to 1. If the verification succeeds, the server uses the nonce N = b(N1, N2) and X = b(X1, X2) together with the old Security Context CTX\_OLD, in order to derive the new Security Context CTX\_NEW. Finally, the server verifies the request by using the Security Context CTX\_NEW and deletes the old Security Context CTX\_OLD.

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

## Key Update without Forward Secrecy {#no-fs-mode}

The main version of the KUDOS procedure defined in {{sec-rekeying-method}} ensures forward secrecy of the OSCORE keying material. However, it requires peers executing KUDOS to preserve their state (e.g., across a device reboot), by writing information such as data from the newly derived OSCORE Security Context CTX\_NEW in non-volatile memory.

However, this can be problematic for devices that cannot dynamically write information to non-volatile memory. For example, some devices may support only a single writing in persistent memory when initial keying material is provided (e.g., at manufacturing or commissioning time), but not more after that. Therefore, these devices cannot perform a stateful key update procedure, which prevents running the main version of KUDOS and ensuring forward secrecy.

In order to address these limitations, this section defines an alternative, stateless version of the KUDOS procedure. This allows two peers to achieve the same results as when running the main version of KUDOS defined in {{sec-rekeying-method}}, with the difference that no forward secrecy is achieved and no state information is required to be dynamically written in non-volatile memory.

Hereafter, "FS mode" and "non-FS mode" refer to the main version of KUDOS defined in {{sec-rekeying-method}} and the alternative version of KUDOS defined in this section, respectively. From a practical point of view, the two modes differ as to what exact OSCORE Master Secret and Master Salt are used as part of the OSCORE Security Context CTX\_OLD provided as input to the updateCtx() function (see {{sec-rekeying-method}}).

In order to run KUDOS in FS mode, both peers have to be able to write in non-volatile memory the OSCORE Master Secret and OSCORE Master Salt from the newly derived Security Context CTX\_NEW. If this is not the case, the two peers have to run KUDOS in non-FS mode.

### Handling and use of Keying Material

In the following, a device is denoted as "CAPABLE" if it is able to store information in non-volatible memory (e.g., on disk), beyond a one-time-only writing occurring at manufacturing or (re-)commissioning time.

The following terms are used to refer to OSCORE keying material.

* Bootstrap Master Secret and Bootstrap Master Salt. If pre-provisioned during manufacturing or (re-)commissioning, these OSCORE Master Secret and Master Salt are initially stored on disk and are never going to be overwritten by the device.

* Latest Master Secret and Latest Master Salt. These OSCORE Master Secret and Master Salt can be dynamically updated by the device. In case of reboot, they are lost unless they have been stored on disk.

Note that:

* A peer running KUDOS can have none of the pairs above associated with another peer, only one or both.

* A peer that has neither of the pairs above associated with another peer, cannot run KUDOS in any mode with that other peer.

* A peer that has only one of the pairs above associated with another peer can attempt to run KUDOS with that other peer, but the procedure might fail depending on the other peer's capabilities. In particular:

   - In order to run KUDOS in FS mode, a peer must be a CAPABLE device. It follows that two peers have to both be CAPABLE devices in order to be able to run KUDOS in FS mode with one another.

   - In order to run KUDOS in no-FS mode, a peer must have Bootstrap Master Secret and Bootstrap Master Salt available as stored on disk.

As a general rule, once successfully generated a new OSCORE Security Context CTX (e.g., CTX is the CTX\_NEW resulting from a KUDOS execution, or it has been established through EDHOC {{I-D.ietf-lake-edhoc}}), a peer considers the Master Secret and Master Salt of CTX as Latest Master Secret and Latest Master Salt. After that:

* If the peer is a CAPABLE device, it SHOULD store Latest Master Secret and Latest Master Salt on disk.

   As an exception, this does not apply to possible temporary OSCORE Security Contexts used during a key update procedure, such as CTX\_1 used during the KUDOS execution. That is, the OSCORE Master Secret and Master Salt from such temporary Security Contexts MUST NOT be stored on disk.

* The peer MUST store Latest Master Secret and Latest Master Salt in volatile memory, thus making them available to OSCORE message processing and possible key update procedures.

#### Actions after Device Reboot

Building on the above, after having experienced a reboot, a peer A checks whether it has a pair P1 = (Latest Master Secret, Latest Master Salt) associated with any another peer B stored on disk.

* If a pair P1 is found, the peer A performs the following actions.

    * The peer A loads the Latest Master Secret and Latest Master Salt to volatile memory, and uses them to derive an OSCORE Security Context CTX\_OLD.

    * The peer A runs KUDOS with the other peer B, acting as initiator. If the peer A is a CAPABLE device, it stores on disk the Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

* If a pair P1 is not found, the peer A checks whether it has a pair P2 = (Bootstrap Master Secret, Bootstrap Master Salt) associated with the other peer B stored on disk.

    * If a pair P2 is found, the peer A performs the following actions.

        * The peer A loads the Bootstrap Master Secret and Bootstrap Master Salt to volatile memory, and uses them to derive an OSCORE Security Context CTX\_OLD.

        * If the peer A is a CAPABLE device, it stores on disk Bootstrap Master Secret and Bootstrap Master Salt as Latest Master Secret and Latest Master Salt, respectively. This supports the situation where A is a CAPABLE device and has never run KUDOS with the other peer B before.

        * The peer A runs KUDOS with the other peer B, acting as initiator. If the peer A is a CAPABLE device, it stores on disk the Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

    * If a pair P2 is not found, the peer A has to use alternative ways to establish a first OSCORE Security Context CTX\_NEW with the other peer B, e.g., by running EDHOC. After that, if A is a CAPABLE device, it stores on disk the Master Secret and Master Salt from the newly established OSCORE Security Context CTX\_NEW, as Latest Master Secret and Latest Master Salt, respectively.

### Signaling of FS Mode or Non-FS Mode # {#no-fs-signaling}

The signaling method that two peers use to agree on whether to run KUDOS in FS or non-FS mode is a signaling bit in the 'x' byte of the OSCORE option. Specifically, the bit "No Forward Secrecy", 'p', is set to 1 by the sender peer to indicate that it wishes to run KUDOS in no-FS mode, or to 0 if it wishes to run KUDOS in FS mode (see {{ssec-oscore-option-extensions}}).

If the second byte of the OSCORE option containing flag bits is present and the 'd' flag is set to 0 (i.e., the message is not a KUDOS message), the bit 'p' MUST be set to 0.

In a KUDOS message (i.e., the 'd' bit is set to 1), the 'p' bit practically determines what OSCORE Security Context to use as CTX\_OLD during the KUDOS execution, consistently with the indicated mode.

* If the 'p' bit is set to 0, the updateCtx() function used to derive CTX\_1 or CTX\_NEW considers as input CTX\_OLD the current OSCORE Security Context shared with the other peer as is. In particular, CTX\_OLD includes Latest Master Secret as Master Secret and Latest Master Salt as Master Salt.

* If the 'p' bit is set to 1, the updateCtx() function used to derive CTX\_1 or CTX\_NEW considers as input CTX\_OLD the current OSCORE Security Context shared with the other peer, with the following difference: Bootstrap Master Secret is used as Master Secret and Boostrap Master Salt is used as Master Salt. That is, every execution of KUDOS in no-FS mode between these two peers considers the same pair (Master Secret, Master Salt) in the OSCORE Security Context CTX\_OLD provided as input to the updateCtx() function, hence the impossibility to achieve forward secrecy.

### Selection of KUDOS Mode

This section defines how a peer determines to run KUDOS either in FS or no-FS mode with another peer.

* If a peer A is not a CAPABLE device, it MUST run KUDOS only in no-FS mode. That is, when sending a KUDOS message, it MUST set the 'p' bit to 1 (see {{no-fs-signaling}}).

* If a peer A is a CAPABLE device, it SHOULD run KUDOS only in FS mode and SHOULD NOT run KUDOS as initiator in no-FS mode. That is, when sending a KUDOS message, it SHOULD set the 'p' bit to 0  (see {{no-fs-signaling}}). An exception applies in the following cases.

   * The peer A is running KUDOS with another peer B, which A has learned to not be a CAPABLE device (and hence not able to run KUDOS in FS mode).

      Note that, if the peer A is a CAPABLE device, it is able to store such information about the other peer B on disk and it MUST do so. From then on, the peer A will perform every execution of KUDOS with the peer B in no-FS mode, including after a possible reboot.

   * The peer A is acting as responder and running KUDOS with another peer B without knowing its capabilities, and A receives a KUDOS message with the 'p' bit set to 1.

* If the peer A is a CAPABLE device and has learned that another peer B is also a CAPABLE device (and hence able to run KUDOS in FS mode), then the peer A MUST NOT run KUDOS with the peer B in non-FS mode. This also means that, if the peer A acts as responder when running KUDOS with the peer B, the peer A MUST terminate the KUDOS execution if it receives a KUDOS message from the peer B with the 'p' bit set to 1.

   Note that, if the peer A is a CAPABLE device, it is able to store such information about the other peer B on disk and it MUST do so. This ensures that the peer A will perform every execution of KUDOS with the peer B in FS mode. In turn, this prevents a possible downgrading attack, aimed at making A believe that B is not a CAPABLE device, and thus to run KUDOS in no-FS mode although the FS mode is actually supported by both peers.

Within the limitations above, two peers running KUDOS generate the new OSCORE Security Context CTX\_NEW according to the mode indicated per the bit 'p' set by the responder in the second KUDOS message.

If, after having received the first KUDOS message, the responder can continue performing KUDOS, the bit 'p' in the reply message has the same value as in the bit 'p' set by the initiator, unless the value is 0 and the responder is not a CAPABLE device. More specifically:

* If both peers are CAPABLE devices, they will run KUDOS in FS mode. That is, both initiator and responder sets the 'p' bit to 0 in the respective sent KUDOS message.

* If both peers are not CAPABLE devices or only the peer acting as initiator is not a CAPABLE device, they will run KUDOS in no-FS mode. That is, both initiator and responder sets the 'p' bit to 1 in the respective sent KUDOS message.

* If only the peer acting as initiator is a CAPABLE device and it has knowledge of the other peer being a not CAPABLE device, they will run KUDOS in no-FS mode. That is, both initiator and responder sets the 'p' bit to 1 in the respective sent KUDOS message.

* If only the peer acting as initiator is a CAPABLE device and it has no knowledge of the other peer being a not CAPABLE device, they will not run KUDOS in FS mode and will rather set to ground for possibly retrying in no-FS mode. In particular, the initiator sets the 'p' bit of its sent KUDOS message to 0. Then:

   * If the responder is a server, it MUST reply with a 5.03 (Service Unavailable) error response. The response is protected with the newly derived OSCORE Security Context CTX\_NEW. The diagnostic payload MAY provide additional information. The 'p' bit in the error response MUST be set to 1.

      When receiving the error response, the initiator learns that the responder is not a CAPABLE device (and hence not able to run KUDOS in FS mode). The initiator MAY try running KUDOS again, by setting the 'p' bit to 1 when sending a new request as first KUDOS message.

   * If the responder is a client, it sends to the initiator the second KUDOS message protected with the newly derived OSCORE Security Context CTX_NEW. The 'p' bit in the request MUST be set to 1.

      When receiving the request above (i.e., with the 'p' bit set to 1 as a follow-up to the previous KUDOS response having the 'p' bit set to 0), the initiator learns that the responder is not a CAPABLE device (and hence not able to run KUDOS in FS mode).

In either case, both KUDOS peers delete the OSCORE Security Contexts CTX\_1 and CTX\_NEW.

## Preserving Observations across Key Updates # {#preserving-observe}

As defined in {{ssec-derive-ctx}}, once a peer has completed the KUDOS execution and successfully derived the new OSCORE Security Context CTX\_NEW, that peer normally terminates all the ongoing observations it has with the other peer {{RFC7641}}, as protected with the old Security Context CTX\_OLD.

This section describes a method that the two peers can use to safely preserve the ongoing observations that they have with one another, after having completed a KUDOS execution. In particular, this method ensures that an Observe notification can never successfully cryptographically match against the Observe requests of two different observations, i.e., an Observe request protected with CTX\_OLD and an Observe request protected with CTX\_NEW.

The actual preservation of ongoing observations has to be agreed by the two peers at each execution of KUDOS that they run with one another, as defined in {{preserving-observe-signaling}}. If, at the end of a KUDOS execution, the two peers have not agreed on that, they MUST terminate the ongoing observations that they have with one another, as defined in {{ssec-derive-ctx}}.

The following sections specify the different actions taken by the peer depending on whether it acts as client or server in an ongoing observation, as well as the signaling method used in KUDOS to agree on preserving the ongoing observations beyond the current KUDOS execution.

\[ NOTE:

This method may be of more general applicability, i.e, also in case an update of the OSCORE keying material is performed through a different means than KUDOS.

\]

### Management of Observations

As per {{Section 3.1 of RFC7641}}, a client can register its interest in observing a resource at a server, by sending a registration request including the Observe option with value 0.

If the server sends back a successful response also including the Observe option, hence confirming that the observation has been registered, then the server registers it as an ongoing observation.

If the client receives back the successful response from the server, then the client likewise registers it as an ongoing observation.

If, later on, the client is not interested in the observation anymore, it MUST NOT simply forget about it. Rather, the client MUST send an explicit cancellation request to the server, i.e., a request including the Observe option with value 1 (see {{Section 3.6 of RFC7641}}). After sending this cancellation request, if the client does not receive back a response confirming that the observation has been terminated, the client MUST NOT consider the observation terminated. The client MAY try again to terminate the observation by sending a new cancellation request.

In case a peer A performs a KUDOS execution with another peer B, and A has ongoing observations with B that it is interested to preserve across the key update, then A explicitly indicates it by using the signaling approach embedded in KUDOS and defined in {{preserving-observe-signaling}}.

After having successfully completed the KUDOS execution (i.e., after having successfully derived the new OSCORE Security Context CTX\_NEW), if the other peer B has confirmed its interest in preserving those ongoing observations also by using the signaling approach defined in {{preserving-observe-signaling}}, then the peer A performs the following actions. This sequence of steps will allow the peer to "jump" over Partial IVs (PIVs) that are occupied and in use for ongoing observations.

1. For each still ongoing observation X that A has with B, such that A acts as client in X:

   a. The peer A considers all the OSCORE Partial IV values used in the Observe registration request associated with any of the still ongoing observations with the other peer B.

   b. The peer A determines the value PIV\* as the highest OSCORE Partial IV among those considered at the previous step.

   c. In the Sender Context of the OSCORE Security Context shared with the other peer B, the peer A sets its own Sender Sequence Number to (PIV\* + 1), rather than to 0.

Note that when running KUDOS the peer must determine if it wishes to preserve ongoing observations or not. Input to this decision can be an understanding on the peer that its value for PIV\* begins to come close to the maximum possible PIV it can use. In such case it may choose to re-run KUDOS without preserving observations in order to "start over" from a fresh fully unused PIV space. In addition application specific policies can have influence on the decision to preserve observations or not.

### Signaling to Preserve Observations # {#preserving-observe-signaling}

When performing KUDOS, a peer can indicate to the other peer its interest in  preserving the ongoing observations that they have with one another and are bound to the OSCORE Security Context to renew. This is signaled by using the extended OSCORE option shown in {{fig-oscore-option}} and included in a KUDOS message, specifically by setting to 1 the bit "Preserve Observations", 'b', in the 'x' byte contained in the OSCORE option (see {{ssec-oscore-option-extensions}}).

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

# Update of OSCORE Sender/Recipient IDs # {#update-oscore-ids}

This section defines an optional procedure that two peers can execute to update the OSCORE Sender/Recipient IDs that they use in their shared OSCORE Security Context.

This procedure can be initiated by either peer. In particular, the client or the server may start it by sending the first OSCORE ID update message. When sending an OSCORE ID update message, a peer provides its new intended OSCORE Recipient ID to the other peer.

Furthermore, this procedure can be executed stand-alone, or rather seamlessly integrated in an execution of KUDOS (see {{sec-rekeying-method}}).

* In the former stand-alone case, updating the OSCORE Sender/Recipient IDs effectively results in updating part of the current OSCORE Security Context.

   That is, a new Sender Key, Recipient Key and Common IV are derived as defined in {{Section 3.2 of RFC8613}}. Also, the Sender Sequence Number and the replay window are re-initialized accordingly, as defined in {{Section 3.2.2 of RFC8613}}. Since the same Master Secret is preserved, forward secrecy is not achieved.

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
|    15    |  Nonce Flag      | Set to 1 if the        | [this     |
|          |                  | compressed COSE object | document] |
|          |                  | contains 'nonce'       |           |
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

# Document Updates # {#sec-document-updates}

RFC EDITOR: PLEASE REMOVE THIS SECTION.

## Version -01 to -02 ## {#sec-01-02}

* Moved procedure for preserving observations across key updates to main body.

* Moved procedure to update OSCORE Sender/Recipient IDs to main body.

* Moved key update without forward secrecy section to main body.

* Modifications and alignment of updateCtx() with EDHOC draft

* Define signaling bits present in the 'x' byte

* Describe CBOR wrapping of involved nonces with examples

* Rename 'id detail' to 'nonce'

* Additions to terminology section

## Version -00 to -01 ## {#sec-00-01}

* Recommendation on limits for CCM_8. Details in Appendix.

* Improved message processing, also covering corner cases.

* Example of method to estimate and not store 'count_q'.

* Added procedure to update OSCORE Sender/Recipient IDs.

* Added method for preserving observations across key updates.

* Added key update without forward secrecy.

# Acknowledgments # {#acknowledgments}
{: numbered="no"}

The authors sincerely thank Christian Amsüss, John Mattsson and Göran Selander for their feedback and comments.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
