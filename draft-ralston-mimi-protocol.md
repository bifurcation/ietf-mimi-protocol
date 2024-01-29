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

The More Instant Messaging Interoperability (MIMI) protocol enables providers of
end-to-end encrypted instant messaging to interoperate. As described in the MIMI
architecture {{?I-D.barnes-mimi-arch}}, group chats and direct messages are
described in terms of "rooms".  Each MIMI protocol room is hosted at a single
provider (the "hub" provider"), but allows users from different providers to
become participants in the room.  The hub provider maintains the policy and
participation for the room, authorizes messages, and is responsible for message
ordering, the participation list, and policy of its rooms. Each provider also
stores initial keying material and consent for its users (who may be offline).

This document describes the communication among different providers necessary to
support messaging application functionality, for example:

* Sharing room policy
* Adding and removing participants in a room
* Exchanging secure messages

In support of these functions, the protocol also has primitives to fetch initial
keying material, fetch the current state of the underlying end-to-end encryption
protocol for the room, and request, grant, and reject consent to communicate
with users on other providers.

Messages sent inside each room are end-to-end encrypted using the Messaging
Layer Security (MLS) protocol {{!RFC9420}}, and each room is associated with an
MLS group. MLS also ensures that clients in a room agree on the room policy and
participation.  MLS is integrated into MIMI in such a way as to ensure that a
client is joined to a room's MLS group only if the client's user is a
participant in the group, and that all clients in the group agree on the state
of the room (including, for example, the room's participant list).

## Known Gaps

In this version of the document, we have tried to capture enough concrete
functionality to enable basic application functionality, while defining enough
of a protocol framework to indicate how other necessary functionality.  The
following functions are likely to be needed by the complete protocol, but are
not covered here:

Authorization policy:
: In this document, we assume that all participants in a room have equal
capability.  Actual messaging systems have authorization policies for which
clients can take which actions in a room.

Advanced join/leave flows:
: In this document, all adds / removes / joins / leaves are initiated from
within the group, since this aligns well with MLS.  Messaging application
support a variety of other flows, some of which this protocol will need to
support.

Consent:
: In this document, we assume that any required consent has already been
obtained, e.g., a user consenting to be added to a room by another user.  The
full protocol will need some mechanisms for establishing this consent.  Some
initial notes are in {{consent}}

Identifiers:
: Certain entities in the MIMI system need to be identified in the protocol.  In
{{identifiers}}, we define a notional syntax for identifiers, but a more
concrete one should be defined.

Abuse reporting:
: There is no mechanism in this document for reporting abusive behavior to a
messaging provider.

Identifier resolution:
: In some cases, the identifier used to initiate communications with a user
might be different from the identifier that should be used internally.  For
example, a user-visible handle might need to be mapped to a durable internal
identifier.  This document provides no mechanism for such resolution.


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


# Example protocol flow

This section shows how three users (Alice, Bob and Cathy) from different
providers can use the protocol specified in this document to join the same room
and send messages to one-another. (Not all of these steps are part of the MIMI
transport protocol):

- Alice get the internal identifier for Bob
- Alice gains consent to talk to Bob
- Alice fetches initial keying material for Bob's clients
- (Alice create a room) †
- Alice adds Bob to a room
- Alice sends a message to the room
- Bob sends a message to the room
- The hub/owning provider fans out the message
- Bob adds a new client
- Alice repeats the first three steps for Cathy
- Alice adds Cathy to the room
- Alice removes Bob from the room

Throughout this section, we will detail the individual operations through
Alice's lens. All actions by Alice are taken through Alice's client and her
server's proprietary API. While some parts of the individual messages originate
from Alice's clients, the MIMI protocol is only concerned with the communication
between Alice's, Bob's and Cathy's servers. So whenever users are referenced
throughout this section, it is implied that they are communicating with their
respective servers, which then communicate with one-another.

† as mentioned in the text, creating a room is a local provider
action, which is out of scope of MIMI.

## Get internal identifier for Bob

This is an example/placeholder for a possibly more sophisticated discovery
mechanism. It is not intended to be directly implemented.

> **TODO**: Replace with or reference a discovery mechanism with WG consensus

> **TODO**: Details will follow once the WG has concensus on identifiers, their
> mapping and their retrieval/discovery. The format of specific identifiers is
> discussed in {{?I-D.mahy-mimi-identity}}. Any specific conventions which are
> needed should be merged into this document.

Alice obtains Bob's internal identifier.

Alice fetches the internal identifier for some field of Bob's, in
this example his handle.

~~~
GET /identifierDiscovery/{domain}
~~~

The request body is described as:

~~~ tls
enum {
  reserved(0),
  handle(1),
  nick(2),
  email(3),
  phone(4),
  partialName(5),
  wholeProfile(6),
  oidcStdClaim(7),
  vcardField(8),
  (255)
} IdentifierType;

struct {
  IdentifierType type;
  string searchValue;
  select(type) {
     case oidcStdClaim:
       string claimName;
    case vcardField:
       string fieldName;
  };
} IdentifierRequest;

~~~

The response body is described as:

~~~ tls
enum {
  success(0),
  notFound(1),
  ambiguous(2),
  forbidden(3),
  unsupportedField(4),
  (255)
} IdentifierDiscoveryCode;

struct {
  IdentifierDiscoverCode responseCode;
  IdentifierUri uri;
} IdentifierResponse;
~~~

> **ISSUE**: In the course of discovering Bob, Alice might or might not
> obtain a list of Bob's clients.

## Fetch initial key material for Bob

The action attempts to claim initial keying material for all the clients
of a list of users at a single provider. The keying material may not be
reused unless identified as "last resort" keying material.

Alice sends an Event (framed in a MIMIMessage) of type `ds.fetch_key_packages`
to Bob's server, requesting key material for each of Bob's clients.

See {{event-schema}} for the definition of Events in general and
{{ev-fetchkeypackage}} for the `ds.fetch_key_packages` event.

> **TODO**: For now, the event fetches key packages based on client identifiers,
> not user identifiers. This is because KeyPackage fetching is part of MIMI DS,
> which doesn't know what a user is. We could, however, send multiple MIMI DS
> requests in a single MIMI protocol request.

If the request was successful, the response contains a list of KeyPackages, one
for each client.

> **TODO**: For now, responses are simple, we can improve granularity when we
> start optimizing.

> **TODO**: Once we have framing for MIMIResponses, add a reference here.

## Create a room

Creating a room is done between a client and its local provider and is out of
scope of MIMI. Since Alice creates the room on the server of her local provider,
it becomes the Hub for the room. However, the room policy format is in scope for MIMI.
We assume that this room is a members-only room, and that only users who have
the admin or owner role can add and remove users.

We also assume that Alice has both the `owner` and `admin` roles within the room.

> **TODO**: Add information on room policy here once we have consensus on what
> that looks like.

## Alice adds Bob

Management of the participants list (i.e. the list of users in the room) is done
through `m.room.user` events. To add Bob to the room, Alice creates an
`m.room.user` event. See {{ev-mroomuser}} for more information on user state
changes.

Room state is anchored in the room's underlying MLS group through a GroupContext
Extension, which mirrors all of the room's state variables.

`m.room.user` events are MLS proposals, which change the room state outside of
the MLS group immediately upon reception and align the mirror of the room state
in the extension (which is part of the group state) upon the next `ds.commit`
event. This reflects the MLS proposal-commit paradigm that allows members and
external parties to propose changes, but only members to commit them. See
{{anchoring}} for more information on how the room state is cryptographically
anchored.

Alice does not only want to add Bob as a participant, but also Bob's client. To
do that, she sends a `ds.commit` event ({{ev-commitupdate}}) to the group. As
`ds.commit` events can contain an arbitrary number of proposals, Alice includes
the `m.room.user` proposal, as well as a proposal to add Bob's client. She
creates the latter by using the key material she previously fetched from Bob's
server.

She then sends the resulting `ds.commit` to the Hub. Upon reception, the Hub
processes the commit and the proposals therein by updating the participant list
to include Bob and the underlying MLS group state to add Bob's client.

Finally, the Hub fans out the commit event to Bob.

> **TODO**: Especially when adding new clients, we will want to forward a
> `ds.commit` event selectively, as existing group members will only be
> interested in the actual commit, whereas new group members only need the
> Welcome.

## Alice sends a message to the room

After Bob has joined the room, Alice creates a message using the MIMI content
format {{?I-D.ietf-mimi-content}}, and sends a `ds.send_message`
({{ev-sendmessage}}) event to the Hub.

## The hub/owning provider fans out the message

If the Hub accepts the message it fans the message out to all guest servers,
which in this case is Bob's server.

Generally, the Hub also fans out any messages which originate from itself (ex:
removing clients of deleted users).

## Bob sends a message to the room

This is no different from Alice sending a message, which is fanned out in
exactly the same way.

## Bob adds a new client

For Bob's new client to join the MLS group and therefore fully participate in
the room with Alice, Bob needs to fetch the current group and room information
through a `ds.fetch_group_info` event ({{ev-fetchgroupinfo}}), which he sends to
the Hub.

> **TODO**: in the MLS case, what security properties are needed to protect a
> GroupInfo object in the MIMI context are still under discussion. It is
> possible that the requester only needs to prove possession of their private
> key. The GroupInfo in another context might be sufficiently sensitive that it
> should be encrypted from the end client to the hub provider (unreadable by the
> local provider).

The new client can use the information to add itself to the group via a
`ds.commit` that contains an MLS ExternalInit proposal.

## Alice adds Cathy

Alice gets the identifier for Cathy and waits for her to join the room in
exactly the same way as in previous steps.

## Alice removes Bob from the room

Alice removes Bob by sending an `ds.commit` event. The `ds.commit` event
contains an `m.room.user` event which removes Bob from the participant list, as
well as proposals to remove Bob's clients.

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
