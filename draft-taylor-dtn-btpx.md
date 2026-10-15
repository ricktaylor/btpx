---

title: Bundle Transfer Protocol - Cross-Link
abbrev: BTPX
category: std

docname: draft-taylor-dtn-btpx-latest
submissiontype: IETF # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Delay/Disruption Tolerant Networking"

keyword:

- DTN
- BPv7
- Bundle Protocol
- BTPU
- BTPX

author:

- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: rtaylor@aalyria.com

venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ricktaylor/btpx"
  latest: "https://ricktaylor.github.io/btpx/draft-taylor-dtn-btpx.html"

normative:

informative:

--- abstract

The Bundle Transfer Protocol - Unidirectional (BTPU) provides for the transfer of large binary objects, typically Bundle Protocol version 7 bundles, over a unidirectional, unreliable, frame-based link-layer protocol, and by design includes no return path.

This document defines the Bundle Transfer Protocol - Cross-Link (BTPX), an extension to BTPU for deployments where independent BTPU channels exist in both directions between a pair of nodes.  It defines how a Transfer on one channel is referenced from another, using the Session ID and Transfer number assigned by its sender, and a first application of such references, the Transfer Status Message, which allows a receiver to indicate to a sender that continued transfer of an object is unnecessary, whether because it has been completely received, or because it is unwanted.

This document deliberately defines no acknowledgement or retransmission semantics; such mechanisms are deferred to future documents.

--- middle

# Introduction

The Bundle Transfer Protocol - Unidirectional (BTPU) {{!I-D.ietf-dtn-btpu}} transfers Bundles over unidirectional, unreliable, frame-based link-layer protocols.  Because the underlying link-layer is unidirectional, the protocol includes no in-band return path: loss is handled solely by the repetition of Messages by the sender.

However, many deployments of unidirectional link-layer protocols are not unidirectional at the system level: pairs of opposing unidirectional channels frequently exist between the same two nodes, for example, a high-rate space downlink paired with a low-rate uplink, and each such channel can carry an independent instance of BTPU.  The Deployment Considerations of {{!I-D.ietf-dtn-btpu}} note that any acknowledgement system designed to provide reliability requires a logically separate path from receiver back to sender.  When such a logically separate path is itself a BTPU channel in the opposing direction, the two channels can be "cross-linked": Messages received on one channel can refer to Transfers occurring on another.

This document defines the Bundle Transfer Protocol - Cross-Link (BTPX), the minimal machinery required for such cross-linking:

- [Transfer References](#transfer-references), which identify a Transfer on any Associated Channel by the Session ID and Transfer number assigned by its sender, without requiring any additional information to be carried in the Messages of the Transfer.
- The [Transfer Status Message](#transfer-status-message), which allows a node receiving a Transfer to report to the sending node that continued transfer of the object being transferred is unnecessary, whether because it has been completely received, or because it is unwanted.

The reference mechanism is intentionally more general than the single application defined in this document: future documents can define further feedback semantics, such as selective or negative acknowledgement, built on the same Transfer References.  This document deliberately defines no segment-level acknowledgement, retransmission, or reliability semantics: a Transfer Status Message reports a disposition of a whole object, and its only effect is the suppression of unnecessary transmission.

All feedback enabled by this document is advisory.  A sender that never receives, or chooses to ignore, a Transfer Status Message behaves exactly as a sender conforming to {{!I-D.ietf-dtn-btpu}} alone, and no mechanism in this document places any additional data retention obligation on a sender.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This document uses the terms Bundle, Link-layer PDU, Message, Segment, Session, Transfer, Transfer Message, and Transfer Window as defined in {{!I-D.ietf-dtn-btpu}}.  Additionally, within the scope of this document, the following terms have specific meaning:

Channel:
: A single instance of the BTPU protocol between one sending node and one or more receiving nodes, operating over a single logical channel of a link-layer protocol, as described in {{!I-D.ietf-dtn-btpu}}.

Transfer Reference:
: The combination of the Session ID of a Session and a Transfer number, which together identify a single Transfer sent by a node, see [](#transfer-references).

Associated Channels:
: The set of Channels, in both directions, between a pair of nodes, known to each node by a-priori configuration, see [](#channel-association).

# Protocol Overview

## Channel Association {#channel-association}

This extension applies when a pair of nodes is connected by Channels in both directions.  It assumes that a-priori configuration identifies, for each Channel, the node at the far end, and hence which Channels form the set of Associated Channels between a pair of nodes.  No in-band discovery or negotiation mechanism is defined.

Because a Transfer Reference is unambiguous within the scope of the node that sent the Transfer, see [](#transfer-references), correlation does not depend on which particular Channel carries a Message: any number of Channels in each direction can be associated between a pair of nodes, and a Message received on any Associated Channel can be correlated with a Transfer on any other.

A Channel carrying Transfer Status Messages is an ordinary BTPU Channel: it can simultaneously carry its own Transfers, with the Transfer Status Messages multiplexed between the Messages of those Transfers in the usual way.  No dedicated feedback channel is required.

## Transfer References {#transfer-references}

A Transfer Reference identifies a Transfer by the Session ID of the Session in which it was sent, as announced in the Session Start Message defined in {{!I-D.ietf-dtn-btpu}}, together with its Transfer number.  A receiving node learns both from the Messages of the Channel itself, so no additional information needs to be carried in the Messages of a Transfer for it to be referenced.

A Transfer number alone is not sufficient: it is meaningful only within a single Session on a single Channel.  The Session ID distinguishes the Sessions of a Channel, for example before and after a sender restart, and a node implementing this extension MUST NOT use the same Session ID for two concurrent Sessions on Channels on which it is the sender, so that the Session ID also distinguishes its Channels.  As a Transfer Reference is only ever interpreted by the node that sent the Transfer, the Session IDs chosen by other nodes are irrelevant.

A Transfer Reference identifies a Transfer, not the object transferred: if the same object is transferred again, for example on a different Channel, or after an earlier Transfer was cancelled, the new Transfer has a different Transfer Reference.  A sending node that wishes to act on reports about an object, for example to avoid transferring it to the same node again, records the object transferred by each Transfer for as long as it wishes to act on reports about that Transfer.  This retention is a matter of local policy, and is not constrained by the Transfer Window: a report can be acted upon long after the Transfer has left the Transfer Window, provided that the sender has not discarded its record.  A Transfer number recurs within a Session only after a further 2^32 Transfers, so a sender that discards its records before then never finds a Transfer Reference ambiguous.

A Bundle Message, as defined in {{!I-D.ietf-dtn-btpu}}, carries no Transfer number, and so the object it contains cannot be referenced.  A sender that wishes to receive reports about an object small enough to be transferred in a Bundle Message MAY instead transfer it as a Transfer consisting of a single Transfer End Message, in place of the Bundle Message recommended by {{!I-D.ietf-dtn-btpu}}; this adds eight octets to the transfer of the object.

## Reporting Transfer Status {#reporting-status}

When a node receives the Messages of a Transfer, it might determine that continued transmission of the object being transferred is unnecessary, for example:

- The node has completed reassembly of the object, or already holds a complete copy, perhaps received via another route.  When the object is a BPv7 bundle, the primary block, which identifies the bundle, is normally contained in the first Segment, allowing an existing copy to be recognized early in the Transfer.
- The node has insufficient resources to store the object.
- An upper-layer protocol has indicated that the object is unwanted.

In such cases, the receiving node MAY emit a [Transfer Status Message](#transfer-status-message), carrying the Transfer Reference of the Transfer and a Status Code describing the disposition of its object, on any Associated Channel on which it is the sender.  On receipt, the original sending node can cancel the Transfer if it is still in progress, cease emitting further repetitions of its Messages, and, depending on the Status Code, refrain from transferring the object to that node again, freeing channel capacity for other Transfers.

Reporting completion is particularly valuable when the sender uses generous Message repetition as a defense against loss, as described in {{!I-D.ietf-dtn-btpu}}: in the style of a fountain code, a sender can plan to repeat the Messages of a Transfer until a Transfer Status Message with a Status Code of 'Completed' is received, or a local repetition budget is exhausted.  The absence of a report never obliges a sender to continue transmission: repetition remains bounded by local policy.

Status reporting is advisory and opportunistic.  A Transfer Status Message can be lost in transmission, like any other Message, and can be repeated by the emitting node as a defense against loss, following the repetition rules of {{!I-D.ietf-dtn-btpu}}.  Due to signal propagation delay, a sender might complete a Transfer before a report about it arrives, and a receiver might continue to receive Segments for some time after emitting a report; both are expected behavior, and the semantics of the Transfer Status Message are deliberately idempotent, such that processing a report multiple times, or after the referenced Transfer has completed, causes no harm.

Because a Channel can have multiple receiving nodes, a Transfer Status Message is evidence about the state of the single reporting node only.  Whether a sender ceases transmission on a Channel with multiple receiving nodes as a result of a report from one of them is a matter of sender policy.

# Transfer Status Message {#transfer-status-message}

The Transfer Status Message is used by a node to report to the sender of an object, via an Associated Channel, its disposition of the object transferred by the Transfer identified by a [Transfer Reference](#transfer-references), indicating that continued transmission of the object to the reporting node is unnecessary.

A Transfer Status Message has a type of TBD1.  The Message Content field is formatted as follows:

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Session ID                                                    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Transfer Number                                               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    | Status Code   |
    +-+-+-+-+-+-+-+-+

Session ID:
: The Session ID of the Session in which the referenced Transfer was sent, encoded as a 32-bit unsigned integer in network byte order.

Transfer Number:
: The Transfer number of the referenced Transfer, encoded as a 32-bit unsigned integer in network byte order.

Status Code:
: The disposition being reported, encoded as an 8-bit unsigned integer, with values allocated from the "BTPU Transfer Status Codes" IANA registry, see [](#iana-considerations).

The Message Content of a Transfer Status Message, excluding any Hint Items, is exactly 9 octets in length.  A Transfer Status Message with Message Content of any other length is invalid, as defined in {{!I-D.ietf-dtn-btpu}}.

The Transfer Status Message is not a Transfer Message: although it carries a Transfer number, that number refers to a Transfer on another Channel, and so the Message is subject to neither the Transfer Window nor the Session of the Channel on which it is emitted.

The following Status Codes are defined by this document: [^1]

| Code | Name | Description |
|:------:|:-----|:------|
| 0 | Unspecified | Continued transfer is unnecessary; no further information is available |
| 1 | Completed | The reporting node holds a complete copy of the object |
| 2 | No Resources | The reporting node currently lacks the resources to store the object |
| 3 | Not Acceptable | The reporting node will not accept the object |
{: #tab-status-codes align="left" title="Transfer Status Codes"}

## Emitting Status Reports

A node MAY emit a Transfer Status Message about any Transfer of which it has received Messages, when continued transmission of the object being transferred is unnecessary.  This includes a node that has itself just completed reassembly of the object: emitting a Transfer Status Message with a Status Code of 'Completed' informs the sender that any further repetition is unnecessary.  The Message is emitted on any Associated Channel on which the reporting node is the sender.

As with any Message, a Transfer Status Message MAY be repeated as a defense against loss, following the repetition rules of {{!I-D.ietf-dtn-btpu}}.

The emission of Transfer Status Messages is never required: a node that never emits them is indistinguishable from a node that does not implement this extension.

## Processing Status Reports

A node that receives a Transfer Status Message on a Channel uses the carried Transfer Reference to identify a Transfer that it is sending, or has recently sent, on any Associated Channel towards the node from which the report was received.

If the Transfer Reference identifies an in-progress Transfer, the node SHOULD cease emitting further Transfer Segment Messages for that Transfer, and SHOULD emit a Transfer Cancel Message, as defined in {{!I-D.ietf-dtn-btpu}}, on the Channel carrying the Transfer, so that all receiving nodes of that Channel discard any partially reassembled data.  If the Transfer has been completely emitted, but its Messages are still being repeated, the node SHOULD cease emitting further repetitions.  These reactions are the same for every Status Code: each code defined in this document indicates that continued transmission to the reporting node is unnecessary.

Whether or not the Transfer is still in progress, the node MAY record the report against the object transferred, together with its Status Code, to inform future decisions about transferring the same object to the reporting node; for example, a report with a Status Code of 'Completed' or 'Not Acceptable' suggests the object need never be sent to that node again, whereas 'No Resources' indicates a condition that might be temporary.  The retention and interpretation of recorded reports is a matter of local policy.

If the Transfer Reference does not identify a Transfer for which the node holds a record, the Message MUST be ignored.  Late-arriving reports are expected under long signal propagation delays, and are not an error.

# Security Considerations

The security considerations of {{!I-D.ietf-dtn-btpu}} apply to this document.

BTPU as defined in {{!I-D.ietf-dtn-btpu}} gives a receiving node no influence over the behavior of a sending node.  This document introduces such an influence: a forged or replayed Transfer Status Message could cause a sender to cancel a Transfer and suppress future Transfers of an object, resulting in a targeted denial of delivery that is not possible with the base protocol.  A forged Status Code of 'Completed' is particularly harmful, as it may additionally cause the sender to believe that the object was successfully delivered.

To forge a plausible report, an attacker needs a Transfer Reference that the sender will recognize.  Transfer numbers are sequential, and hence predictable, but the Session ID is chosen at random by the sender, as described in {{!I-D.ietf-dtn-btpu}}, so an off-path attacker that has not observed a Channel would need to guess a 32-bit value.  This is not intended to provide cryptographic protection: where the forgery of reports by an off-path attacker is a concern, deployments rely on link-layer mechanisms to prevent the injection of Link-layer PDUs, as they do for the other Messages of {{!I-D.ietf-dtn-btpu}}.

An on-path attacker able to observe a Channel can learn its Session ID and Transfer numbers, and subsequently forge reports on a return Channel.  Consistent with {{!I-D.ietf-dtn-btpu}}, protection against on-path attackers is a matter for link-layer security mechanisms; upper-layer mechanisms such as BPSec {{?RFC9172}} protect the transferred bundles themselves, but cannot protect the Messages of this protocol.

A sender that records reports to suppress future Transfers SHOULD bound the lifetime of such records, limiting the period during which a replayed or forged report can suppress delivery.

# Deployment Considerations

In addition to the Deployment Considerations of {{!I-D.ietf-dtn-btpu}}, which continue to apply to each Channel independently, the following caveats are to be considered before deploying this extension:

1. All feedback defined in this document is advisory and opportunistic.  Status reports can be lost in transmission; a deployment gains efficiency when they arrive, but correct operation never depends on their delivery.

1. The association of Channels between a pair of nodes requires a-priori configuration.  No in-band discovery or negotiation mechanism is defined.

1. In mixed deployments, nodes not implementing this extension will encounter the Transfer Status Message as an unrecognized Message type, and handle it accordingly, as specified in {{!I-D.ietf-dtn-btpu}}.

# IANA Considerations

## BTPU Message Types

IANA is requested to assign a new value from the "BTPU Message Types" registry, defined in {{!I-D.ietf-dtn-btpu}}, for the new Message type defined in this document:

| Type | Message | Reference |
|:------:|:-----|:------|
| TBD1 | [Transfer Status Message](#transfer-status-message) | This document |
{: #tab-message-types align="left" title="Additional BTPU Message Type"}

## BTPU Transfer Status Codes Registry

IANA is requested to create a new registry entitled "BTPU Transfer Status Codes", in the existing "Bundle Protocol" registry group.  The registration procedures for this registry, using terms defined in {{!RFC8126}}, is:

| Values | Registration Procedure |
|:-------:|:------ |
| 0..0x6F | Standards Action |
| 0x70..0x7F | Private Use |
| 0x80..0xFF | Reserved for future expansion |
{: #tab-status-codes-reg align="left" title="BTPU Transfer Status Codes registration policies"}

The initial values for the registry are given in [](#tab-status-codes).

--- back

# Examples

## Suppression of an in-progress Transfer

An example of the suppression of an in-progress Transfer between two nodes connected by one Channel in each direction is shown in [](#fig-suppression).  Node A is transferring a large Bundle to Node B as Transfer 7 of Session s1 on Channel X, where TSeg(T, I) denotes a Transfer Segment Message for Transfer T with Segment Index I.  Node B already holds a complete copy of the Bundle, received earlier via another route.

    Node A                                                Node B
       |                                                     |
       |  Channel X (A to B), Session ID s1                  |
       |  Channel Y (B to A)                                 |
       |                                                     |
       | TSeg(T=7, I=0) -----------------------------------> |
       | TSeg(T=7, I=1) -----------------------------------> |
       |                                                     |
       |           <---------- Status(Completed, s1, T=7)    |
       |                                                     |
       | TSeg(T=7, I=2) -----------------------------------> |
       |                                                     |
       | TCancel(T=7) -------------------------------------> |
       |                                                     |
{: #fig-suppression title="Suppression of an in-progress Transfer" }

On receipt of the first Transfer Segment Message, Node B recognizes from the primary block of the Bundle that it already holds a complete copy, and emits a Transfer Status Message with Status Code 'Completed', referencing Transfer 7 of Session s1, on Channel Y.  Due to the signal propagation delay, Node A emits a further Segment before the report arrives; this is expected behavior.  On receipt of the report, Node A ceases emitting Segments of Transfer 7, emits a Transfer Cancel Message on Channel X, and uses the reclaimed capacity of Channel X for other Transfers.  Node A records that Node B holds the Bundle transferred by Transfer 7, and does not offer it to Node B again.

# Acknowledgments
{:numbered="false"}

The status reporting semantics defined in this document draw on the XFER_REFUSE and XFER_ACK mechanisms of the TCP Convergence-Layer Protocol Version 4 {{?RFC9174}}.

[^1]: The initial Status Code list is loosely aligned with the XFER_REFUSE reason codes of RFC 9174, with 'Completed' recast as a positive report, minus those codes that only make sense with sessions or segment-level acknowledgement. The list needs discussing by the WG.
