---
title: "Extended Key Update for QUIC"
category: std

docname: draft-rosomakho-quic-extended-key-update-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: Transport
# workgroup: QUIC
keyword:
 - quic
 - extended key update
 - forward secrecy
venue:
#  group: QUIC
#  type: Working Group
#  mail: quic@ietf.org
#  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "yaroslavros/quic-extended-key-update"
  latest: "https://yaroslavros.github.io/quic-extended-key-update/draft-rosomakho-quic-extended-key-update.html"

author:
 -
    ins: "Y. Rosomakho"
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

normative:

informative:


--- abstract

This document specified an Extended Key Update mechanism for the {{!QUIC=RFC9000}} protocol, building on the
foundation of the Transport Layer Security (TLS) 1.3 Extended Key Update {{!I-D.ietf-tls-extended-key-update}}.
This specification replaces QUIC Key Update mechanism described in {{Section 6 of !QUIC-TLS=RFC9001}}.

--- middle

# Introduction

The {{QUIC}} protocol provides a secure, versatile transport for various applications, suitable for long-lived sessions
in environments like industrial IoT, telecommunication networks or Virtual Private Networks (VPN), as specified in {{?RFC9484}}.
In such settings, session interruptions to perform a fresh key exchange and update the master secret can be highly disruptive.
Other encrypted communication protocols, such as IPSec {{?IKEv2=RFC7296}} and SSH {{?SSH-TRANSPORT=RFC4253}},
include mechanisms for re-exchanging keys without interrupting active sessions.

The Extended Key Update feature for {{QUIC}} expands the protocol applicability to environments requiring periodic key
re-exchanges, facilitates migration from other secure transport mechanisms, and enhances overall security.
This specification builds on concepts from {{I-D.ietf-tls-extended-key-update}}, while adapting to the unique challenges
of the potentially unreliable medium that carries QUIC packets.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Extended Key Update Negotiation

QUIC peers negotiate Extended Key Update through the TLS handshake process, as outlined in {{Section 4 of I-D.ietf-tls-extended-key-update}}.
Extended Key Update MUST NOT be used unless both QUIC peers include the TLS flags extension {{!I-D.ietf-tls-tlsflags}} in the handshake and
set the "Extended_Key_Update" flag.

Once Extended Key Update has been succesfully negotiated, QUIC peers MUST only use the Extended Key Update process defined in this document.
The standard QUIC Key Update mechanism from {{Section 6 of QUIC-TLS}} MUST NOT be used for the duration of the session, as both
Key Update and Extended Key Update use the Key Phase bit to signal the use of updated keys. The Key Phase bit is initially set to 0 and
toggled to indicate a key update following the succesful post-handshake exchange of Extended Key Update messsages.

# Extended Key Update Messages

Either party MAY initiate the Extended Key Update process by sending an ExtendedKeyUpdateRequest TLS handshake message in a QUIC CRYPTO frame.
This message MUST NOT be sent before the QUIC handshake is confirmed, as described in {{Section 4.1.2 of QUIC-TLS}}, or before previous Extended
Key Update process is complete. If a QUIC endpoints receives an ExtendedKeyUpdateRequest message before the QUIC handshake is complete or during ongoing
prior Extended Key Update, it MUST terminate the connection with an error of type 0x010a, equivalent to TLS unexpected_message alert as specified
in {{Section 4.8 of QUIC-TLS}}.

Upon receiving an ExtendedKeyUpdateRequest, the recipient MUST respond with an ExtendedKeyUpdateResponse TLS handshake message within a QUIC CRYPTO frame.
If a QUIC endpoint receives an unexpected ExtendedKeyUpdateResponse message from its peer, it MUST treat this as a TLS protocol error and terminate
the connection.

Specifications of the ExtendedKeyUpdateRequest and ExtendedKeyUpdateResponse messages are provided in {{Section 5 of I-D.ietf-tls-extended-key-update}}.
Any mismatch between the negotiated NamedGroup during the initial handshake and the group used in the Extended Key Update message, or an incorrect length
of the encapsulated key MUST result in connection termination with error of type 0x012f, equivalend to TLS illegal_parameter alert.

If the Extended Key Update initiator receives a retry or rejected status in the ExtendedKeyUpdateResponse message, it MAY terminate the connection with
error of type 0x01TBD, equivalent to TLS extended_key_update_required alert.

# Updating the Traffic Secrets

After sending an ExtendedKeyUpdateResponse with accepted status, the responder derives new packet protection traffic secrets. The responder MUST continue
using the previous secrets until it has recieved a packet with the Key Phase bit flipped and has successfully unprotected it using the new keys.

After recieving and succesfully processing an ExtendedKeyUpdateResponse with accepted status, the initiator derives new packet protection traffic secrets,
flips the Key Phase bit for new packets, and uses the new write secret to protect them. The initiator MUST retain the old read secret until
it has received a packet with a flipped Key Phase bit from the responder and succesfully unprotected it using the new read secret.

Both endpoints SHOULD retain old read secrets for some time after unprotecting a packet encrypted with the new keys. Discarding old secret too early may
cause delayed packets to be discarded, which the peer may interpreted as packet loss, potentially impacting performance.

{{fig-extended-key-update}} shows this interaction graphically.

~~~aasvg
        Initiator                              Responder

[  packet using old key
    and prior Key Phase   ] -------->
                            <--------  [  packet using old key
                                           and prior Key Phase  ]
                               ...
[ExtendedKeyUpdateRequest]  -------->
                            <--------  [ExtendedKeyUpdateResponse]
[  packet using new key
   and updated Key Phase  ] -------->
                            <--------  [ packet using new key
                                         and updated Key Phase ]
                               ...
~~~
{: #fig-extended-key-update title="Extended Key Update Process in QUIC."}

QUIC endpoints MUST NOT send NewKeyUpdate TLS handshake messages. Endpoints MUST treat the receipt of a TLS NewKeyUpdate message as a connection error
of type 0x010a. QUIC endpoints that have agreed to the Extended Key Update process MUST NOT change the Key Phase bit without a succesful exchange of
Extended Key Update TLS messages. Receiving a packet with the Key Phase bit changed without a success Extended Key Update exchange MUST be treated as
a connection error of type KEY_UPDATE_ERROR (0x0e).

The design of the key deriviation function for computing the next generation of secrets is similar to the one described in
{{Section 6 of I-D.ietf-tls-extended-key-update}}.

~~~pseudocode
sk = HKDF-Extract(Transcript-Hash(ExtendedKeyUpdateRequest,
                                  ExtendedKeyUpdateResponse), secret)

secret_<n+1> = HKDF-Expand-Label(sk, "quic eku",
                                 secret_<n>, Hash.length)
~~~


The corresponding key and IV are derived from the new secret as defined in {{Section 5.1 of QUIC-TLS}}. The header protection key is not updated.

# Security Considerations

Since Extended Key Update updates the key schedule, implementations MUST ensure that peers adhere strictly to the process described in this document.
Packets with higher packet numbers MUST NOT be protected using an older generation of secrets, as this could compromise key synchronization and security.

As key exchange may be computationally intensive, responders SHOULD consider rate-limiting Extended Key Exchange requests. This can be done by responding
with retry status as outlined in {{Section 5 of I-D.ietf-tls-extended-key-update}} and terminating connections for initiators that violate the back-off timer.
This approach helps prevent excessive load on endpoints and mitigates the risk of denial-of-service attacks.


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

We would like to thank Martin Thomson and Sean Turner for their early review of this design and their invaluable feedback.
