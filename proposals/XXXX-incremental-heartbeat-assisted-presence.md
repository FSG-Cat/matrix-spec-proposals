# MSC0000: Incremental Heartbeat Assisted Presence

- [MSC0000: Incremental Heartbeat Assisted Presence](#msc0000-incremental-heartbeat-assisted-presence)
  - [Introduction](#introduction)
    - [Background](#background)
    - [High level overview of how to fix this.](#high-level-overview-of-how-to-fix-this)
  - [Proposal](#proposal)
    - [Heartbeat System](#heartbeat-system)
    - [Client Server API Level](#client-server-api-level)
    - [Incremental presence](#incremental-presence)
  - [Potential issues](#potential-issues)
  - [Alternatives](#alternatives)
  - [Security considerations](#security-considerations)
  - [Unstable prefix](#unstable-prefix)
  - [Dependencies](#dependencies)

## Introduction

This MSC is a redesign of the presence system to enable it to scale better. And make its failure
modes more safe. This MSC is inspired by the incremental updates and state tracking that EIGRP uses.

This MSC is a derivative of my earlier work on the topic of presence in the form of this [gist](https://gist.github.com/FSG-Cat/bd621bac3346497e2c336362d3260ddb).
And builds upon it with my knowledge from the 2 years that has passed since i wrote that document
on my iPad during downtime and time i was bored during class.

Due to this MSCs considerable length it includes not only a Table of Contents but an explanation of the sections.

### Background

[Presence](https://spec.matrix.org/v1.11/client-server-api/#presence) is only enabled by homeserver admins who have the resources to burn on it.
All homeserver implementations recognise presence as a very inefficient feature, and also a feature that does not scale.
In addition, the current presence system is fragile: there have been presence bugs in sliding sync proxy, and presence flooding
can be accidentally triggered by clients.

### High level overview of how to fix this.

The High level overview goes into a higher level overview of this MSC and goes into background.
The overview is provided for the purpose of making the MSC easier to write, and to make the proposal easier to understand.
As the high level overview explains the system without getting too concerned with the minute details found in the proposal section.

This MSC proposes fixing presence by taking a page out of the routing protocol world. The precursor to
this MSC in the form of the gist from earlier exists due to my studies having us learn the ins and outs of
the EIGRP and OSPF routing protocols. It turns out that these routing protocols have a similar problem to us.
They need to distribute Alive or Dead status to all the nodes so your routing tables are up to date.

And what is presence if not Alive or Dead status information. Now OSPF takes the problematic but works at limited
enough scale approach of sending a complete copy of state every time anything changes. Its a known flaw but it works.
Now EIGRP uses incremental updates and that's exactly what i propose we copy. This MSC therefore proposes
that we change presence to be a incremental system that only communicates state changes and move expiry into a
separate subsystem of presence.

To facilitate incremental presence we only need to keep track of whether a given party is alive or dead. For example,
if matrix.org is upgrading their database over a duration of six hours, we wouldn't want matrix.org's users to be
marked as online for the entire maintenance duration.
The proposal section goes into detail around how this alive tracking works but in the high level overview its
also important to include that servers can tell other servers that they are going offline/disabling presence.

So how does changes only and knowing alive or dead status for homeservers help? Well simple if one takes a look at
EDU charts. Unless your reporting as away for a 8-16 sleep cycle assuming your desktop follows that one should report as
online at the start of the 16 hours awake cycle and offline at start of the 8 hour sleep cycle. That's 2 EDUs for the whole day.

Reality? You see a LOT more than 2 EDUs for the whole day and we have non essential fields like last active that complicate matters.
If we could cut down presence volumes to these extremely low volumes to make presence volumes comparable to PDU volumes or less
than PDU volumes, i am convinced we could reach a point where matrix.org and the other servers that disabled presence for performance
concerns can flip it back on.

The last big change is to introduce a totem pole mechanism for presence where sates have ranks.
This means that for example if your laptop is online and reports presence as the `online` state.
And then you also open your desktop where you report as `offline` explicitly your desktop gets ignored.

This happens because under the current presence states and disabled from [MSC4042](https://github.com/matrix-org/matrix-spec-proposals/pull/4042)
the succession order is from top down the following. `online` > `unavailable` > `offline` > `disabled`.
Any client that wants presence to decrease in this totem pole has to instead use the override mechanism
from [MSC4043](https://github.com/matrix-org/matrix-spec-proposals/pull/4043) to force the state to
win. This mechanism exists to eradicate the possibility of fighting over the presence state present in the
legacy APIs for presence.


## Proposal

  - [Proposal](#proposal)
    - [Heartbeat System](#heartbeat-system)
    - [Client Server API Level](#client-server-api-level)
    - [Incremental presence](#incremental-presence)

There are three components to the proposal:
  - The [Heartbeat system](#heartbeat-system) to determine whether a server is still online and therefore if
    incremental presence information is still accurate.
  - [Incremental presence](#incremental-presence) making servers only track changes to presence state and
    move expiry to a separate subsystem that being the heartbeat system.
  - The [CS API](#client-server) changes required to implement incremental presence

### Heartbeat System

The heartbeat system is used to make sure that a remote server is vaguely alive. To achieve this we use
the metric of has a server pair executed a transaction in the last 15-30 minute window. You pick a random
time in that 15 minute window and if things are quiet when that timer expires you initiate the heartbeat
protocol.

The purpose of this being a random timespan is to lessen synchronisation load spikes that would otherwise
happen if you send a message in a room as large as matrix HQ assuming things went quiet after that.

If the heartbeat goes through and the remote server responds by acknowledging the heartbeat both servers
reset their heartbeat timers for each other and you rinse and repeat this process whenever you have not heard
from each other in enough time.

FIXME: The exact shape of the API used for heartbeats is for a Revision 2 of this MSC as its currently a
draft.

If heartbeat fails you use exponential backoff(Consider ME: Alternative backoff algorithms? )
and retry. After a period of 5 minutes of having retried and failed you drop all presence states for the remote server down to offline.

After the 5 minute retry period has expired and presence state has been dropped to offline revert to the same exponential backoff schedule
as is used for normal federation for a given server.

If a homeserver expects to be down for a significant time period like how matrix.org was down for
database upgrade some time ago a server can be courteous and send a drop update to all interested servers
announcing that its going to be going offline and to seize heartbeats to the server until the server sends
a presence update of any kind.

Servers can issue this drop command with 2 different instructions. Drop to `offline` or mark as `disabled`.
Dropping to offline is used if you expect to return later to online status but you ask for mark as `disabled`
to indicate that your server is turning off presence completely for the time being. These settings
are social and are not technical necessities but they are very useful. You get to stop the constant
screams of heartbeats asking are you there, and you get to tell others if you will return or not as far
as presence is concerned.

### Client Server API Level

Under this MSC presence adopts a succession order where only the state highest in the succession
order is used for your account wide presence state. Under the current presence states and disabled from
[MSC4042](https://github.com/matrix-org/matrix-spec-proposals/pull/4042) the order is `online` > `unavailable` > `offline` > `disabled`.

The `unavailable` state is either set by the client using a last interaction timestamp or assumed by the server
after a sufficient period of inactivity assuming that the user has this state enabled.

The `last_active_ago` field of presence is removed on the S2S API level and replaced with `last_active_timestamp`.
To facilitate clients being able to say how long ago a given presence state transition happened the server will report
to the client the timestamp of the last transition and the transition that occurred. Its up to the client
to determine if the information is deemed useful.

A client may still choose to set presence to `unavailable` manually and if it chooses to do so may provide a
timestamp for `last_active_timestamp`. If not provided the server will assume a timestamp based on the same
factors that it uses to automatically set the `unavailable` presence status and `last_active_timestamp`.
### Incremental presence

Upon initial contact with a homeserver or after being asked for a full table you send a full copy of your
presence table that the remote homeserver in question is interested in. This presence table is considered
valid under the rules in the heartbeat system defined earlier in this MSC or until a change is made to
a entry. When a user changes their presence state that change is updated on the server side and put in a bucket.
Once this bucket is old enough or full enough you send that whole batch of presence updates to all remote servers
that are interested. The size of this bucket and maximum delay is a implementation detail as long as the resulting
transaction conforms to S2S API rules.

Presence states in this system are valid until a change is communicated and servers are to only send changes
when not requesting a full copy due to having dropped its copy. The most common cause for dropping your copy is heartbeat
expiry.

FIXME: Consider if a recommendation to drop table on restart is a good idea or not?

Status message changes are also considered changes ofc as status messages are part of the presence EDUs.

The `last_active_ago` field in the presence EDU is removed and replaced by a `last_active_timestamp` field.
This `last_active_timestamp` is a standard unix milliseconds timestamp of when the user was last active if their current
presence status is `unavailable`. If their status is any other status this field is to be ignored if present as its a bug.

The `currently_active` field is also removed as its purpose of suppressing the `last_active_ago` field is removed since
that field is also removed. In addition since the last active ago behaviour is only reported once the users state is flipped
to away this is no longer in need of being suppressed as it wont cause presence spam.

## Potential issues

The Author foresees the following potential issues. Stale presence data and the bucket system introducing
a undesired by some speed bump.

The speed bump issue is considered by the author as a feature not a bug. Without it realistically speaking presence
can never go federation wide as we have to batch this or else we are going to be sending 1 EDU at a time
for no reason when a 15s bucket might reach the max EDUs per transaction limit in the same situation.


And well stale presence is considered an acceptable sacrifice if this system works. Slightly flawed but
works good enough to make everyone happy presence is an order of magnitude better than it not existing at all.

## Alternatives

This author doesn't see too many alternatives but is happy to be informed of options. Please help contribute
alternatives that need to be addressed here.


## Security considerations

The author isn't sure there are too many new security considerations in this MSC but this MSC does
aim to fix the Flip Flop flaw in presence together with its partner in [MSC4043](github.com/matrix-org/matrix-spec-proposals/pull/4043)

## Unstable prefix

This MSC is too early in the process currently to even entertain a unstable implementation. The APIs are all missing and the
data to send over them is also missing. Please help flesh this MSC out before you even consider
implementations that interoperate and therefore need a unstable prefix.

## Dependencies

This MSC builds on [MSC4042](github.com/matrix-org/matrix-spec-proposals/pull/4042) and [MSC4043](github.com/matrix-org/matrix-spec-proposals/pull/4043)
(which at the time of writing have not yet been accepted into the specification).
