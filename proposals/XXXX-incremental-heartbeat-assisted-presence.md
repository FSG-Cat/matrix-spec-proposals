# MSC0000: Incremental Heartbeat Assisted Presence

## Introduction

Welcome to Incremental Heartbeat Assisted Presence. This MSC is a derivative of my earlier work in
the form of this [gist](https://gist.github.com/FSG-Cat/bd621bac3346497e2c336362d3260ddb). And builds upon it
with my knowledge from the 2 years that has passed since i wrote that document on my iPad during downtime
and time i was bored during class.

### Background 

Presence is known as one of those nice to haves that is a privilege of us homeserver admins who have
the resources to burn on it. All the homeserver software's recognise this as a very taxing feature if scaled
up to matrix.org scale. Also incidents like the Sliding Sync proxy going nuts or the presence flooding
i triggered accidentally shows the fragility and bad design or code or both involved in presence.

### High level overview of how to fix this.

So as the background establishes presence as it stands today is flawed. So how do we fix it?

This MSC proposes fixing presence by taking a page out of the routing protocol world. The precursor to
this MSC in the form of the gist from earlier exists due to my studies having us learn the ins and outs of
the EIGRP and OSPF routing protocols. It turns out that these routing protocols have a similar problem to us.
They need to distribute Alive or Dead status to all the nodes so your routing tables are up to date.

And what is presence if not Alive or Dead status information. Now OSPF takes the problematic but works at limited
enough scale approach of sending the whole thing every time anything changes. Its a known flaw but it works. 
Now EIGRP uses incremental updates and that's exactly what i propose we copy. This proposal therefore proposes
that we change presence to be Changes only and move expiry into a separate subsystem of presence.

To facilitate changes only presence you need to keep track of if the other party is alive or dead. I mean 
if matrix.org is mid database upgrade that takes 6 hours you wouldn't want her users to be online reported for the duration.
(Yes matrix.org had a db upgrade that took like 6 hours. Its fine my point was just an example of a long single day maintenance window.)
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

So while the high level overview has given a pretty clear picture of this proposal its not a complete
picture. One example of a system that lacked elaboration was the Heartbeat system that gives its name
to this proposal.

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

FIXME: The exact shape of the API used for heartbeats is for a Revision 2 of this proposal as its currently a 
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

Under this proposal presence adopts a succession order where only the state highest in the succession
order is used for your account wide presence state. Under the current presence states and disabled from 
[MSC4042](https://github.com/matrix-org/matrix-spec-proposals/pull/4042) the order is `online` > `unavailable` > `offline` > `disabled`.

The `unavailable` state is either set by the client using a last interaction timestamp or assumed by the server
after a sufficient period of inactivity assuming that the user has this state enabled.

Note to Cat: What goes here in-between this and the S2S Section? Figure this out before posting Rev 1 as a MSC on the repo.

### Incremental presence

Upon initial contact with a homeserver or after being asked for a full table you send a full copy of your
presence table that the remote homeserver in question is interested in. This presence table is considered
valid under the rules in the heartbeat system defined earlier in this proposal or until a change is made to
a entry. When a user changes their presence state that change is updated on the server side and put in a bucket.
Once this bucket is old enough or full enough you send that whole batch of presence updates to all remote servers
that are interested. The size of this bucket and maximum delay is a implementation detail as long as the resulting
transaction conforms to S2S API rules. 

Presence states in this system are valid until a change is communicated and servers are to only send changes
when not requesting a full copy due to having dropped its copy. The most common cause for dropping your copy is heartbeat
expiry. 

FIXME: Consider if a recommendation to drop table on restart is a good idea or not?

Status message changes are also considered changes ofc under this proposal as status messages are part
of the presence EDUs.


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

The author isn't sure there are too many new security considerations in this proposal but this proposal
aims to fix the Flip Flop flaw in presence together with its partner in [MSC4043](github.com/matrix-org/matrix-spec-proposals/pull/4043)

## Unstable prefix

This proposal is too early in the process currently to even entertain a unstable implementation. The APIs are all missing and the
data to send over them is also missing. Please help flesh this proposal out before you even consider
implementations that interoperate and therefore need a unstable prefix.

## Dependencies

This MSC builds on MSC4042 and MSC4043 (which at the time of writing have not yet been accepted into the specification).
