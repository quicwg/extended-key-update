---
title: "Extended Key Update for QUIC"
category: std

docname: draft-ietf-quic-extended-key-update-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
category: std
updates: 9001
consensus: true
v: 3
area: Transport
workgroup: QUIC
keyword:
 - quic
 - extended key update
 - forward secrecy
venue:
  group: QUIC
  type: Working Group
  mail: quic@ietf.org
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "quicwg/extended-key-update"
  latest: "https://quicwg.org/extended-key-update/draft-ietf-quic-extended-key-update.html"

author:
 -
    ins: "Y. Rosomakho"
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com
 -
    ins: H. Tschofenig
    name: Hannes Tschofenig
    abbrev: H-BRS
    organization: University of Applied Sciences Bonn-Rhein-Sieg
    country: Germany
    email: Hannes.Tschofenig@gmx.net
 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    city: Bangalore
    region: Karnataka
    country: India
    email: "kondtir@gmail.com"

normative:

informative:


--- abstract

This document specifies an Extended Key Update mechanism for the QUIC protocol, building on the
foundation of the TLS Extended Key Update. The TLS Extended Key Update specification enhances the
TLS protocol by introducing key updates with forward secrecy, eliminating the need to perform a
full handshake. This feature is particularly beneficial for maintaining security in scenarios
involving long-lived connections.

This specification replaces the QUIC Key Update mechanism described in the "Using TLS to
Secure QUIC" specification.

--- middle

# Introduction

The QUIC protocol {{!QUIC=RFC9000}} provides a secure, versatile transport for various applications, suitable for long-lived sessions
in environments like industrial IoT, telecommunication networks or Virtual Private Networks (VPN), as specified in {{?RFC9484}}.

The TLS Extended Key Update {{!I-D.ietf-tls-extended-key-update}} introduces a mechanism to enhance the security and flexibility
of encrypted communication protocols by enabling frequent key updates without requiring a full handshake renegotiation. This
approach allows applications to refresh their encryption keys more often using ephemeral keys, improving forward secrecy and reducing the risk of key compromise over long-lived connections. By separating key updates from the computationally expensive handshake process,
the specification provides a lightweight method for maintaining robust encryption in scenarios where connections need to
remain secure for extended periods.

The TLS Extended Key Update mechanism is particularly valuable in environments where interruptions to perform a full key exchange would cause significant disruption. Other encrypted communication protocols, such as IPsec {{?IKEv2=RFC7296}} and SSH {{?SSH-TRANSPORT=RFC4253}}, include mechanisms for rekeying without interrupting active sessions. The TLS Extended Key Update specification helps protect sensitive data even in the event of a potential key compromise by enabling frequent key rotation and leveraging forward secrecy.

This specification builds on concepts from {{I-D.ietf-tls-extended-key-update}} and applies them to the QUIC protocol context.
It thereby replaces the QUIC Key Update mechanism described in {{Section 6 of !QUIC-TLS=RFC9001}}. Unlike the previous QUIC key update process, which independently updated keys based on the Key Phase bit, the extended key update mechanism derives a new shared secret using the TLS Extended Key Update procedure. This approach enables a coordinated key transition, integrating TLS for key exchange while refining the QUIC key update process to maintain QUIC-specific key derivation.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Readers are assumed to be familiar with {{!I-D.ietf-tls-extended-key-update}}.

# Extended Key Update Negotiation

QUIC peers negotiate Extended Key Update through the TLS handshake process, as outlined in {{Section 4 of I-D.ietf-tls-extended-key-update}}.
Extended Key Update MUST NOT be used unless both QUIC peers include the TLS flags extension {{!I-D.ietf-tls-tlsflags}} in the handshake and
set the "Extended_Key_Update" flag.

Once the Extended Key Update has been successfully negotiated, QUIC peers MUST use only the Extended Key Update process defined in this document. The standard QUIC Key Update mechanism from {{Section 6 of QUIC-TLS}} MUST NOT be used for the duration of the session, as both
Key Update and Extended Key Update use the Key Phase bit to signal the use of updated keys. The Key Phase bit is initially set to 0 and
toggled to indicate a key update following the successful post-handshake exchange of Extended Key Update messages.

# Extended Key Update Messages

Either party MAY initiate the Extended Key Update process by sending an ExtendedKeyUpdateRequest TLS handshake message in a QUIC CRYPTO frame. This message MUST NOT be sent before the QUIC handshake is confirmed, as described in {{Section 4.1.2 of QUIC-TLS}}. If a QUIC endpoint receives an ExtendedKeyUpdateRequest message before the handshake is complete, it MUST terminate the connection with an error of type 0x010a, equivalent to the TLS unexpected_message alert, as specified in {{Section 4.8 of QUIC-TLS}}.

If both QUIC peers independently initiate an Extended Key Update and their ExtendedKeyUpdateRequest messages cross in flight, the conflict MUST be resolved following the clash error handling defined in {{I-D.ietf-tls-extended-key-update}}. Specifically, the lexicographic order of the key_exchange value in the KeyShareEntry determines which request is rejected, ensuring a coordinated key update process without advancing by two key generations.

Upon receiving an ExtendedKeyUpdateRequest, the recipient MUST respond with an ExtendedKeyUpdateResponse TLS handshake message within a QUIC CRYPTO frame. If a QUIC endpoint receives an ExtendedKeyUpdateResponse without having previously sent an ExtendedKeyUpdateRequest, it MUST treat this as a TLS protocol error and terminate the connection with an error of type 0x010a, equivalent to the TLS unexpected_message alert, as specified in {{Section 4.8 of QUIC-TLS}}.

The ExtendedKeyUpdateRequest and ExtendedKeyUpdateResponse messages are defined in {{Section 5 of I-D.ietf-tls-extended-key-update}}.
Any mismatch between the negotiated NamedGroup during the initial handshake and the group used in the Extended Key Update message, or an incorrect length of the encapsulated key MUST result in connection termination with error of type 0x012f, equivalent to TLS illegal_parameter alert.

If the Extended Key Update initiator receives a retry status in the ExtendedKeyUpdateResponse message, it MUST wait for the duration specified in the response before attempting another key update. The ExtendedKeyUpdateResponse message contains a delay value (in seconds) indicating how long the initiator MUST wait before retrying. The initiator MUST NOT retry within this interval and SHOULD retry once it has lapsed. If the initiator cannot proceed without an immediate Extended Key Update, it MUST terminate the connection with an error of type TBD1, equivalent to the TLS extended_key_update_required alert.

If the initiator receives a rejected status, it MAY terminate the connection with an error of type TBD1, equivalent to the TLS extended_key_update_required alert.

# Updating the Traffic Secrets

After sending an ExtendedKeyUpdateResponse with accepted status, the responder derives new packet protection traffic secrets. The responder MUST continue
using the previous secrets until it has received a packet with the Key Phase bit flipped and has successfully decrypted it using the new keys.

After receiving and succesfully processing an ExtendedKeyUpdateResponse with accepted status, the initiator derives new packet protection traffic secrets,
flips the Key Phase bit for new packets, and uses the new write secret to protect them. The initiator MUST retain the old read secret until
it has received a packet with a flipped Key Phase bit from the responder and succesfully decrypted it using the new read secret.

Both endpoints SHOULD retain old read secrets for some time after unprotecting a packet encrypted with the new keys. Discarding old secret too early may
cause delayed packets to be discarded, which the peer may interpreted as packet loss, potentially impacting performance.

Both endpoints SHOULD retain old read secrets for some time after successfully decrypting a packet encrypted with the new keys. Discarding old secrets too early may cause delayed packets to be discarded, which the peer may interpret as packet loss, potentially impacting performance. However, implementations may choose to discard old secrets sooner in environments where memory limitations or security policies require minimizing the lifetime of old keys. The retention period should be chosen carefully to mitigate the risk of cryptographic attacks while still allowing late-arriving packets to be processed.

{{fig-extended-key-update}} shows this interaction graphically where the initial set of keys used (identified with @M) are replaced by updated keys (identified with @N). The value of the Key Phase bit is indicated in brackets [].

~~~aasvg
        Initiator                              Responder

@M [0] QUIC Packets
                             -------->
                                        @M [0] QUIC Packets
                             <--------

[ ExtendedKeyUpdateRequest ] -------->
                             <--------  [ ExtendedKeyUpdateResponse ]
... Update to @N
@N [1] QUIC Packets
                             -------->
                                        Update to @N ...
                                        QUIC Packets [1] @N
                             <--------
                                        QUIC Packets [1] @N
                                        containing ACK
                             <--------
... Key Update Permitted

@N [1] QUIC Packets
containing ACK for @N packets
                             -------->
                                        Key Update Permitted ...

~~~
{: #fig-extended-key-update title="Extended Key Update Process in QUIC."}

QUIC endpoints MUST NOT send NewKeyUpdate TLS handshake messages, defined in {{I-D.ietf-tls-extended-key-update}}, and
instead rely on the use of the Key Phase bit. Endpoints MUST treat the receipt of a TLS NewKeyUpdate message as a connection error
of type 0x010a. QUIC endpoints that have agreed to the Extended Key Update process MUST NOT change the Key Phase bit without a succesful exchange of
Extended Key Update TLS messages. Receiving a packet with the Key Phase bit changed without a success Extended Key Update exchange MUST be treated as
a connection error of type KEY_UPDATE_ERROR (0x0e).

The design of the key derivation function for computing the next generation of secrets corresponds to the one described in
{{Section 6 of I-D.ietf-tls-extended-key-update}} with the exception of the use of a different label.

~~~pseudocode
sk = HKDF-Extract(Transcript-Hash(ExtendedKeyUpdateRequest,
                                  ExtendedKeyUpdateResponse), secret)

secret_<n+1> = HKDF-Expand-Label(sk, "quic eku",
                                 secret_<n>, Hash.length)
~~~


The corresponding key and IV are derived from the new secret as defined in {{Section 5.1 of QUIC-TLS}}. The header protection key is not updated.

# Security Considerations

This specification describes an update to the key schedule of QUIC. Therefore, implementations MUST ensure that peers adhere strictly to the process
described in this document. Packets with higher packet numbers MUST NOT be protected using an older generation of secrets, as this could compromise key
synchronization and forward security.

As key exchange may be computationally intensive, responders SHOULD consider rate-limiting Extended Key Exchange requests. This can be done by responding
with retry status as outlined in {{Section 5 of I-D.ietf-tls-extended-key-update}} and terminating connections for initiators that violate the back-off timer.
This approach helps prevent excessive load on endpoints and mitigates the risk of denial-of-service attacks.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

We would like to thank Martin Thomson and Sean Turner for their early review of this design and their invaluable feedback.
