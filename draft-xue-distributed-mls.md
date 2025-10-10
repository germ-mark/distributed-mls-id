---
title: "Distributed MLS"
abbrev: "DMLS"
category: info

docname: draft-xue-distributed-mls-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Messaging Layer Security"
keyword:
 - messaging layer security
 - end-to-end encryption
 - post-compromise security
venue:
  group: "Messaging Layer Security"
  type: "Working Group"
  mail: "mls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mls/"
  github: "germ-mark/distributed-mls-id"
  latest: "https://germ-mark.github.io/distributed-mls-id/draft-xue-distributed-mls.html"

author:
 -
    fullname: "Mark Xue"
    organization: Germ Network, Inc.
    email: "mark@germ.network"
 -
    fullname: "Joseph W. Lukefahr"
    organization: US Naval Postgraduate School
    email: "joseph.lukefahr@nps.edu"
 -
    fullname: "Britta Hale"
    organization: US Naval Postgraduate School
    email: "britta.hale@nps.edu"

normative:

informative:

--- abstract

The Messaging Layer Security (MLS) protocol enables a group of participants to
negotiate a common cryptographic state for messaging, providing Forward
Secrecy (FS) and Post-Compromise Security (PCS). There are some use cases
where message ordering challenges may make it difficult for a group of
participants to agree on a common state or use cases where reaching eventual
consistency is impractical for the application. This document describes
Distributed-MLS (DMLS), a configuration for using MLS sessions to protect messages
among participants without negotiating a common group state.

--- middle

# Introduction

Participants operating in peer-to-peer or partitioned network topologies
may find it impractical to access a centralized Delivery Service (DS), or reach
consensus on message sequencing to arrive at a consistent commit for each
MLS epoch.

DMLS is a configuration of MLS for facilitating group messaging in such use
cases by instantiating an MLS group per participant, such that each participant
has a dedicated 'send' group within a communication superset of such groups.
This allows each participant to locally and independently control the sequence
of update processing and encrypt messages using MLS accordingingly. This draft
further addresses how to incorporate randomness from other participant's 'send'
groups to ensure post-compromise security (PCS) is maintained.

## Terminology

Send Group: An MLS group where one designated member (the group 'owner') authors
all messages and other members use the group only to receive from the designated sender.

Universe: A superset of MLS participants comprised of the owners of all Send
Groups.

## Protocol Overview

Within a universe U of distributed participants, we can resolve state conflict by
assigning each member local state that only they control. In DMLS, we assign
each member an MLS group to operate as a Send Group. The Send Group owner can export
secrets from other groups owned by the Universe and import the epoch randomness
through use of Proposal messages into their own Send Group. This enables each Send Group
to include entropy from other receive-only members of their Send Group, providing for
both PCS and FS without the need to reach global consensus on ordering of updates.

## Meeting MLS Delivery Service Requirements

The MLS Architecture Guide specifies two requirements for an abstract Delivery
Service related to message ordering.
First, Proposal messages should all arrive before the Commit that references them.
Second, members of an MLS group must agree on a single MLS Commit message that
ends each epoch and begins the next one.

An honest centralized DS, in the form of a message queuing server or content
distribution network, can guarantee these requirements to be met.
By controlling the order of messages delivered to MLS participants, for example,
it can guarantee that Commit messages always follow their associated Proposal messages.
By filtering Commit messages based on some pre-determined criteria, it can ensure
that only a single Commit message per epoch is delivered to participants.

A decentralized DS, on the other hand, can take the form of a message queuing server
without specialized logic for handling MLS messages, a mesh network, or, prehaps, simply
a local area network. These DS instantiations cannot offer any such guarantees.

The MLS Architecture Guide highlights the risk of two MLS members generating different
Commits in the same epoch and then sending them at the same time. The impact of this risk is
inconsistency of MLS group state among members. This perhaps leads to inability of some
authorized members to read other authorized members' messages, i.e., a loss of availability
of the message-passing service provided by MLS. A decentralized DS offers no mitigation
strategy for this risk, so the members themselves must agree on strategies, or in our
terminology, operating constraints. We could say that the full weight of the CAP theorem
is thus levied directly on the MLS members in this case. However, use cases exist that
benefit from, or even necessitate, MLS and its accompanying security guarantees for
group message passing.

The DMLS operating constraints specified above allow honest members to form a distributed
system that satisfies these requirements despite a decentralized DS.

# Send Group Operation

An MLS Send Group operates in the following constrained way:
  * The creator of the group, occupying leaf index 0, is the designated owner of the Send Group
  * Other members only accept messages from the owner(creator)

## Send Group Mutation

Under this configuration, only the send group creator can mutate the group.
The creator can commit to their group to broadcast new keys and/or to incorporate
new keys from other members of the universe.

### (DMLS Update) Broadcast new keys for creator
Alice can provide PCS for herself in her send group by authoring a (full or empty)
commit that updates her own leaf node.

### (DMLS Commit) Incorporate new key material from others
If Alice has received DMLS updates from other members, Alice can incorporate them as
follows:

If the latest DMLS Update Alice received from Bob in his send group is a commit
starting epoch k, and was not already incorporated into Alice's send group,
Alice can author a commit that
*  replaces Bob's leaf node in Alice's send group with Bob's new leaf note in commit k
*  imports a PSK from Bob's send group, epoch k with the following parameters
   *  psk_id: k || (bob's send group id)
      where k is a fixed width 8-byte encoding of the epoch in network byte order
   *  psk: MLS-Exporter("exporter-psk", "psk_id", KDF.Nh)

An MLS commit can convey either a DMLS Update or Commit, or both.

# Universe Mechanics

A DMLS implementation constructs a DMLS context U by defining
*  the Universe of members
*  a random universe identifier
*  a scheme for assigning a send group identifier for each member
*  allowed cipher suites, and an export key length.
and distributing initial keypackages for each members

## Send Group Creation
Within U, members create their send group by constructing a MLS group
*  with the assigned send group identifier
*  adding all other members
*  distributing the resulting welcome message

## Group Operations

Members of U encrypt and broadcast application messages in their send group.
Members provide PCS against themselves by authoring and distributing DMLS updates.
When they receive DMLS Updates from other group members,
they can incorporate the new PCS key material with a DMLS commit.

DMLS updates are ordered by the committer's epoch. Members may skip DMLS updates
if they have received multiple, but must commit the latest pending DMLS update for each user
before sending application messages of their own.

# Properties

Under DMLS, members can successfully encrypt messages at any time without waiting for
in-flight handshake messages from other members. A DMLS commit by Alice acknowledges
to everyone else the newest DMLS update Alice has received from each member.
Alice can delete her kth leaf node private key when all members have committed
a newer leafNode from her.

Applications may handle offline members by dropping offline members. If Bob
has been offline and not acknowleged Alice's kth update, Alice may choose
to delete her kth key anyway, foreclosing the possiblity of receiving future
messages to Bob. Alice can signal this in her next DMLS update or commit by
removing Bob from her send group. This allows each member of the universe to
independently excise offline members, and signal to everyone (including the removed member)
that they are doing so.

Reintroducing them is outside the scope of this draft, and likely involves creating a new
Universe of participants.


# Wire Formats

DMLS uses standard wire formats as defined in {{!RFC9420}}.  An application using DMLS should define formats for any additional messages containing common configuration or operational parameters.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

DMLS inherits and matches MLS in most security considerations with one notable change to PCS nuances. In
MLS each group member can largely control when their updates will be introduced to the group state, with
deconfliction only down to the DS. In contrast, in DMLS the Send Group owner controls when key update
material is included from each member; namely, every member updates in their own Send Group and fresh
keying material is then imported to other Send Groups through use of the exporter key and PSK Proposal
option, with timing controlled by the respective Send Group owners. This means that while the PCS
healing frequency of a given member in MLS is under their own control, in DMLS the PCS healing frequency
and timeliness of PSK import is controlled by the Send Group owner. However, the Send Group owner is also
the only member sending data in the Send Group. This means that there is a natural incentive to update
frequently and in a timely manner.


# IANA Considerations

This document has no IANA actions.

# References
 CAPBR: # Brewer, E., "Towards robust distributed systems (abstract)", ACM, Proceedings of the nineteenth annual ACM symposium on Principles of distributed computing, DOI 10.1145/343477.343502, July 2000, <https://doi.org/10.1145/343477.343502>.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
