# MSC0000: Policy Documents

In the spec process sometimes MSCs try to change things in a way that disagrees with policies on what matrix even is.

Examples of documents that currently define these policies are the spec it self and the principles of matrix therein.

This MSC proposes to make it so you can not refer to non public documents as references to these policies.

## Proposal

Under this proposal the MSC process is changed to only allow invoking documents that are public and have been approved
via the MSC process when saying this isn't matrix. So if something violates for example
the matrix security model you refer to the matrix security model document.

You are still allowed to nix MSCs based on the opinion that something is just a plain old bad idea
but this cant be used as an appeal to authority as your not able to appeal to the established policies.

We propose that feedback, review, and proposals MUST NOT refer to
documents, ideas, processes that are not publicly available for
review and scrutiny except in extenuating circumstances defined below
(for example, a security issue).

This is to avoid situations where reviewers refer to an inaccessible
authority in order to provide an unfalsifiable argumentation in their
review. Because this stops feedback, or defences of proposals,
reviews, and proposals themselves from receiving fair evaluation and
scrutiny. And at the moment there is an unjust asymmetry in the review
process because of the ability for the SCT to use inaccessible
authorities to dismiss or raise concerns without scrutiny, and without
any means of verification.

### Providing a process for appealing to inaccessible authority

The SCT should still have the discretion to appeal to an inaccessible
authority, however they MUST first seek approval from a to be
established governing board committee or working group so that the
claims can be verified or scrutinised BEFORE they are used in a
review.

This process does not exist to determine whether an appeal to
inaccessible authority is justified, it is to obtain access to any
documentation, evaluate the claims, and verify the authority.

This mechanism is deliberately proactive so that appeals to
inaccessible authority cannot be made in excess or under the
radar. Which would wear down the governing board's transparency
mechanism, as they would otherwise have to chase down each claim and
investigate in reaction. Which would again provide an asymmetry in the
SCT's favour.

## Potential issues

This new process is in theory more complicated since you have to know what
document your going to appeal to when trying to nix something on the basis of violating a policy.
The problem with this argument is that you should know what document
your invoking when nixing something based on a policy.

## Alternatives

*This is where alternative solutions could be listed. There's almost always another way to do things
and this section gives you the opportunity to highlight why those ways are not as desirable. The
argument made in this example is that all of the text provided by the template could be integrated
into the proposals introduction, although with some risk of losing clarity.*

Instead of adding a template to the repository, the assistance it provides could be integrated into
the proposal process itself. There is an argument to be had that the proposal process should be as
descriptive as possible, although having even more detail in the proposals introduction could lead to
some confusion or lack of understanding. Not to mention if the document is too large then potential
authors could be scared off as the process suddenly looks a lot more complicated than it is. For those
reasons, this proposal does not consider integrating the template in the proposals introduction a good
idea.

## Security considerations

Security considerations are covered in the [proposal](#proposal) section.

## Unstable prefix

Not applicable. This MSC is about Process not about code.

## Dependencies

This MSC has no dependencies.
