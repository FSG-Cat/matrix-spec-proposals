# MSCXXXX: Remote Network Targeting Policies

This MSC introduces the `remote_network` and `remote_id` field as optional keys for membership events.
This has the purpose to allow policy application against bridged users without reliance on glob bans while still
allowing the ban to be shared across bridges.

The exact shape of the `remote_network` and `remote_id` are left relatively loose by this MSC because
its meant to account for all bridges. Past, current and future bridges should all be able to use the same
format. So instead of burdening the spec with having to keep a bridge directory up to date we dont.

This is achieved by having the policies just blindly match `remote_network` and `remote_id` that is supplied
by the bridge. This works on the simple logic that as long as you dont poison the data that we use
to write policies blind matching is perfectly adequate.

The policy half of this proposal will be achieved via extending the current `m.policy.rule.user` policy
with `remote_network` and `remote_id` fields just like membership events.

## Proposal

`remote_network` takes the form of a string that identifies the remote side of the bridge. This proposal
does not specify any strict rules for how remote networks should be identified except that the matrix
namespace rules are followed.

A example shape for how you identify a centralised network like telegram is `org.telegram`. This follows
the standard matrix naming conventions. This proposal is intentionally leaving this problem loosely defined
as to let bridges sort this out as its only purpose is to identify what namespace a `remote_id` belongs
inside of.

`remote_id` is defined as the remote networks immutable identifier for that user. Examples include
Discord and their snowflakes and all the equivalents out there. The example for matrix users would be
the MXID. The purpose of providing the `remote_id` is to give policies a immutable identifier from the
remote network to latch onto.

A `m.policy.rule.user` policy that contains a ban targeting one of these remote users can look like this.

```
{
  "content": {
    "entity": "@example_bridge_example_1420070400000:example.com",
    "reason": "example",
    "recommendation": "m.ban",
    "remote_network": "com.discord",
    "remote_id": "1420070400000",
  },
  "origin_server_ts": 1420070400000,
  "sender": "@example_bot:example.com",
  "state_key": "3KQYRDed/WMEod8n6vjheIgNk/+hsW4qq7WjJg1pQAM=",
  "type": "m.policy.rule.user",
  "event_id": "$example",
  "room_id": "!example"
}
```

This example event targets for bridges not compatible with this proposal the user in the entity field.

_Note: The Example Snowflake picked is not actually a snowflake but the unix timestamp of the discord epoch
its just filler after all._

But for bridges compatible with this proposal we are instead targeting any user whos membership event contains
a matching set of `remote_network` and `remote_id` fields.

When writing a policy that targets a user with `remote_network` and `remote_id` set you have to ensure you trust
said data to be accurate as to not ban innocent users. Its left as an implementation detail how exactly a
user is determined to belong to a trusted namespace.

In drafting this proposal a recommendation around bridge metadata tracked via state event was mentioned and
future work in this area is welcome to explore that avenue.

## Potential issues

This proposal does open the door to disagreement between bridges about how to identify a remote network
but that problem is left as a implementation detail on purpose.

This proposal does also not work well with networks that completely lack the idea of a true immutable identifier
that moves with your account in the case of networks that allow for account migration. This problem is
accepted as we cant address it and the vast majority of popular remote networks for matrix bridges
do have immutable identifiers.

## Alternatives

This metadata could be tracked via extensible profile like in [MSC4503](https://github.com/matrix-org/matrix-spec-proposals/pull/4503)
but this was dismissed in initial design as totally infeasible to the point of not being explored. The basis
for this instant rejection as a alternative worth consideration is that this metadata needs to be accessible
at 0 extra cost for anyone doing policy application or else its useless.

## Security considerations

The primary security flaw this MSC introduces is the fact we have to trust the bridges that provide
the metadata and that originate users.

Anyone can set these fields to whatever value they find entertaining or useful but as long as whoever
writes policies doesn't trust them the harm is limited. This proposal is intended to be used together
with approved bridges where the bridge is trusted to not lie. In the case of a lying bridge there can be damage
due to bad policies being issued. This is an accepted risk for the provided benefits.

## Unstable prefix

Before this proposal is stable the following prefix maps apply.

| Stable           | Unstable                                |
|------------------|-----------------------------------------|
| `remote_network` | `support.feline.mscXXXX.remote_network` |
| `remote_id`      | `support.feline.mscXXXX.remote_id`      |

## Dependencies

This proposal has no dependencies that are not part of the specification.
