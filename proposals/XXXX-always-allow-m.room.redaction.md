# MSCXXXX: Always allow m.room.redaction

## Introduction 

This proposal is very simple in what it aims to do. It removes the ability to block the sending of `m.room.redaction`
events as long as you are a member of said room with membership of `join`. 

## Proposal

To make it so you can always send a m.room.redacts if your a member of the room with membership of `join`
a new Authorisation rule will be introduced. Based on the room version 11 authorisation rules its
number 7. It reads as "If type is `m.room.redacts`, allow"

## Potential issues

The author is currently not aware of why this "feature" was implemented so the author cant address
the issues properly that are reintroduced.

But the author also cant foresee any issues that cant be easily mitigated or are fabricated. That aren't
outclassed by the T&S flaw that this "feature" existing presents.


## Alternatives

None exist that the Author is aware of.

## Security considerations

This should not be security relevant to event authorisation as all event authorisation critical fields are supposed
to be protected from redaction already. 

This feature existing is a T&S flaw as far as the author is concerned as it allows you to strip users of the ability
to use redactions for their intended purposes. And as clients fail to communicate this mode being active it causes
problems. This feature has too much room for abuse to be reasonable able to be justified as far as the author sees it.
Especially as tooling exists that allows you to replicate the behaviour as if this MSC did not exist nor get implemented
if you need it for a private federation.

This MSC concerns it self's with the needs and T&S concerns of the public federation.

This MSC concerns it selfs with the needs and T&S concerns of the public federation.

## Unstable prefix

While MSCXXXX is not stable implementations should use `support.feline.mscXXXX.rev.1` as the room version identifier, using [v11](https://spec.matrix.org/v1.9/rooms/v11) as a base.
