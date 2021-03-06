%%%

    #
    # Solution Framework for Private Media
    # Generation tool: mmark (https://github.com/mmarkdown/mmark)
    #

    title = "A Solution Framework for Private Media in Privacy Enhanced RTP Conferencing (PERC)"
    abbrev = "Private Media Framework"
    category = "std"
    ipr= "trust200902"
    area = "Internet"
    workgroup = ""
    keyword = ["PERC", "Private Media Framework", "conferencing"]

    [seriesInfo]
    status = "standard"
    name = "Internet-Draft"
    value = "draft-ietf-perc-private-media-framework-12"
    stream = "IETF"

    [[author]]
    initials="P."
    surname="Jones"
    fullname="Paul E. Jones"
    organization = "Cisco"
      [author.address]
      email = "paulej@packetizer.com"
      phone = "+1 919 476 2048"
      [author.address.postal]
      street = "7025 Kit Creek Rd."
      city = "Research Triangle Park"
      region = "North Carolina"
      code = "27709"
      country = "USA"
    [[author]]
    initials="D."
    surname="Benham"
    fullname="David Benham"
    organization = "Independent"
      [author.address]
      email = "dabenham@gmail.com"
    [[author]]
    initials="C."
    surname="Groves"
    fullname="Christian Groves"
    organization = "Independent"
      [author.address]
      email = "cngroves.std@gmail.com"
      [author.address.postal]
      city = "Melbourne"
      country = "Australia"

    #
    # Revision History
    #   00 - Initial WG document
    #   01 - See IETF meeting slides
    #   02 - See IETF meeting slides
    #   03 - See IETF meeting slides
    #   04 - Addressed markdown rendering issue
    #        Added appendices for key and packet information
    #   05 - Clarified key exchange procedures
    #        Editorial corrections
    #   06 - Editorial improvements (https://github.com/ietf/perc-wg/pull/150)
    #   07 - Expiration refresh
    #   08 - Address comments from Ben Campbell
    #   09 - Last call comments (Gorry, Linda, and Vincent)
    #   10 - Addressed additional comments from Vincent
    #   11 - Addressed additional comments received from Vincent, Barry,
    #        Mirja, Adam, Magnus, Alissa, and Roman
    #   12 - Minor edits based on exchange with Roman
    #

%%%

.# Abstract

This document describes a solution framework for ensuring that media
confidentiality and integrity are maintained end-to-end within the
context of a switched conferencing environment where media
distributors are not trusted with the end-to-end media
encryption keys.  The solution builds upon existing security
mechanisms defined for the real-time transport protocol (RTP).

{mainmatter}

# Introduction

Switched conferencing is an increasingly popular model for multimedia
conferences with multiple participants using a combination of audio,
video, text, and other media types.  With this model, real-time media
flows from conference participants are not mixed, transcoded,
transrated, recomposed, or otherwise manipulated by a Media
Distributor, as might be the case with a traditional media server or
multipoint control unit (MCU).  Instead, media flows transmitted by
conference participants are simply forwarded by Media Distributors
to each of the other participants.  Media Distributors often forward only a subset of
flows based on voice activity detection or other criteria.  In some
instances, Media Distributors may make limited modifications to
RTP [@!RFC3550] headers, for example, but the actual media content
(e.g., voice or video data) is unaltered.

An advantage of switched conferencing is that Media Distributors can
be more easily deployed on general-purpose computing hardware,
including virtualized environments in private and public clouds.
Virtualized public cloud environments have been viewed as less
secure since resources are not always physically controlled by
those who use them.  This document defines improved security so as to
lower the barrier to taking advantage of those environments.

This document defines a solution framework wherein media privacy is
ensured by making it impossible for a Media Distributor to
gain access to keys needed to decrypt or authenticate the actual media
content sent between conference participants.  At the same time, the
framework allows for the Media Distributors to modify certain RTP
headers; add, remove, encrypt, or decrypt RTP header extensions; and
encrypt and decrypt RTP Control Protocol (RTCP) [@!RFC3550] packets.
The framework also prevents replay
attacks by authenticating each packet transmitted between a given
participant and the Media Distributor using a unique key per
Endpoint that is independent from the key for media encryption and
authentication.

This solution framework provides for enhanced privacy
in RTP-based conferencing environments while utilizing existing
security procedures defined for RTP with minimal enhancements.

# Conventions Used in This Document

The key words "**MUST**", "**MUST NOT**", "**REQUIRED**", "**SHALL**",
"**SHALL NOT**", "**SHOULD**", "**SHOULD NOT**", "**RECOMMENDED**",
"**NOT RECOMMENDED**", "**MAY**", and "**OPTIONAL**" in this document
are to be interpreted as described in BCP 14 [@!RFC2119] [@!RFC8174]
when, and only when, they appear in all capitals, as shown here.

Additionally, this solution framework uses the following
terms and acronyms:

End-to-End (E2E): Communications from one Endpoint through one or more
Media Distributors to the Endpoint at the other end.

Hop-by-Hop (HBH): Communications between an Endpoint and a Media
Distributor or between Media Distributors.

Trusted Endpoint (or simply Endpoint): An RTP flow terminating entity that has possession
of E2E media encryption keys and terminates E2E encryption.  This may
include embedded user conferencing equipment or browsers on computers,
media gateways, MCUs, media recording devices and more that are in the
trusted domain for a given deployment. In the context of WebRTC
[@W3C.CR-webrtc-20180927], where
control of a session is divided between a JavaScript application and a
browser, the browser acts as the Trusted Endpoint for purposes of this
framework (just as it acts as the endpoint for DTLS-SRTP [@RFC5764] in
one-to-one calls).

Media Distributor (MD): An RTP middlebox that forwards Endpoint media
content (e.g., voice or video data) unaltered, either a subset or all
of the flows at any given time, and is never allowed to have access
to E2E encryption keys.  It operates according to the
Selective Forwarding Middlebox RTP topologies [@RFC7667] per the
constraints defined by the PERC system, which includes, but is not limited
to, having no access to RTP media plaintext and having limits on what
RTP header field it can alter.  The header fields that may be
modified by a Media Distributor are enumerated in Section 4 of the Double
cryptographic transform specification [@!I-D.ietf-perc-double] and chosen
with respect to utility and the security considerations outlined in this
document.

Key Distributor: An entity that is a logical function which
distributes keying material and related information to Trusted
Endpoints and Media Distributor(s), only that which is appropriate for
each.  The Key Distributor might be co-resident with another entity
trusted with E2E keying material.

Conference: Two or more participants communicating via Trusted
Endpoints to exchange RTP flows through one or more Media Distributor.

Call Processing: All Trusted Endpoints in the conference connect to it
by a call processing dialog, such as with the Focus defined in the
Framework for Conferencing with SIP [@RFC4353].

Third Party: Any entity that is not an Endpoint, Media Distributor,
Key Distributor or Call Processing entity as described in this
document.

# PERC Entities and Trust Model

The following figure depicts the trust relationships, direct or
indirect, between entities described in the subsequent sub-sections.
Note that these entities may be co-located or further divided into
multiple, separate physical devices.

Please note that some entities classified as untrusted in the simple,
general deployment scenario used most commonly in this document might
be considered trusted in other deployments.  This document does not
preclude such scenarios, but keeps the definitions and examples
focused by only using the simple, most general deployment
scenario.

{#fig-trust-model align="center"}
~~~

                       |
   +----------+        |        +-----------------+
   | Endpoint |        |        | Call Processing |
   +----------+        |        +-----------------+
                       |
                       |
+----------------+     |       +--------------------+
| Key Distributor|     |       | Media Distributor  |
+----------------+     |       +--------------------+
                       |
     Trusted           |             Untrusted
     Entities          |             Entities
                       |

~~~
Figure: Trusted and Untrusted Entities in PERC


## Untrusted Entities

The architecture described in this framework document enables
conferencing infrastructure to be hosted in domains, such as in a
cloud conferencing provider's facilities, where the trustworthiness is
below the level needed to assume the privacy of participant's media
is not compromised.  The conferencing infrastructure in such a
domain is still trusted with reliably connecting the participants
together in a conference, but not trusted with keying material needed
to decrypt any of the participant's media.  Entities in such lower
trustworthiness domains are referred to as untrusted
entities from this point forward.

It is important to understand that untrusted in this document does not
mean an entity is not expected to function properly.  Rather, it means
only that the entity does not have access to the E2E media encryption
keys.

### Media Distributor

A Media Distributor forwards RTP flows between Endpoints in the
conference while performing per-hop authentication of each RTP packet.
The Media Distributor may need access to one or more RTP headers or
header extensions, potentially adding or modifying a certain subset.
The Media Distributor also relays secured messaging between the
Endpoints and the Key Distributor and acquires per-hop key
information from the Key Distributor.  The actual media content
must not be decryptable by a Media Distributor, as it is untrusted to
have access to the E2E media encryption keys.  The key exchange
mechanisms specified in this framework prevent the Media Distributor
from gaining access to the E2E media encryption keys.

An Endpoint's ability to connect to a conference serviced by a Media
Distributor does imply that the Endpoint is authorized to
have access to the E2E media encryption keys, as the Media Distributor
does not have the ability to determine whether an Endpoint is
authorized.  Instead, the Key Distributor is responsible for
authenticating the Endpoint (e.g., using WebRTC Identity
[@I-D.ietf-rtcweb-security-arch]) and determining its
authorization to receive E2E and HBH media encryption keys.

A Media Distributor must perform its role in properly forwarding
media packets while taking measures to mitigate the adverse effects of
denial of service attacks (refer to (#attacks)) to a level equal
to or better than traditional conferencing (non-PERC)
deployments.

A Media Distributor or associated conferencing infrastructure may also
initiate or terminate various conference control related messaging,
which is outside the scope of this framework document.

### Call Processing

The call processing function is untrusted in the simple, general
deployment scenario.  When a physical subset of the call processing
function resides in facilities outside the trusted domain, it should
not be trusted to have access to E2E key information.

The call processing function may include the processing of call
signaling messages, as well as the signing of those messages.  It may
also authenticate the Endpoints for the purpose of call signaling and
subsequently joining of a conference hosted through one or more Media
Distributors.  Call processing may optionally ensure the privacy of
call signaling messages between itself, the Endpoint, and other
entities.

## Trusted Entities

From the PERC model system perspective, entities considered trusted
(refer to (#fig-trust-model)) can be in possession of the E2E media
encryption keys for one or more conferences.

### Endpoint

An Endpoint is considered trusted and has access to E2E key
information.  While it is possible for an Endpoint to be compromised,
subsequently performing in undesired ways, defining Endpoint
resistance to compromise is outside the scope of this document.
Endpoints take measures to mitigate the adverse effects of denial
of service attacks (refer to (#attacks)) from other entities,
including from other Endpoints, to a level equal to or better than
traditional conference (non-PERC) deployments.

### Key Distributor {#key_distributor}

The Key Distributor, which may be colocated with an Endpoint or exist
standalone, is responsible for providing key information to Endpoints
for both end-to-end (E2E) and hop-by-hop (HBH) security and for providing key
information to Media Distributors for the hop-by-hop security.

Interaction between the Key Distributor and the call processing
function is necessary for proper conference-to-Endpoint
mappings. This is described in (#conf-id).

The Key Distributor needs to be secured and managed in a way to
prevent exploitation by an adversary, as any kind of compromise of the
Key Distributor puts the security of the conference at risk.

They Key Distributor needs to know which Endpoints and which Media
Distributors are authorized to participate in the conference.  How the
Key Distributor obtains this information is outside the scope of this
document.  However, Key Distributors **MUST** reject DTLS associations
with any unauthorized Endpoint or Media Distributor.

# Framework for PERC

The purpose for this framework is to define a means through which
media privacy is ensured when communicating within a conferencing
environment consisting of one or more Media Distributors that only
switch, hence not terminate, media.  It does not otherwise attempt to
hide the fact that a conference between Endpoints is taking place.

This framework reuses several specified RTP security technologies,
including Secure Real-time Transport Protocol (SRTP) [@!RFC3711],
Encrypted Key Transport (EKT) [@!I-D.ietf-perc-srtp-ekt-diet],
and DTLS-SRTP.

## End-to-End and Hop-by-Hop Authenticated Encryption

This solution framework focuses on the end-to-end privacy and
integrity of the participant's media by limiting access to only trusted
entities to the E2E key used for authenticated end-to-end encryption.
However, this framework does give a Media Distributor access to RTP headers
fields and header extensions, as well as the ability to modify a certain
subset of the header fields and to add or change header extensions.  Packets
received by a Media Distributor or an Endpoint are authenticated
hop-by-hop.

To enable all of the above, this framework defines the use of two
security contexts and two associated encryption keys: an "inner" key
(an E2E key distinct for each transmitted media flow) for authenticated
encryption of RTP media between Endpoints and an "outer" key (HBH key)
known only to Media Distributor or the adjacent Endpoint
for the hop between an Endpoint and a Media Distributor or peer Endpoint.
An Endpoint will receive one or more E2E keys from
every other Endpoint in the conference that correspond to the media flows
transmitted by those other Endpoints, while HBH keys are derived from
the DTLS-SRTP association with the Key Distributor.  Two communicating
Media Distributors use DTLS-SRTP associations directly with each other to
obtain the HBH keys they will use.  See (#key_exchange) for more details
on key exchange.

{#fig-e2e-and-hbh-keys-used align="center"}
~~~
+-------------+                                +-------------+
|             |################################|             |
|    Media    |------------------------ *----->|    Media    |
| Distributor |<----------------------*-|------| Distributor |
|      X      |#####################*#|#|######|      Y      |
|             |                     | | |      |             |
+-------------+                     | | |      +-------------+
   #  ^ |  #          HBH Key (XY) -+ | |         #  ^ |  #
   #  | |  #           E2E Key (B) ---+ |         #  | |  #
   #  | |  #           E2E Key (A) -----+         #  | |  #
   #  | |  #                                      #  | |  #
   #  | |  #                                      #  | |  #
   #  | |  *---- HBH Key (AX)    HBH Key (YB) ----*  | |  #
   #  | |  #                                      #  | |  #
   #  *--------- E2E Key (A)      E2E Key (A) ---------*  #
   #  | *------- E2E Key (B)      E2E Key (B) -------* |  #
   #  | |  #                                      #  | |  #
   #  | v  #                                      #  | v  #
+-------------+                                +-------------+
| Endpoint A  |                                | Endpoint B  |
+-------------+                                +-------------+
~~~
Figure: E2E and HBH Keys Used for Authenticated Encryption of SRTP
Packets

The Double transform [@!I-D.ietf-perc-double] enables Endpoints
to perform encryption using both the end-to-end and hop-by-hop contexts while
still preserving the same overall interface as other SRTP
transforms.  The Media Distributor simply uses the corresponding
normal (single) AES-GCM transform, keyed with the appropriate HBH
keys.  See [@keyinventory] for a description of the keys used in PERC
and [@packetformat] for diagram of how encrypted RTP packets appear on the
wire.

RTCP is only encrypted hop-by-hop, not end-to-end.  This framework
introduces no additional step for RTCP authenticated encryption, so
the procedures needed are specified in [@!RFC3711] and use the same
outer, hop-by-hop cryptographic context chosen in the Double operation
described above.  For this reason, Endpoints **MUST NOT** send
confidential information via RTCP.

## E2E Key Confidentiality

To ensure the confidentiality of E2E keys shared between Endpoints,
Endpoints use a common Key Encryption Key (KEK) that is
known only by the trusted entities in a conference.  That KEK, defined
in the EKT specification [@!I-D.ietf-perc-srtp-ekt-diet] as the EKT Key, is
used to subsequently encrypt the SRTP master key used for E2E
authenticated encryption of media sent by a given Endpoint.
Each Endpoint in the conference creates an SRTP master
key for E2E authenticated encryption and
keep track of the E2E keys received via the Full EKT Tag for
each distinct synchronization source (SSRC) in the conference so that it
can properly decrypt received media.  An Endpoint may change its E2E key at any
time and advertise that new key to the conference as specified in
[@!I-D.ietf-perc-srtp-ekt-diet].

## E2E Keys and Endpoint Operations {#e2e_keys_ops}

Any given RTP media flow is identified by its SSRC, and an Endpoint
might send more than one at a time and change the mix of media flows
transmitted during the life of a conference.

Thus, an Endpoint **MUST** maintain a list of SSRCs from received RTP
flows and each SSRC's associated E2E key information.  An Endpoint **MUST**
discard old E2E keys no later than when it leaves the conference
(see [@keyexchange]).

If the packet is to contain RTP header extensions, it should be noted
that those are only encrypted HBH per [@!I-D.ietf-perc-double].  For
this reason, Endpoints **MUST NOT** transmit confidential information
via RTP header extensions.

## HBH Keys and Per-hop Operations

To ensure the integrity of transmitted media packets, it is
**REQUIRED** that every packet be authenticated hop-by-hop between
an Endpoint and a Media Distributor, as well between Media
Distributors.  The authentication key used for hop-by-hop
authentication is derived from an SRTP master key shared only on the
respective hop.  Each HBH key is distinct per hop and no two hops ever
use the same SRTP master key.

While Endpoints also perform HBH authentication, the ability of the Endpoints
to reconstruct the original RTP header also enables the Endpoints to
authenticate RTP packets E2E.  This design yields flexibility to the Media
Distributor to change certain RTP header values as packets are
forwarded.  Which values the Media Distributor can change in the RTP header
are defined in
[@!I-D.ietf-perc-double].  RTCP can only be encrypted hop-by-hop, giving the
Media Distributor the flexibility to forward RTCP content unchanged,
transmit compound RTCP packets or to initiate RTCP packets for
reporting statistics or conveying other information.  Performing
hop-by-hop authentication for all RTP and RTCP packets also helps
provide replay protection (see (#attacks)).  The use of the replay
protection mechanism specified in Section 3.3.2 of [@!RFC3711] is
**REQUIRED** at each hop.

If there is a need to encrypt one or more RTP header extensions
hop-by-hop, the Endpoint derives an encryption key from the HBH SRTP
master key to encrypt header extensions as per [@!RFC6904].  This
still gives the Media Distributor visibility into header extensions,
such as the one used to determine audio level [@RFC6464] of conference
participants.  Note that when RTP header extensions are encrypted, all
hops need to decrypt and
re-encrypt these encrypted header extensions.  Please refer to
Sections 5.1 through 5.3 of [@!I-D.ietf-perc-double] for procedures
to perform RTP header extension encryption and decryption.

## Key Exchange {#key_exchange}

In brief, the keys used by any given Endpoints are determined in the
following way:

* The HBH keys that the Endpoint uses to send and receive SRTP media
  are derived from a DTLS handshake that the Endpoint performs with
  the Key Distributor (following normal DTLS-SRTP procedures).

* The E2E key that an Endpoint uses to send SRTP media can either be
  set from the DTLS-SRTP association with the Key Distributor or chosen
  by the Endpoint.  It is then distributed to other Endpoints in a
  Full EKT Tag, encrypted under an EKT Key provided to the client by the
  Key Distributor within the DTLS channel they negotiated.  Note that
  an Endpoint **MAY** create a different E2E key per media flow, where a
  media flow is identified by its SSRC value.

* Each E2E key that an Endpoint uses to receive SRTP media is set
  by receiving a Full EKT Tag from another Endpoint.

* The HBH keys used between two Media Distributors is derived from
  DTLS-SRTP procedures employed directly between them.

### Initial Key Exchange and Key Distributor

The Media Distributor maintains a tunnel with the Key Distributor
(e.g., using [@I-D.ietf-perc-dtls-tunnel]), making it
possible for the Media Distributor to facilitate the establishment of
a secure DTLS association between each Endpoint and the Key
Distributor as shown the following figure.  The DTLS association
between Endpoints and the Key Distributor enables each Endpoint to
generate E2E and HBH keys and receive the KEK.
At the same time, the Key Distributor securely
provides the HBH key information to the Media Distributor.  The key
information summarized here may include the SRTP master key, SRTP
master salt, and the negotiated cryptographic transform.

{#fig-initial-key-exchange align="center"}
~~~

                          +-----------+
                 KEK info |    Key    | HBH Key info to
             to Endpoints |Distributor| Endpoints & Media Distributor
                          +-----------+
                             # ^ ^ #
                             # | | #--- Tunnel
                             # | | #
+-----------+             +-----------+             +-----------+
| Endpoint  |   DTLS      |   Media   |   DTLS      | Endpoint  |
|    KEK    |<------------|Distributor|------------>|    KEK    |
|  HBH Key  | to Key Dist | HBH Keys  | to Key Dist |  HBH Key  |
+-----------+             +-----------+             +-----------+

~~~
Figure: Exchanging Key Information Between Entities

In addition to the secure tunnel between the Media Distributor and the
Key Distributor, there are two additional types of security associations
utilized as a part of the key exchange as discussed in the following
paragraphs.  One is a DTLS-SRTP association between an Endpoint and the Key
Distributor (with packets passing through the Media Distributor) and the
other is a DTLS-SRTP association between peer Media Distributors.

Endpoints establish a DTLS-SRTP association over the RTP session with the
Media Distributor and its media ports for the purposes of key information
exchange with the Key Distributor.  The Media Distributor does not terminate
the DTLS signaling, but instead forwards DTLS packets received
from an Endpoint on to the Key Distributor (and vice versa) via a
tunnel established between Media Distributor and the Key Distributor.

In establishing the DTLS association between Endpoints and the
Key Distributor, the Endpoint **MUST** act as the DTLS client and the
Key Distributor **MUST** act as the DTLS server.  The KEK
is conveyed by the Key Distributor over the DTLS
association to Endpoints via procedures defined in EKT
[@I-D.ietf-perc-srtp-ekt-diet] via the EKTKey message.

The Key Distributor **MUST NOT** establish DTLS-SRTP associations with
Endpoints without first authenticating the Media Distributor tunneling the
DTLS-SRTP packets from the Endpoint.

Note that following DTLS-SRTP procedures for the [@!I-D.ietf-perc-double]
cipher, the Endpoint generates both E2E and HBH encryption keys
and salt values.  Endpoints **MUST** either use the DTLS-SRTP generated E2E key
for transmission or generate a fresh E2E key.  In either case, the generated
SRTP master salt for E2E encryption **MUST** be replaced with the salt value
provided by the Key Distributor via the EKTKey message.  That is because
every Endpoint in the conference uses the same SRTP master salt.  The
Endpoint only transmits the SRTP master key (not the salt) used for E2E
encryption to other Endpoints in RTP/RTCP packets per
[@I-D.ietf-perc-srtp-ekt-diet].

Media Distributors use DTLS-SRTP directly with a peer
Media Distributor to establish the HBH key for transmitting RTP and RTCP
packets to that peer Media Distributor.  The Key Distributor does not
facilitate establishing a HBH key for use between Media Distributors.

### Key Exchange during a Conference {#keyexchange}

Following the initial key information exchange with the Key
Distributor, an Endpoint is able to encrypt media end-to-end with
an E2E key, sending that E2E key to other Endpoints encrypted with the
KEK, and is able to encrypt and authenticate RTP packets
using a HBH key.  The procedures defined do not allow the Media
Distributor to gain access to the KEK information, preventing it from
gaining access to any Endpoint's E2E key and subsequently decrypting
media.

The KEK may need to change from time-to-time during the
life of a conference, such as when a new participant joins or leaves a
conference.  Dictating if, when or how often a conference is to be
re-keyed is outside the scope of this document, but this framework
does accommodate re-keying during the life of a conference.

When a Key Distributor decides to re-key a conference, it transmits a
new EKTKey message containing the new EKT Key
to each of the conference participants.
Upon receipt of the new EKT Key, the Endpoint **MUST** create a
new SRTP master key and prepare to send that key inside a Full EKT
Field using the new EKT Key as per Section 4.5 of [@!I-D.ietf-perc-srtp-ekt-diet].
In order to allow time for all Endpoints in the conference to receive the new
keys, the sender should follow the recommendations in Section 4.7 of
[I-D.ietf-perc-srtp-ekt-diet].  On receiving a new EKT Key, Endpoints **MUST**
be prepared to decrypt EKT tags using the new key.  The EKT SPI field is
used to differentiate between tags encrypted with the old and new keys.

After re-keying, an Endpoint **SHOULD** retain prior SRTP master keys and
EKT Key for a period of time sufficient for the purpose of ensuring it can
decrypt late-arriving or out-of-order packets or packets sent by other
Endpoints that used the prior keys for a period of time after re-keying began.
An Endpoint **MAY** retain old keys until the end of the conference.

Endpoints **MAY** follow the procedures in section 5.2 of [@RFC5764]
to renegotiate HBH keys as desired.  If new HBH keys are generated,
the new keys are also delivered to the Media Distributor following
the procedures defined in [@I-D.ietf-perc-dtls-tunnel] as one possible method.

Endpoints **MAY** change the E2E encryption key used at
any time.  An Endpoint **MUST** generate a new E2E encryption key
whenever it receives a new EKT Key.  After switching to a new key,
the new key is conveyed to other Endpoints in the conference
in RTP/RTCP packets per [@!I-D.ietf-perc-srtp-ekt-diet].

# Authentication

It is important to this solution framework that the entities can
validate the authenticity of other entities, especially the Key
Distributor and Endpoints.  The details of this are outside the scope
of specification but a few possibilities are discussed in the
following sections.  The critical requirements are that an Endpoint can verify
it is connected to the correct Key Distributor for the conference
and the Key Distributor can verify the Endpoint is the correct
Endpoint for the conference.

Two possible approaches to solve this are Identity Assertions and
Certificate Fingerprints.

## Identity Assertions

WebRTC Identity assertion [@I-D.ietf-rtcweb-security-arch] is used
to bind the identity of the user of the Endpoint to the fingerprint of
the DTLS-SRTP certificate used for the call.  This certificate is
unique for a given call and a conference.  This allows the Key
Distributor to ensure that only authorized users participate in the
conference. Similarly the Key Distributor can create a WebRTC Identity
assertion to bind the fingerprint of the unique certificate used by
the Key Distributor for this conference so that the Endpoint can
validate it is talking to the correct Key Distributor. Such a setup
requires an Identity Provider (Idp) trusted by the Endpoints and the
Key Distributor.

## Certificate Fingerprints in Session Signaling

Entities managing session signaling are generally assumed to be
untrusted in the PERC framework.  However, there are some deployment
scenarios where parts of the session signaling may be assumed
trustworthy for the purposes of exchanging, in a manner that can be
authenticated, the fingerprint of an entity's certificate.

As a concrete example, SIP [@RFC3261] and
Session Description Protocol (SDP) [@RFC4566] can be used
to convey the fingerprint information per [@RFC5763].  An Endpoint's
SIP User Agent would send an INVITE message containing SDP for the
media session along with the Endpoint's certificate fingerprint, which
can be signed using the procedures described in [@RFC8224] for the
benefit of forwarding the message to other entities by the Focus
[@RFC4353].  Other entities can verify the fingerprints match the
certificates found in the DTLS-SRTP connections to find the identity
of the far end of the DTLS-SRTP connection and verify that is the
authorized entity.

Ultimately, if using session signaling, an Endpoint's certificate
fingerprint would need to be securely mapped to a user and conveyed to
the Key Distributor so that it can check that that user is authorized.
Similarly, the Key Distributor's certificate fingerprint can be
conveyed to an Endpoint in a manner that can be authenticated as being an
authorized Key Distributor for this conference.

## Conference Identification {#conf-id}

The Key Distributor needs to know what Endpoints are being added to a
given conference. Thus, the Key Distributor and the Media Distributor
need to know Endpoint-to-conference mappings, which is enabled by
exchanging a conference-specific unique identifier defined in
[@I-D.ietf-perc-dtls-tunnel].  How this unique identifier is assigned
is outside the scope of this document.

# PERC Keys

This section describes the various keys employed by PERC.

## Key Inventory and Management Considerations {#keyinventory}

This section summarizes the several different keys used in the PERC framework,
how they are generated, and what purpose they serve.

The keys are described in the order in which they would typically be
acquired.

The various keys used in PERC are shown in
[@key-inventory-table] below.

{#key-inventory-table align="center"}
~~~
+-----------+----------------------------------------------------+
| Key       | Description                                        |
+-----------+----------------------------------------------------+
| HBH Key   | SRTP master key used to encrypt media hop-by-hop.  |
+-----------+----------------------------------------------------+
| KEK       | Key shared by all Endpoints and used to encrypt    |
| (EKT Key) | each Endpoint's E2E SRTP master key so receiving   |
|           | Endpoints can decrypt media.                       |
+-----------+----------------------------------------------------+
| E2E Key   | SRTP master key used to encrypt media end-to-end.  |
+-----------+----------------------------------------------------+
~~~
Figure: Key Inventory

While the number of key types is very small, it should be understood that
the actual number of distinct keys can be large as the conference
grows in size.

As an example, with 1,000 participants in a conference, there would be at
least 1,000 distinct SRTP master keys, all of which share the same master salt.
Each of those keys are passed through the KDF defined in [@RFC3711] to produce
the actual encryption and authentication keys.

Complicating key management is the fact that the KEK can change and, when
it does, the Endpoints generate new SRTP master keys that are associated with
a new EKT SPI.  Endpoints might retain old keys for a period of time to
ensure they can properly decrypt late-arriving or out-of-order packets, which
means the number of keys held during that period of time might substantially
more.

A more detailed explanation of each of the keys follows.

## DTLS-SRTP Exchange Yields HBH Keys

The first set of keys acquired are for hop-by-hop encryption and
decryption.  Per the Double [@!I-D.ietf-perc-double] procedures, the
Endpoint performs DTLS-SRTP exchange with the Key Distributor
and receives a key that is, in fact, "double" the size that is needed.
The end-to-end part is the first half of the key, so the Endpoint discards
that information when generating its own key.  The second half of the key
material is for hop-by-hop operations, so that half of the key
(corresponding to the least significant bits) is assigned internally as
the HBH key.

The Key Distributor informs the Media Distributor of the HBH key.  Specifically,
the Key Distributor sends the least significant bits corresponding to the
half of the keying material determined through DTLS-SRTP with the Endpoint
to the Media Distributor.  A salt value is
generated along with the HBH key.  The salt is also longer than needed
for hop-by-hop operations, thus only the least significant bits of the
required length (half of the generated salt material) are sent to the
Media Distributor.  One way to transmit this key and salt information
is via the tunnel protocol defined in [@I-D.ietf-perc-dtls-tunnel].

No two Endpoints have the same HBH key, thus the Media Distributor
**MUST** keep track each distinct HBH key (and the corresponding salt) and
use it only for the specified hop.

The HBH key is also used for hop-by-hop encryption of RTCP.  RTCP is not
end-to-end encrypted in PERC.

## The Key Distributor Transmits the KEK (EKT Key)

Via the aforementioned DTLS-SRTP association, the Key Distributor
sends the Endpoint the KEK (EKT Key per
[@!I-D.ietf-perc-srtp-ekt-diet]).  This key is known only to
the Key Distributor and Endpoints.  This key is the most important to
protect since having knowledge of this key (and the SRTP master salt
transmitted as a part of the same message) allows an entity to
decrypt any media packet in the conference.

Note that the Key Distributor can send any number of EKT Keys to
Endpoints.  This is used to re-key the entire conference.  Each
key is identified by a "Security Parameter Index" (SPI) value.
Endpoints **MUST** expect that a conference might be re-keyed
when a new participant joins a conference or when a participant
leaves a conference in order to protect the confidentiality of
the conversation before and after such events.

The SRTP master salt to be used by the Endpoint is transmitted along
with the EKT Key.  All Endpoints in the conference utilize
the same SRTP master salt that corresponds with a given EKT Key.

The Full EKT Tag in media packets is encrypted using a cipher specified
via the EKTKey message (e.g., AES Key Wrap with a 128-bit key).  This
cipher is different than the cipher used to protect media and is only
used to encrypt the Endpoint's SRTP master key (and other EKT Tag data
as per [@!I-D.ietf-perc-srtp-ekt-diet]).

The Media Distributor is not given the KEK.

## Endpoints fabricate an SRTP Master Key

As stated earlier, the E2E key determined via DTLS-SRTP **MAY** be
discarded in favor of a locally-generated E2E SRTP master key.  While the
DTLS-SRTP-derived SRTP master key can be used initially, the Endpoint might
choose to change the SRTP master key periodically and **MUST** change the
SRTP master key as a result of the EKT key changing.

A locally-generated SRTP master key is used along with the master salt
transmitted to the Endpoint from the Key Distributor via the EKTKey
message to encrypt media end-to-end.

Since the Media Distributor is not involved in E2E functions, it does not
create this key nor have access to any Endpoint's E2E key.  Note, too,
that even the Key Distributor is unaware of the locally-generated E2E keys
used by each Endpoint.

The Endpoint transmits its E2E key to other Endpoints in the conference
by periodically including it in SRTP packets in a Full EKT Tag.  When
placed in the Full EKT Tag, it is encrypted using the EKT Key provided
by the Key Distributor.  The master salt is not transmitted, though,
since all Endpoints receive the same master salt via the EKTKey
message from the Key Distributor.  The recommended frequency with which an
Endpoint transmits its SRTP master key is specified in
[@!I-D.ietf-perc-srtp-ekt-diet].

## Summary of Key Types and Entity Possession

All Endpoints have knowledge of the KEK.

Every HBH key is distinct for a given Endpoint, thus Endpoint A and
Endpoint B do not have knowledge of the other's HBH key.  Since HBH keys
are derived from a DTLS-SRTP association, there is at most one HBH key
per Endpoint,  (The only exception is where the DTLS-SRTP association might
be rekeyed per Section 5.2 of [@RFC5764] and a new key is created to
replace the former key.)

Each Endpoint generates its own E2E key (SRTP master key), thus
there is a distinct E2E key per endpoint.  This key is transmitted (encrypted) via
the Full EKT Tag to other Endpoints.  Endpoints that receive media from
a given transmitting Endpoint gain knowledge of the
transmitter's E2E key via the Full EKT Tag.

To summarize the various keys and which entity is in possession
of a given key, refer to [@fig-who-has-what-key].

{#fig-who-has-what-key align="center"}
~~~
+----------------------+------------+-------+-------+------------+
| Key     /    Entity  | Endpoint A |  MD X |  MD Y | Endpoint B |
+----------------------+------------+-------+-------+------------+
| KEK (EKT Key)        |    Yes     |  No   |  No   |     Yes    |
+----------------------+------------+-------+-------+------------+
| E2E Key (A and B)    |    Yes     |  No   |  No   |     Yes    |
+----------------------+------------+-------+-------+------------+
| HBH Key (A<=>MD X)   |    Yes     |  Yes  |  No   |     No     |
+----------------------+------------+-------+-------+------------+
| HBH Key (B<=>MD Y)   |    No      |  No   |  Yes  |     Yes    |
+----------------------+------------+---------------+------------+
| HBH Key (MD X<=>MD Y)|    No      |  Yes  |  Yes  |     No     |
+----------------------+------------+---------------+------------+
~~~
Figure: Keys Types and Entity Possession

# Encrypted Media Packet Format {#packetformat}

[@fig-perc-packet-format] presents a complete picture of what an encrypted
media packet per this framework looks like when transmitted over the wire.
The packet format shown is encrypted using the Double cryptographic transform
with an EKT Tag appended to the end.

{#fig-perc-packet-format align="center"}
~~~
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<++
    |V=2|P|X|  CC   |M|     PT      |       sequence number         | IO
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ IO
    |                           timestamp                           | IO
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ IO
    |           synchronization source (SSRC) identifier            | IO
    +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+ IO
    |            contributing source (CSRC) identifiers             | IO
    |                               ....                            | IO
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+O
    |                    RTP extension (OPTIONAL) ...               | |O
+>+>+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+O
O I |                          payload  ...                         | IO
O I |                               +-------------------------------+ IO
O I |                               | RTP padding   | RTP pad count | IO
O +>+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+<+O
O | |                    E2E authentication tag                     | |O
O | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |O
O | |                            OHB ...                            | |O
+>| +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ |+
| | |                    HBH authentication tag                     | ||
| | +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ ||
| | |   EKT Tag ...   | R                                             ||
| | +-+-+-+-+-+-+-+-+-+ |                                             ||
| |                     +- Neither encrypted nor authenticated;       ||
| |                        appended after Double is performed         ||
| |                                                                   ||
| |                                                                   ||
| +- E2E Encrypted Portion               E2E Authenticated Portion ---+|
|                                                                      |
+--- HBH Encrypted Portion               HBH Authenticated Portion ----+

    I = Inner (E2E) encryption / authentication
    O = Outer (HBH) encryption / authentication

~~~
Figure: Encrypted Media Packet Format

# Security Considerations {#attacks}

##  Third Party Attacks

Third party attacks are attacks attempted by an adversary that is not
supposed to have access to keying material or is otherwise not an
authorized participant in the conference.

On-path attacks are mitigated by hop-by-hop integrity protection and
encryption.  The integrity protection mitigates packet modification
and encryption makes selective blocking of packets harder, but not
impossible.

Off-path attackers could try connecting to different PERC entities to
send specifically crafted packets with an aim of forcing the receiver to
forward or render bogus media packets.  Endpoints and Media Distributors mitigate
such an attack by performing hop-by-hop authentication and discarding packets
that fail authentication.

Another attack vector is a third party claiming to be a Media
Distributor, fooling Endpoints into sending packets to the false
Media Distributor instead of the correct one.  The deceived sending
Endpoints could incorrectly assume their packets have been delivered
to Endpoints when they in fact have not.  While this attack is possible,
the result is a simple denial of service with no leakage of confidential
information, since the false Media Distributor would not have access
to either HBH or E2E encryption keys.

A third party could cause a denial-of-service by transmitting many bogus
or replayed packets toward receiving devices that ultimately degrade
conference or device performance.  Therefore, implementations might wish to
devise mechanisms to safeguard against such illegitimate packets, such as
utilizing rate-limiting or performing basic sanity-checks on packets
(e.g., looking at packet length or expected sequence number ranges) before
performing more expensive decryption operations.

Use of mutual DTLS authentication (as required by DTLS-SRTP) also helps to
prevent a denial-of-service attack by preventing a false Endpoint or false
Media Distributor from successfully participating as a perceived valid media
sender that could otherwise carry out an on-path attack.  When mutual
authentication fails, a receiving Endpoint would know that it could safely
discard media packets received from the Endpoint without inspection.

##   Media Distributor Attacks

A malicious or compromised Media Distributor can attack the session in a
number of possible ways.

###   Denial of service

A simple form of attack is discarding received packets that should be
forwarded.  This solution framework does not introduce any mitigation for
Media Distributors that fail to forward media packets.

Another form of attack is modifying received packets before forwarding.
With this solution framework, any modification of the end-to-end
authenticated data results in the receiving Endpoint getting an integrity
failure when performing authentication on the received packet.

The Media Distributor can also attempt to perform resource consumption
attacks on the receiving Endpoint.  One such attack would be to insert
random SSRC/CSRC values in any RTP packet along with a Full EKT Tag.
Since such a message would trigger the receiver to form a new cryptographic
context, the Media Distributor can attempt to consume the receiving
Endpoint's resources.  While E2E authentication would fail and the
cryptographic context would be destroyed, the key derivation operation
would nonetheless consume some computational resources.  While resource
consumption attacks cannot be mitigated entirely, rate-limiting packets
might help reduce the impact of such attacks.

###  Replay Attack

A replay attack is when an already received packet from a previous
point in the RTP stream is replayed as new packet.  This could, for
example, allow a Media Distributor to transmit a sequence of packets
identified as a user saying "yes", instead of the "no" the user
actually said.

A replay attack is mitigated by the requirement to implement
replay protection as
described in Section 3.3.2 of [@!RFC3711].
End-to-end replay protection **MUST** be provided for the
duration of the conference.

###  Delayed Playout Attack

A delayed playout attack is one where media is received and held by
a Media Distributor and then forwarded to Endpoints at a later point
in time.

This attack is possible even if E2E replay protection is in place.
Because the Media Distributor is allowed to select a
subset of streams and not forward the rest to a receiver, such as in
forwarding only the most active speakers, the receiver has to accept
gaps in the E2E packet sequence.  The issue with this is that a Media
Distributor can select to not deliver a particular stream for a while.

While the Media Distributor can purposely stop forwarding media flows, it
can also select an arbitrary starting point to resume forwarding those
media flow, perhaps forwarding old packets rather than current packets.
As a consequence, what the media source sent can be substantially delayed
at the receiver with the receiver believing that newly arriving packets
are delayed only by transport delay when the packets may actually be
minutes or hours old.

While this attack cannot be eliminated entirely, its effectiveness
can be reduced by re-keying the conference periodically since
significantly-delayed media encrypted with expired keys would not be
decrypted by Endpoints.

###  Splicing Attack

A splicing attack is an attack where a Media Distributor receiving
multiple media sources splices one media stream into the other.  If
the Media Distributor were able to change the SSRC without the receiver
having any method for verifying the original source ID, then the Media
Distributor could first deliver stream A and then later forward stream
B under the same SSRC as stream A was previously using.  By including
the SSRC in the integrity check for each packet, both HBH and E2E, PERC
prevents splicing attacks.

###  RTCP Attacks

PERC does not provide end-to-end protection of RTCP messages.  This allows
a compromised Media Distributor to impact any message that might be
transmitted via RTCP, including media statistics, picture requests, or loss
indication.  It is also possible for a compromised Media Distributor to forge
requests, such as requests to the Endpoint to send a new picture.  Such
requests can consume significant bandwidth and impair conference performance.

##   Key Distributor Attacks

As stated in (#key_distributor), the Key Distributor needs to be secured
since exploiting the Key Server can allow an adversary to gain access to
the keying material for one or more conferences.  Having access to that
keying material would then allow the adversary to decrypt media sent from
any Endpoint in the conference.

As a first line of defense, the Key Distributor authenticates every
security association, both associations with Endpoints and Media
Distributors.  The Key Distributor knows which entities are authorized to
have access to which keys and inspection of certificates will substantially
reduce the risk of providing keys to an adversary.

Both physical and network access to the Key Distributor should be severely
restricted.  This may be more difficult to achieve when the Key Distributor
is embedded within and Endpoint, for example.  Nonetheless, consideration
should be given to shielding the Key Distributor from unauthorized access
or any access that is not strictly necessary for the support of an
ongoing conference.

Consideration should be given to whether access to the keying material
will be needed beyond the conclusion of a conference.  If not needed,
the Key Distributor's policy should be to destroy the keying material
once the conference concludes or when keying material changes during
the course of the conference.  If keying material is needed beyond the
lifetime of the conference, further consideration should be given to
protecting keying material from future exposure.  While it might be
obvious, it is worth stating to avoid any doubt that if an adversary were
to record the media packets transmitted during a conference and then
gain unauthorized access to the keying material left unsecured on the
Key Distributor even years later, the adversary could decrypt the
content every packet transmitted during the conference.

##   Endpoint Attacks

A Trusted Endpoint is so named because conference confidentiality relies
heavily on the security and integrity of the Endpoint.  If an adversary
successfully exploits a vulnerability in an Endpoint, it might be possible
for the adversary to obtain all of the keying material used in the
conference.  With that keying material, an adversary could decrypt any
of the media flows received from any other Endpoint, either in real-time
or at a later point in time (assuming the adversary makes a copy of the
media packets).

Additionally, if an adversary successfully exploits an Endpoint, the
adversary could inject media into the conference. One way an adversary
could do that is by manipulating the RTP or SRTP software to transmit
whatever media the adversary wishes to send.  This could involve
re-use of the Endpoint's SSRC, a new SSRC, or the SSRC value of an existing
endpoint.  This is made possible since all conference participants share
the same KEK.  Only a single SRTP cipher suite defined provides source
authentication properties that allow an endpoint to cryptographically
assert that it sent a particular E2E protected packet (namely, TESLA
[@RFC4383]), and its usage is presently not defined for PERC.  The suite
defined in PERC only allows an Endpoint to determine that whoever sent a
packet had received the KEK.

However, attacks on the endpoint are not limited to the PERC-specific
software within the Endpoint.  An attacker could inject media or record
media by manipulating the software that sits between the PERC-enabled
application and the hardware microphone of video camera, for example.
Likewise, an attacker could potentially access confidential media by
accessing memory, cache, disk storage, etc. if the endpoint is no secured.

# IANA Considerations

There are no IANA considerations for this document.

# Acknowledgments

The authors would like to thank Mo Zanaty, Christian Oien, and Richard Barnes
for invaluable input on this document.  Also, we would like to acknowledge
Nermeen Ismail for serving on the initial versions of this document as
a co-author.  We would also like to acknowledge John Mattsson, Mats Naslund,
and Magnus Westerlund for providing some of the text in the document,
including much of the original text in the security considerations section.

{backmatter}
