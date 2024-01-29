---
title: "More Instant Messaging Interoperability (MIMI) using HTTPS and MLS"
abbrev: "MIMI+MLS Protocol"
category: std

docname: draft-ralston-mimi-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - mimi
 - interoperable messaging
 - chat
 - secure messaging
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "bifurcation/ietf-mimi-protocol"
  latest: "https://bifurcation.github.io/ietf-mimi-protocol/draft-ralston-mimi-protocol.html"

author:
 -
    fullname: Travis Ralston
    organization: The Matrix.org Foundation C.I.C.
    email: travisr@matrix.org
 -
    fullname: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de
 -
    fullname: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com
 -
    fullname: Matthew Hodgson
    organization: The Matrix.org Foundation C.I.C.
    email: matthew@matrix.org

contributor:
- name: Rohan Mahy
  org: Wire
  email: rohan.mahy@wire.com

normative:

informative:

--- abstract

This document specifies the More Instant Messaging Interoperability (MIMI)
protocol, which allows users of different messaging providers to share
membership in rooms and send messages to one-another.

More specifically, the MIMI protocol defines message formats and conventions
that allows the servers of different providers to interoperate, leaving the
provider-internal client-server communication up to the provider.

To ensure that communications between users are confidential and authentic, the
MIMI protocol relies on MLS ({{!RFC9420}}, through the MIMI DS protocol
{{!I-D.robert-mimi-delivery-service}}) to protect messages end-to-end.

--- middle

# Introduction

Providers of secure messaging services typically allow their users to only
communicate with other users of the same provider. This can be limiting for
users, who frequently have to run clients of multiple messaging applications in
parallel, each for a different secure messaging provider.

The More Instant Messaging Interoperability (MIMI) protocol specified in this
document allows interoperability on the provider level based on rooms. Each MIMI
protocol room is hosted at a single provider, but allows users from different
providers to become participants in the room.

Users and providers can use events to interact with the provider hosting the
room and with one-another, for example, to add or remove users from the room.

Each user can have clients in the room, who can then communicate with
one-another, where messages are end-to-end encrypted using the Messaging Layer
Security (MLS) protocol ({{!RFC9420}}).

An MLS group (through the MIMI DS protocol
{{!I-D.robert-mimi-delivery-service}}) forms the cryptographic core of each room
and ensures that clients in the room agree on room state and group state such as
user and client level membership.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms and definitions are inherited from {{!I-D.barnes-mimi-arch}}.

Throughout this document, the examples use the TLS Presentation Language
{{!RFC8446}} and the semantics of HTTP {{!RFC7231}} respectively as
placeholder a set of binary encoding mechanism and transport semantics.

> **ISSUE**: Come to consensus on a specific binary encoding for the MIMI
> transport protocol.

> **ISSUE**: Come to consensus on a specific set of MIMI transport protocol
> semantics.

# Protocol Overview

As shown in {{fig-layers}}, the overall MIMI protocol is built out of three layers:

1. An application layer that enables messaging functionality
2. A security layer that provides end-to-end security guarantees:
    * Confidentiality for messages
    * Authentication of actors making changes to rooms
    * Agreement on room state across the clients involved in a room
3. A transport layer that provides secure delivery of protocol objects between
   servers.

The MIMI transport is based on HTTPS over mutually-authenticated TLS.  MIMI uses
MLS {{!RFC9420}} for end-to-end security, using the MLS AppSync proposal type to
efficiently synchronize room state across the clients involved in a room.

~~~ aasvg
+------------------------+
|       Application      |
|       +----------------+
|       |  E2E Security  |
+-------+----------------+
|        Transport       |
+------------------------+
~~~
{: #fig-layers title="MIMI protocol layering" }

The remainder of this section walks through a basic scenario that illustrates
how a room works in the MIMI protocol.  The scenario involves the following
actors:

* Service providers `a.example`, `b.example`, and `c.example` represented by
  servers `ServerA`, `ServerB`, and `ServerC` respectively
* Users Alice (`alice@a.example`), Bob (`bob@b.example`) and Cathy
  (`cathy@c.example`) of the respective service providers
* Clients `ClientA1`, `ClientA2`, `ClientB1`, etc. belonging to these users
* A room `clubhouse@a.example` where the three users interact

As noted in {{!I-D.mimi-arch}}, the MIMI protocol only defines interactions
between service providers' servers.  Interactions between clients and servers
within a service provider domain are shown here for completeness, but
surrounded by `[[ double brackets ]]`.

## Alice Creates a Room

The first step in the lifetime of a MIMI room is its creation on the hub server.
This operation is local to the service provider, and does not entail any MIMI
protocol operations.  However, it must establish the initial state of the room,
which is then the basis for protocol operations related to the room.

Here, we assume that Alice uses ClientA1 to create a room with the following
properties:

* Identifier: `clubhouse@a.example`
* Participants: `[alice@a.example]`

ClientA1 also creates an MLS group with group ID `clubhouse@a.example` and
ensures via provider-local operations that Alice's other clients are members of
this MLS group.

## Alice adds Bob to the Room

Adding Bob to the room entails operations at two levels.  First, Bob's user
identity must be added to the room's participant list.  Second, Bob's clients
must be added to the room's MLS group.

The process of adding Bob to the room thus begins by Alice fetching key material
for Bob's clients.  Alice then updates the room by sending an MLS Commit over
the following proposals:

* An AppSync proposal updating the room state by adding Bob to the
  participant list
* Add proposals for Bob's clients

The MIMI protocol interactions are between Alice's server ServerA and Bob's
server ServerB.  ServerB stores KeyPackages on behalf of Bob's devices.  ServerA
performs the key material fetch on Alice's behalf, and delivers the resulting
KeyPackages to Alice's clients.  Both ServerA and ServerB remember the sources
of the KeyPackages they handle, so that they can route a Welcome mesasage for
those KeyPackages to the proper recipients -- ServerA to ServerB, and ServerB to
Bob's clients.

[[ NOTE: In the full protocol, it will be necessary to have consent and access
control on these operations.  We have elided that step here in the interest of
simplicity. ]]

~~~ aasvg
ClientA1       ServerA         ServerB         ClientB*
  |               |               |               |
  |               |               |     Store KPs |
  |               |               |<~~~~~~~~~~~~~~+
  |               |               |<~~~~~~~~~~~~~~+
  | Request KPs   |               |               |
  +~~~~~~~~~~~~~~>| /keyMaterial  |               |
  |               +-------------->|               |
  |               |        200 OK |               |
  |           KPs |<--------------+               |
  |<~~~~~~~~~~~~~~+               |               |
  |               |               |               |

ClientB*->ServerB: [[ Store KeyPackages ]]
ClientA1->ServerA: [[ request KPs for bob@b.example ]]
ServerA->ServerB: POST /keyMaterial KeyMaterialRequest
ServerB: Verify that Alice is authorized to fetch KeyPackages
ServerB: Mark returned KPs as reserved for Alice’s use
ServerB->ServerA: 200 OK KeyMaterialResponse
ServerA: Remember that these KPs go to b.example
ServerA->ClientA1: [[ KPs ]]
~~~
{: #fig-ab-kp-fetch title="Alice Fetches KeyPackages for Bob's Clients" }

~~~ aasvg
ClientA1       ServerA         ServerB         ClientB*
  |               |               |               |
  | Commit, etc.  |               |               |
  +~~~~~~~~~~~~~~>| /notify       |               |
  |               +-------------->| Welcome, Tree |
  |               |               +~~~~~~~~~~~~~~>|
  |               |               +~~~~~~~~~~~~~~>|
  |               |        200 OK |               |
  |      Accepted |<--------------+               |
  |<~~~~~~~~~~~~~~+               |               |
  |               |               |               |

ClientA1: Prepare Commit over AppSync(+bob@b.example), Add*
ClientA1->ServerA: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerA: Verify that AppSync, Adds are allowed by policy
ServerA: Identifies Welcome domains based on KP hash in Welcome
ServerA->ServerB: POST /notify/clubhouse@a.example Intro{ Welcome, RatchetTree? }
ServerB: Recognizes that Welcome is adding Bob to room clubhouse@a.example
ServerB->ClientB*: [[ Welcome, RatchetTree? ]]
~~~
{: #fig-ab-add title="Alice Adds Bob to the Room and Bob's Clients to the MLS Group" }


## Bob adds Cathy to the Room

The process of adding Bob was a bit abbreviated because Alice is a user of the
hub service provider.  When Bob adds Cathy, we see the full process, involving
the same two steps (KP fetch followed by add), but this time indirected via the
hub server ServerA.  Also, now that there are users on ServerB involved in the
room, the hub ServerA will have to distribute the Commit adding Cathy and
Cathy's clients to ServerB as well as forwarding the Welcome to ServerC.

~~~ aasvg
ClientB1       ServerB         ServerA         ServerC         ClientC*
  |               |               |               |               |
  |               |               |               |     Store KPs |
  |               |               |               |<~~~~~~~~~~~~~~+
  |               |               |               |<~~~~~~~~~~~~~~+
  | Request KPs   |               |               |               |
  +~~~~~~~~~~~~~~>| /keyMaterial  | /keyMaterial  |               |
  |               +-------------->+-------------->|               |
  |               |        200 OK |        200 OK |               |
  |           KPs |<--------------+<--------------+               |
  |<~~~~~~~~~~~~~~+               |               |               |
  |               |               |               |               |

ClientC*->ServerC: [[ Store KeyPackages ]]
ClientB1->ServerB: [[ request KPs for bob@b.example ]]
ServerB->ServerA: POST /keyMaterial KeyMaterialRequest
ServerA->ServerC: POST /keyMaterial KeyMaterialRequest
ServerB: Verify that Bob is authorized to fetch KeyPackages
ServerB: Mark returned KPs as reserved for Bob’s use
ServerC->ServerA: 200 OK KeyMaterialResponse
ServerA: Remember that these KPs go to b.example
ServerA->ServerB: 200 OK KeyMaterialResponse
ServerB->ClientB1: [[ KPs ]]
~~~
{: #fig-bc-kp-fetch title="Bob Fetches KeyPackages for Cathy's Clients" }

~~~ aasvg
ClientB1       ServerB         ServerA         ServerC         ClientC*  ClientB*  ClientA*
  |               |               |               |               |         |         |
  | Commit, etc.  |               |               |               |         |         |
  +~~~~~~~~~~~~~~>| /update       |               |               |         |         |
  |               +-------------->|               |               |         |         |
  |               |        200 OK |               |               |         |         |
  |               |<--------------+               |               |         |         |
  |      Accepted |               | /notify       |               |         |         |
  |<~~~~~~~~~~~~~~+               +-------------->| Welcome, Tree |         |         |
  |               |               |               +~~~~~~~~~~~~~~>|         |         |
  |               |               |               +~~~~~~~~~~~~~~>|         |         |
  |               |       /notify |               |               |         |         |
  |               |<--------------+               |               |         |         |
  |               | Commit        |               |               |         |         |
  |               +~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|         |
  |               |               | Commit        |               |         |         |
  |               |               +~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|
  |               |               |               |               |         |         |

ClientB1: Prepare Commit over AppSync(+cathy@c.example), Add*
ClientB1->ServerB: [[ Commit, Welcome, GroupInfo?, RatchetTree? ]]
ServerB->ServerA: POST /update/clubhouse@a.example CommitBundle
ServerA: Verify that Adds are allowed by policy
ServerA->ServerB: 200 OK
ServerA->ServerC: POST /notify/clubhouse@a.example Intro{ Welcome, RatchetTree? }
ServerC: Recognizes that Welcome is adding Cathy to clubhouse@a.example
ServerC->ClientC*: [[ Welcome, RatchetTree? ]]
ServerA->ServerB: POST /notify/clubhouse@a.example Commit
ServerB->ClientB*: [[ Commit ]]
ServerA->ClientA*: [[ Commit ]]
~~~
{: #fig-bc-add title="Bob Adds Cathy to the Room and Cathy's Clients to the MLS Group" }

## Cathy Sends a Message

Now that Alice, Bob, and Cathy are all in the room, Cathy wants to say hello to
everyone.  Cathy's client encapsulates the message in an MLS PrivateMessage and
sends it to ServerC, who forwards it to the hub ServerA on Cathy's behalf.
Assuming Cathy is allowed to speak in the room, ServerA will forward Cathy's
message to the other servers involved in the room, who distribute it to their
clients.

~~~ aasvg
ClientC1       ServerC         ServerA         ServerB         ClientB*  ClientC*  ClientA*
  |               |               |               |               |         |         |
  | Message       |               |               |               |         |         |
  +~~~~~~~~~~~~~~>| /submit       |               |               |         |         |
  |               +-------------->|               |               |         |         |
  |               |        200 OK |               |               |         |         |
  |               |<--------------+               |               |         |         |
  |      Accepted |               | /notify       |               |         |         |
  |<~~~~~~~~~~~~~~+               +-------------->| Message       |         |         |
  |               |               |               +~~~~~~~~~~~~~~>|         |         |
  |               |       /notify |               |               |         |         |
  |               |<--------------+               |               |         |         |
  |               | Message       |               |               |         |         |
  |               +~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|         |
  |               |               | Message       |               |         |         |
  |               |               +~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~>|
  |               |               |               |               |         |         |

ClientC1->ServerC: [[ MLSMessage(PrivateMessage) ]]
ServerC->ServerA: POST /submit/clubhouse@a.example MLSMessage(PrivateMessage)
ServerA: Verifies that message is allowed
ServerA->ServerC: POST /notify/clubhouse@a.example Message{ MLSMessage(PrivateMessage) }
ServerA->ServerB: POST /notify/clubhouse@a.example Message{ MLSMessage(PrivateMessage) }
ServerA->ClientA*: [[ MLSMessage(PrivateMessage) ]]
ServerB->ClientB*: [[ MLSMessage(PrivateMessage) ]]
ServerC->ClientC*: [[ MLSMessage(PrivateMessage) ]]
~~~
{: #fig-c-msg title="Cathy Sends a Message to the Room" }

## Bob Leaves the Room

A user removing another user follows the same flow as adding the user.  The
user performing the removal creates an MLS commit covering Remove proposals for
all of the removed user's devices, and an AppSync proposal updating the room
state to remove the removed user from the room's participant list.

Leaving is slightly more complicated because the leaving user cannot remove all
of their devices from the MLS group.  Instead, the leave happens in three steps:

1. The leaving client constructs MLS Remove proposals for all of the user's
   devices (including the leaving client), and an AppSync proposal that removes
   its user from the participant list.
2. The leaving client sends this proposals to the hub.  The hub caches the proposals.
3. The next time a client attempts to commit, the hub requires the client to
   include the cached proposals.

The hub thus guarantees the leaving client that they will be removed as soon as
possible.

~~~ aasvg
ClientB1       ServerB         ServerA         ServerC         ClientC1
  |               |               |               |               |
  | Proposals     |               |               |               |
  +~~~~~~~~~~~~~~>| /update       |               |               |
  |               +-------------->|               |               |
  |               |        200 OK |               |               |
  |               |<--------------+               |               |
  |      Accepted |               |  /notify      |               |
  |<~~~~~~~~~~~~~~+               +-------------->|               |
  |               |               |               | Proposals     |
  |               |               |               +~~~~~~~~~~~~~~>|
  |               |               |               |               |
  |               |               |               | Commit(Props) |
  |               |               |               |<~~~~~~~~~~~~~~+
  |               |               |       /update |               |
  |               |               |<--------------+               |
  |               |               | 200 OK        |               |
  |               |               +-------------->|               |
  |               |               |               | Accepted      |
  |               |               |               +~~~~~~~~~~~~~~>|
  |               |       /notify | /notify       |               |
  |               |<--------------+-------------->|               |
  |               |               |               |               |

ClientB1: Prepare Remove*, AppSync(-bob@b.example)
ClientB1->ServerB: [[ Remove*, AppSync ]]
ServerB->ServerA: POST /update/clubhouse@a.example Remove*, AppSync
ServerA: Verify that Removes, AppSync are allowed by policy; cache
ServerA->ServerB: 200 OK
ServerA->ServerC: POST /notify/clubhouse@a.example Proposals
ServerC1->ClientC1: [[ Proposals ]]
ClientC1->ServerC: [[ Commit(Props), Welcome, GroupInfo?, RatchetTree? ]]
ServerC->ServerA: POST /update/clubhouse@a.example CommitBundle
ServerA: Check whether Commit includes queued proposals; accept
ServerA->ServerC: 200 OK
ServerA->ServerB: POST /notify/clubhouse@a.example Commit
ServerA->ServerC: POST /notify/clubhouse@a.example Commit
~~~
{: #fig-b-leave title="Bob Leaves the Room" }

# Framing

MIMI protocol messages are sent described using the TLS presentation language
format ({{Section 3 of RFC8446}}).

All MIMI protocol messages are framed by a MIMIMessage.

~~~ tls
enum {
   reserved(0),
   mimi10(1), // MIMI 1.0
   (65535)
} ProtocolVersion;

enum {
   reserverd(0),
   event(1),
   event_response(2),
} MIMIMessageType;

struct {
   // The protocol version this event is created for.
   ProtocolVersion version;

   // The room ID where the event is sent in context of.
   opaque roomId;

   // Who or what sent this event. For example, a user ID.
   opaque sender;

   MIMIMessageType message_type;
   select (MIMIMessage.message_type) {
      case event:
        Event event;
      case event_response:
        EventResponse response;
   }
} MIMIMessage
~~~

# Rooms and Events {#rooms-and-events}

Rooms, described by {{!I-D.barnes-mimi-arch}}, consist of a user participation
list, a cryptographic representation of the room as defined by
{{!I-D.robert-mimi-delivery-service}}, policy, and other metadata as needed.

> **TODO**: Consider renaming "event" to something else.

A room's state is modified, used, and retrieved through *events*. Some events
are fanned out to other participating servers while other events are operations
performed exclusively with the hub server for a room.

Events that change the state of the room are implemented through MLS proposals
as defined in {{!RFC9420}}, thus allowing the underlying MIMI DS protocol to
anchor the current room state cryptographically. MLS proposals are signed,
allowing every recipient in their path to verify their authenticity.

This document defines additional events to encapsulate MIMI DS protocol
messages, for example, to add clients to a room's underlying MLS group or to
request single-use key material for another user's clients.

Events carry information, and are not required to be persisted. The current
participation and policy state is confirmed by the cryptographic security layer
rather than being confirmed in events specifically.

## Event Schema {#event-schema}

Events are validated against their TLS presentation language format
({{Section 3 of RFC8446}}):

~~~
// See the "MIMI Event Types" IANA registry for values.
// Example: "m.room.info"
opaque EventType;

struct {
   // The event type.
   EventType type;

   // Additional fields may be present as dependent on event type.
   select (Event.type) {
      case "m.room.user":
         // MLSMessage containing a UserEvent proposal
         MLSMessage user_event_proposal;
      case "ds.proposal":
         DSRequest ds_proposal;
      case "ds.commit":
         DSRequest ds_commit;
      case "ds.fetch_key_packages":
         DSRequest fetch_key_packages;
      case "ds.fetch_group_info":
         DSRequest fetch_group_info;
      case "ds.send_message":
         DSRequest send_message;
      // more cases as required by registry
   }
} Event;
~~~

> **TODO**: Consider splitting `sender` into an object of `{type, identifier}`.

> **TODO**: The `sender` field might be a bit redundant now that signaling is
> largely handled through MLS proposals.

Note an "event ID" is not specified on the struct. Events are sent ephemerally
and confirmed by the underlying cryptographic group state rather than referenced
by a consistent identifier.

The "origin server" of an event is the server implied by the `sender` field.

Recipients of an event respond with a MIMIMessage of type event_response.

~~~ tls
enum {
  reserved(0),
  ok(1),
  key_package(2),
  group_info(3),
  error(4),
} EventResponseType

enum {
  // TODO
} EventErrorType

struct {
  EventErrorType type;

   select (EventResponse.type) {
      // TODO
   }
} EventError

struct {
   EventResponseType type;

   // Additional fields may be present as dependent on event type.
   select (EventResponse.type) {
      case ok:
         struct {};
      case key_package:
         DSResponse key_package;
      case group_info:
         DSResponse group_info;
      case error:
         EventError error;
   }
} EventResponse
~~~

## Room state

The state of a room consists of the room's RoomID, its policy, and the
participant list (including the role and participation state of each
participant). Also associated with the room is the MLS group managed by the MIMI
DS protocol, which anchors the room state cryptographically as part of the group
state.

While (through the MIMI DS protocol) all parties involved in a room agree on the
room's state, the Hub is the arbiter that decides if a state change is valid.
All state-changing events are sent to the Hub, checked for their validity and
policy conformance before they are forwarded to any follower servers.

As soon as the Hub accepts an event that changes the room state, its effect is
applied to the room state and future events are validated in the context of that
new state.

The room state is thus changed based on events, even if the MLS proposal
implementing the event was not yet committed by a client. Note that this only
applies to events changing the room state, but not for MIMI DS specific events
that change the group state. For more information on the proposal-commit
paradigm and the role of the MIMI DS protocol see {{mimi-ds}}.

## Cryptographic room representation {#mimi-ds}

Each room is represented cryptographically by an MLS group and the Hub that
manages the room uses the MIMI DS protocol specified in
{{!I-D.robert-mimi-delivery-service}} to manage that group.

In particular, the MIMI DS protocol manages the list of group members, i.e. the
list of clients belonging to users currently in the room.

### Proposal-commit paradigm

The MIMI DS protocol uses MLS, which follows a proposal-commit paradigm. Any
party involved in a room (follower server, Hub or clients) can send proposals
(e.g. to add/remove/update clients of a user or to re-initialize the group with
different parameters). However, only clients can send commits, which contain all
valid previously sent proposals and apply them to the MLS group state.

The MIMI DS protocol ensures that the Hub, all follower servers and the clients
of all participants (or at least those in the `join` state) agree on the group
state, which includes the client list and the key material used for message
encryption (although the latter is only available to clients). Since the group
state also includes a copy of the room state at the time of the most recent
commit, it is also covered by the agreement.

### Cryptographically anchoring room state {#anchoring}

To allow all parties involved to agree on the state of the room in addition to
the state of the associated group, the room state is anchored in the MLS group
via a GroupContext extension.

~~~ tls
struct {
   opaque user_id;
   opaque role;
   ParticipationState state;
} ParticipantData

struct {
  opaque room_id;
  ParticipantData participants<V>;
  // TODO: Add any remaining room data
} RoomState;
~~~

As part of the MIMI DS protocol, clients create commits to update the group
state, which are then included in MIMI DS specific events. The time between two
commits denotes an epoch.

Whenever a client creates a commit, it MUST include all valid proposals accepted
by the Hub during the current epoch. This includes both proposals that carry
room-state changes, as well as proposals sent as part of MIMI DS events.

Note that the validity of a proposal depend on the current room state, which may
change during an epoch based on room-state changing events. The changes of these
events are applied to the room state even if the commits that carry the event
information have not yet been committed.

### Authenticating proposals

The MLS specification {{!RFC9420}} requires that MLS proposals from the Hub and
from follower servers (external senders in MLS terminology) be authenticated
using key material contained in the `external_senders` extension of the MLS
group. Each MLS group associated with a MIMI room MUST therefore contain an
`external_senders` extension. That extension MUST contain at least the
Certificate of the Hub.

When a user from a follower server becomes a participant in the room, the
Certificate of the follower server MAY be added to the extension. When the last
participant belonging to a follower server leaves the room, the certificate of
that user MUST be removed from the list. Changes to the `external_senders`
extension only take effect when the MLS proposal containing the event is
committed by a MIMI DS commit. See {{ev-mroomuser}} for more information.

### MIMI DS events

The MIMI DS protocol operations are encapsulated in DSRequest structs and
contain a `request_type` field that details the operation in question. To
disambiguate MIMI DS operations on the event-level, each operation is assigned
its own distinct event type.

The MIMI DS protocol deals with authentication of each request and upon
successful processing returns a DSResponse to be sent to the sender of the
event, optionally an MLSMessage for full fan-out and optionally one or more
Welcome messages for fan-out to individual follower servers.

Depending on the event, a DSResponse either indicates successful processing, the
requested data (e.g. group information required for joins), or an error message.

Messages meant for fan-out are DSFanoutRequests, which contain an MLS message,
as well as information to which clients messages should be fanned out to.

> **TODO**: Update the MIMI DS doc to allow for messages to contain more than one
> proposal and a generic "commit" action.

#### Propose group update {#ev-proposeupdate}

**Event type**: `ds.proposal`

Group members, the Hub, and follower servers can use this event to propose
updates to the group. Each such event contains one or more proposals that can be
committed to update the state of the MLS group associated with the room. In
particular, this event can be used to add, remove or update clients in the
group.

~~~tls
struct {
  DSRequest proposal;
} DSProposal
~~~

**Additional validation rules**:

* Clients can only be added to the group if the associated user is on the
  participant list and in the `join` state.

#### Commit group update {#ev-commitupdate}

**Event type**: `ds.commit`

Group members can use this event to commit any pending proposals (including both
group updates and room updates). The sender of this event can include additional
group updates without proposing them separately through the `ds.proposal`
event.

Note that this event can also be used by a client to add itself to the group. To
do that, the sender requires the current group information (see
{{ev-fetchgroupinfo}}).

~~~tls
struct {
  DSRequest commitBundle;
} DSCommit
~~~

**Additional validation rules**:

* Clients can only be added to the group if the associated user is on the
  participant list and in the `join` state.

#### Fetch KeyPackage {#ev-fetchkeypackage}

**Event type**: `ds.fetch_key_packages`

> **TODO**: For now, we assume that KeyPackages are fetched directly, i.e. not in the
> context of a room and via a Hub. This might change in the future. If it does
> change, this event needs an additional authentication mechanism.

Group members can use this event to request a
KeyPackage from the Hub or another follower server.

~~~tls
struct {
  DSRequest fetch_key_packages;
} DSFetchKeyPackages
~~~

**Additional validation rules**:

None

#### Fetch group information {#ev-fetchgroupinfo}

**Event type**: `ds.fetch_group_info`

Group members or follower servers can use this event to request group
information from the Hub. Up-to-date group information is required for clients
to be able to add themselves to a group via the `ds.commit` event. The
group info returned to the sender includes any pending proposals.

~~~tls
struct {
  DSRequest fetch_group_info;
} DSFetchGroupInfo
~~~

**Additional validation rules**:

None

#### Send Message {#ev-sendmessage}

**Event type**: `ds.send_message`

> **TODO**: This is not a proposal and there is no way for the Hub or follower servers
> to authenticate this event at the moment. We might want to a way to do that
> later.

Group members can use this event to request to send an encrypted (application)
message to the other group members.

~~~tls
struct {
  DSRequest send_message;
} DSSendMessage
~~~

**Additional validation rules**:

None

## Creation {#room-creation}

Rooms (and the underlying MLS groups) are first created within the provider, out
of scope from MIMI. When the room is exposed to another server over the MIMI
protocol, such as with an explicit invite to another user, the creating server
MUST produce the following details:

* An `m.room.info` ({{ev-mroomcreate}}) event describing the encryption
  and policy details for the room.
* A universally unique room ID (represented by the room info event).
* An `m.room.user` ({{ev-mroomuser}}) event which invites the desired user.
* Any relevant cryptographic state needed to verify the invite is legitimate.
  For example, the ciphersuite used by the cryptographic security layer.

This is the minimum state required by a MIMI room. Room creators MAY wish to
include additional details in the initial state, such as configuration of the
room's policy, adding the creator's other clients to the MLS group state, etc.

### `m.room.info` {#ev-mroomcreate}

The `m.room.info` event contains the current room state, thus allowing servers
to bootstrap the room. Note that this event does not contain any of the
cryptographic state of the underlying MIMI DS group. Clients that want to join a
room need to use a `ds.fetch_group_info` event to obtain the information
required to join the room.

**Event type**: `m.room.info`

**Additional event fields**:

~~~
struct {
  RoomState room_state;
} InfoEvent;
~~~

> **TODO**: Include fields for policy information (previously called a "policy
> ID" in ralston-mimi-signaling).

**Fanout considerations**:

`InfoEvent` is *unsigned* in all cases it is used, but authenticated implicitly
through the transport layer ({{transport}}). The room info event is used during
invites to ensure the server is capable of participating in the room and is not
fanned out more generally. See {{op-check}} for usage.

# User Participation and Client Membership {#membership}

In a MIMI room, users are *participants* with an associated
*participation state* whereas clients of those users are *members* of the
cryptographic state. The user's participation state is updated concurrent to or
before changes are made to the cryptographic state.

Users will always exist in one of the following participation states:

~~~
enum {
   invite,  // "Invited" state.
   join,    // "Joined" state.
   ban,     // "Banned" state.
   knock,   // "Knocking" state.
} ParticipationState;
~~~

These states allow a user to remain logically "joined" to the conversation when
they have zero encryption-capable clients available. The user will not be able to see
messages sent while they had no clients, but can add their clients to the
cryptographic state at any time. A user with zero clients in the cryptographic
state is considered to be an *inactive participant*. Users with one or more clients
in the cryptographic state are *active participants*.

All servers with at least one user of theirs in the "joined" participation state
are considered to be "in" or "participating" in the room. Events which require
full fanout ({{fanout}}) are sent to all participating servers by default. Some
events MAY be sent to additional servers as needed by their fanout considerations.

The participant list is anchored in the cryptographic state of the room as
described in {{anchoring}}.

## Adds {#adds}

> **TODO**: We will probably want some kind of mechanism here that allows the
> adder to signal that they are authorized (by the added user) to add the added
> user to the room.

An *add* is when a user adds another user to the list of participants in the
*join* state. The `m.room.user` event that effects this change is typically sent
as part of a commit that also adds the user's clients to the room's MLS group.

1. The adder generates an `m.room.user` ({{ev-mroomuser}}) event to add the
   target user.

2. The adder sends ({{op-send}}) the `m.room.user` event to the hub server. If
   the adder is a client, the event is likely sent as part of a `ds.commit`
   event.

3. The hub server validates the event to ensure the following:

   * The target user of the add MUST NOT already be in the banned or joined
     states.

   * The sender of the invite MUST already be in the joined state.

4. If the event is invalid, it is rejected. Otherwise, it is forwarded by the
   hub to the servers of all participants in the joined state. This includes the
   server of the user added by the event.

5. The target user (or its server) can reject the addition by sending an
   `m.room.user` event that proposes the removal of the user and its clients
   ({{leaves}}).

## Invites {#invites}

An *invite* is when a user (or more specifically, a user's client) adds another
user to the list of participants in the `invite` state.

Once the user is on the participant list (and has been notified of this fact by
the Hub), one of the user's clients can add itself, as well as any other clients
to the room's underlying group.

Updating the target user's participation state is done using the following
steps, and is visualized in {{fig-invites}}.

1. The inviter's server generates an `m.room.user` ({{ev-mroomuser}})
   event to invite the target user. Typically this begins with a
   client-initiated request to the server using the provider-specific API.

2. The inviter's server sends ({{op-send}}) the `m.room.user` event to the hub
   server.

3. The hub server validates the event to ensure the following:

   * The target user of the invite MUST NOT already be in the banned or joined
     states.

   * The sender of the invite MUST already be in the joined state.

4. If the event is invalid, it is rejected. Otherwise, it is forwarded by the
   hub to the target user's server to give it the opportunity to reject the
   invite early in the process. This is described by {{op-check}}.

5. If the target server rejected the event, the event is rejected by the hub as
   well. Otherwise, the event is fanned out ({{fanout}}) to all participating
   servers, plus the target server if not already participating.

At this stage, the *user* is now invited but their clients are not members of
the cryptographic state. The invite is delivered to the target's clients through
relevant provider-specific API where the user can then accept or decline the invite.

If the user declines the invite, they are removed from the participant list.
Accepting is done by joining ({{joins}}) the room.

~~~ aasvg
+---+                            +-----+                         +---+
| A |                            | Hub |                         | B |
+---+                            +-----+                         +---+
  |                                 |                              |
  | Create m.room.user invite       |                              |
  |-------------------------+       |                              |
  |                         |       |                              |
  |<------------------------+       |                              |
  |                                 |                              |
  | Send event request initiated    |                              |
  |-------------------------------->|                              |
  |                                 |                              |
  |                                 | Validate m.room.user event   |
  |                                 |--------------------------+   |
  |                                 |                          |   |
  |                                 |<-------------------------+   |
  |                                 |                              |
  |    200 OK to send event request |                              |
  |<--------------------------------|                              |
  |                                 |                              |
  |                                 | Check event request          |
  |                                 |----------------------------->|
  |                                 |                              |
  |                                 |                       200 OK |
  |                                 |<-----------------------------|
  |                                 |                              |
  |           Async fanout of event | Async fanout of event        |
  |<--------------------------------|----------------------------->|
  |                                 |                              |
~~~
{: #fig-invites title="Invite happy path" }

## Joins {#joins}

Users join a room either in response to an invite (therefore accepting it) or
after discovering it as a public room. In both cases, the user first updates
their participation state before the cryptographic security layer is engaged to
add their clients. Note that both of these actions can be performed either
sequentially, or through a single `ds.commit` event.

> **TODO**: Describe policy considerations for what makes a room "public".

> **TODO**: Move the following paragraph to the MIMI DS subsection describing
> `ds.commit`.

A user already in the join participation state MAY add and remove their own
clients from the cryptographic state at will. Clients are unable to remove
themselves via `ds.commit`, however they are able to propose that they be
removed in the next commit via `ds.proposal`.

The joining user can follow one of two flows. Either it first updates the
participation state and then adds their clients, or it perfoms both actions in
the same event.

The two-step flow looks as follows:

1. Option a: The joiner's server generates an `m.room.user` ({{ev-mroomuser}})
   event to add the user.

   Option b: The joiner's client generates a commit that contains an
   `m.room.user` event, as well as an Add proposal for itself (this requires
   that the client has previously obtained a the room's group info through a
   `ds.fetch_group_info` event ({{ev-fetchgroupinfo}})). The joiner's server
   generates a `ds.commit` event from the commit.

2. The joiners's server sends ({{op-send}}) the generated event to the hub
   server.

3. The hub server validates the event to ensure the following:

   * The joining user MUST NOT already be in the banned from the room.

   * The sender and joining user MUST be the same.

4. If the event is invalid, it is rejected. Otherwise, the event is fanned out
   ({{fanout}}) to all participating servers, plus the joiner's server as they
   are now participating in the room too.

If the user was added to the room via a standalone `m.room.user` event, the
user's clients are able to add themselves to the cryptographic group state via
one or more `ds.commit` events after fetching the room's current information via
a `ds.fetch_group_info` event.

## Leaves/Kicks {#leaves}

Leaving a room can signal a user declining an invite, voluntarily leaving the
room, or being kicked (removed) from the room. When the sender and target of
an `m.room.user` ({{ev-mroomuser}}) leave event are different, the target user
is being kicked. Otherwise the event represents a voluntary leave or declined
invite (if the previous participation state was "invited").

Like with other participation/membership operations, a user's leave is initiated
by updating their participation state first. This is done by sending
({{op-send}}) the relevant `m.room.user` ({{ev-mroomuser}}) state event to the
hub, which validates it as follows:

* If the target and sender are the same, the user MUST be in the invited,
  joined, or knocking participation state.

* Otherwise:

  * The target user of the kick MUST be in the joined participation state.
  * The sender for a kick MUST be in the joined participation state.

> **TODO**: Include special case permissions constraints.

If the event is valid, it is fanned out ({{fanout}}) to all particpating
servers, plus the target user's server.

The next `ds.commit` event MUST remove *all* of the target user's clients. If
multiple users leave the room, all of their clients MUST be removed in the same
operation. Other cryptographically-relevant changes MAY be committed alongside
the removals, however the operation MUST at a minimum remove the affected
clients.

The hub server MAY be permitted to generate the needed changes to remove the
affected clients, requiring that those changes be confirmed/accepted by a client
remaining in the group promptly.

As mentioned in {{joins}}, a user already in the join participation state MAY
add and remove their own clients from the cryptographic state at will.

## Bans {#bans}

Bans imply kick, and are operated the same way as {{leaves}}, though with the
`m.room.user` ({{ev-mroomuser}}) state event using a `ban` participation state.

In contrast to leaving users, banned users remain on the participant list in the
`ban` state.

An added exception on the validation is also applied to permit preemptive bans:
the target user is not required to be in the joined state to allow the
participation state change.

Unbans can be performed by removing a user in the banned participation
state from the participant list {{leaves}}.

## Knocks {#knocks}

In this state, the sender of a knock is requesting an invite ({{invites}}) to
the room. They do not have access to the cryptographic state.

> **TODO**: Discuss if this participation state is desirable, and figure out
> details for how it works. It'd likely just be an `m.room.user` state event
> with no MLS interaction, like invites are.

> **TODO**: If we have an Add event as discussed in a TODO in the "Invites"
> section, an "Add" would probably be the response to a knock.

## `m.room.user` {#ev-mroomuser}

**Event type**: `m.room.user`

An `m.room.user` event can be used to change the participation state of a user.

> **TODO**: Do we also want this to be able to change a participant's role?

It is transported via an MLS proposal of type UserEvent. If the event adds a
user to the room and it is the first user in the room that belongs to the
sending follower server, the UserEvent MAY contain the Certificate that can be
used to validate external proposals from that follower server. If it does, the
commit that contains the proposal adds the Certificate to `external_senders`
extension of the underlying MLS group.

If the event removes the last user of a follower server from a room, the commit
that contains the MLS proposal that carries the event removes the Certificate of
that follower server from the extension.

> **TODO**: This proposal needs to be added to the IANA proposal list, or
> specified as an extension proposal as specified in the MLS extensions
> document. We might want to have one MIMIProposal type that in turn can
> encapsulate more than just this event.

~~~tls
enum {
   invite,
   join,
   leave,
   ban,
   knock,
} ParticipationStateChange;

struct {
   // The user ID being affected by this participation state change.
   opaque targetUserId;

   // The new participation state for the target user. "Leave" removes
   // the user from the list.
   ParticipationStateChange state;

   // Optional human-readable reason for the change. Typically most
   // useful on bans and knocks.
   opaque [[reason]];
   optional<Certificate> follower_server_certificate;
} UserEvent;
~~~

**Additional validation rules**:

* Rules described by {{invites}}, {{joins}}, {{leaves}}, {{bans}}, {{knocks}}.
* The proposal MUST be authenticated as an MLS message based on the room's
  underlying MLS group.

> **TODO**: Include validation rules for permissions.

**Fanout considerations**:

Each `m.room.user` event is fanned out as normal ({{fanout}}). The event MAY be
sent to additional servers, as required by {{invites}}, {{joins}}, {{leaves}},
{{bans}}, {{knocks}}.

**Additional validation rules**:

None.

**Fanout considerations**:

This event is not fanned out.

# Transport {#transport}

Servers communicate with each other over HTTP {{!RFC9110}} by "sending" events
({{event-schema}}) to each other. Responses are also events for ease of handling.

## Authentication

All endpoints, with the exception of `.well-known` endpoints, use the mutually
authenticated mode of TLS {{!RFC5246}}. This provides guarantees that each
server is speaking to an expected party.

> **TODO**: More information specific to how TLS should be used, i.e. mandate
> best practices that make sense in a mutually authenticated scenario that
> involves two WebPKI based certificates.

Individual events MAY transit between multiple servers. TLS provides
point-to-point security properties while an event's `signature` provides
authenticity over multiple hops.

## Endpoint Discovery

A messaging provider that wants to query the endpoint of another messaging
provider first has to discover the fully qualified domain name it can use to
communicate with that provider. It does so by performing a GET request to
`https://example.org/.well-known/mimi/domain`. example.org could, for example,
answer by providing the domain `mimi.example.org` (assuming that this is where
it responds to the REST endpoints defined in {{rest-api}}).

The expected response format is simply a `text/plain` body containing the fully
qualified domain name.

~~~
GET https://example.org/.well-known/mimi/domain

Response
mimi.example.org
~~~

## REST Endpoints {#rest-api}

The following REST endpoints can be used to communicate with a MIMI server.

All operations rely on TLS-encoded structs and therefore requests and responses
SHOULD use a `Content-Type` of `application/octet-stream`.

### Send Event {#op-send}

Asks the server to send an event ({{event-schema}}). Each event is subject to
additional validation and handling within this endpoint, such as ensuring the
room's policy is not violated.

Follower servers in a room MUST only send to the hub server. The hub server is
responsible for further fanout ({{fanout}}) if required by the event, after the
send request has been completed.

Follower servers receiving an event from another
follower server MUST reject the request with a `400` HTTP status code. The hub
server MUST validate the event according to the event's rules, then perform any
additional actions on the event as required by the event. For example, the hub
server may check that an invite is legal under the room's policy, then ensure
the target server accepts the event with {{op-check}}, then finally continue
processing.

Rejected send requests MUST return a `400` HTTP status code. Accepted send
requests MUST return a `200` HTTP status code, and an event in the response body
if one is applicable.

If the event requires fanout ({{fanout}}), the event is then fanned out
{{fanout}} to relevant servers in the room.

Follower servers SHOULD apply the same validation as hub servers upon receiving
a send request to identify potentially malicious hub servers.

~~~
POST /send
Content-Type: application/octet-stream

Body
TLS-serialized Event

Response
TLS-serialized Event, or empty if no useful event.
~~~

Servers SHOULD retry this request with exponential backoff (to a limit) if they
receive timeout/network errors.

#### Fanout {#fanout}

A hub server fans an event out by using the send endpoint described above on all
participating servers in the room. A server is considered "participating" if it
has at least one user in the joined participation state, described by
{{membership}}.

Additional servers MAY have the event sent to them if required by the steps
leading up to fanout.

### Check Invite Event {#op-check}

> **TODO**: Consider reducing this down to `m.room.check_invite` or something,
> to reuse `/send`.

Used by the hub server to ensure a follower server can (and is willing to)
process an incoming invite. The called server MAY use this opportunity to ensure
the inviting user has general consent to invite the target user. For example,
ensuring the invite does not appear spammy in nature and if the inviter already
has a connection with the invitee.

If the server does not recognize the event format of the `InfoEvent`
({{ev-mroomcreate}}) event, or does not understand the policy/encryption
configuration contained within, it MUST reject the request.

The request MAY be rejected with a `400` HTTP status code. If everything looks
OK to the server, it responds with a `200` HTTP status code.

~~~
struct {
   // The `m.room.user` invite event.
   Event invite;

   // The room creation information.
   InfoEvent roomInfo;
} CheckInviteRequest;
~~~

> **TODO**: If we plan to keep this as an independent request, it will need a
> protocol version field.

~~~
POST /check-invite
Content-Type: application/octet-stream

Body
TLS-serialized CheckInviteRequest

Response
Any meaningful information. The pass/fail is identified by the HTTP response
status code, not the response body.
~~~

The hub server SHOULD consider a network error as a rejection. It is expected
that the original sender will attempt to re-send the invite once the server is
reachable again.

# Security Considerations

Overall, the user participation state leads any possible MLS group state to
ensure malicious clients are not able to easily get access to messages.

> **TODO**: Other security guarantees? Consensus may be required here.

# IANA Considerations

IANA has created the following registries:

* MIMI Event Types

## MIMI Event Types

An event type denotes the nature of a payload contained in an event, in the
context of the MIMI protocol. The event type is a string composed of substrings
separated by dots.

The first substring is "m", followed by the logical container being affected
(typically just "room"), then a number of descriptor strings.

Example: `m.room.info`

> **TODO**: Does IANA need any other information for legal event types?

--- back

# Acknowledgments
{:numbered="false"}

> **TODO**: Refactor acknowledgements to match sections of interest.

This document is the consolidation of the following documents:

* {{?I-D.kohbrok-mimi-transport}} forms the majority of {{transport}}.

* {{?I-D.robert-mimi-delivery-service}} describes details for {{membership}},
  subsections of {{rest-api}} (per transport draft).

* {{?I-D.ralston-mimi-signaling}} describes {{event-schema}},
  {{room-creation}}, details of {{membership}}, and subsections of {{rest-api}}.

Aspects of {{?I-D.ralston-mimi-policy}} are additionally taken into
consideration in this document through subsections of {{membership}}, but is
largely unincorporated and may require updates to match this document's
specifics.

{{!I-D.barnes-mimi-arch}} was additionally used throughout the writing
of this document.
