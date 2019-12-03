---
layout: default
title: RFCs Process
nav_order: 2
---
# Hyperledger Fabric RFCs Process

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal [GitHub pull request workflow](https://guides.github.com/introduction/flow/).

Some changes though are substantial, and we ask that these be put through a bit
of a design process and produce a consensus among the Fabric maintainers and
broader community.

The RFC (request for comments) process is intended to provide a consistent and
controlled path for major changes to Fabric and other official project
components, so that all stakeholders can be confident about the direction in
which Fabric is evolving.

This process is intended to be substantially similar to the RFCs process other
Hyperledger teams have adopted, customized as necessary for use with Fabric.
The `README.md` and `0000-template.md` files were forked from the
[Sawtooth RFCs repo](https://github.com/hyperledger/sawtooth-rfcs), which was
derived from the Rust project.

## Table of Contents

- [When you need to follow this process]
- [Before creating an RFC]
- [What the process is]
- [The RFC life-cycle]
- [Reviewing RFCs]
- [Implementing an RFC]
- [License]
- [Contributions]

## When you need to follow this process

[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make substantial changes to
Fabric or any of its sub-components including but not limited to
fabric-baseimage, fabric-sdk-node, fabric-sdk-java, fabric-ca,
fabric-chaincode-go, fabric-chaincode-java, fabric-chaincode-node,
fabric-chaincode-evm, fabric-protos, fabric-protos-go, or the RFC process
itself. What constitutes a substantial change is evolving based on community
norms and varies depending on what part of the ecosystem you are proposing to
change, but may include the following:

- Architectural changes
- Substantial changes to component interfaces
- New core features
- Backward incompatible changes
- Changes that affect the security of communications or administration

Some changes do not require an RFC:

- Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not
change meaning".
- Additions that strictly improve objective, numerical quality criteria
(warning removal, speedup, better platform coverage, more parallelism, trap
more errors, etc.).

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.

## Before creating an RFC

[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected changes, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the RFC can
make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from other project developers
beforehand, to ascertain that the RFC may be desirable; having a consistent
impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include
talking the idea over to the [Fabric mailing list](https://lists.hyperledger.org/g/fabric/topics).

As a rule of thumb, receiving encouraging feedback from long-standing
project developers, and particularly the project's maintainers is a good
indication that the RFC is worth pursuing.

## What the process is

[What the process is]: #what-the-process-is

In short, to get a major feature added to Fabric, one must first get the RFC
merged into the RFC repository as a markdown file. At that point the RFC is
"active" and may be implemented with the goal of eventual inclusion into
Fabric.

- Fork [the RFC repository](https://github.com/hyperledger/fabric-rfcs).
- Copy `0000-template.md` to `text/0000-my-feature.md`, where "my-feature" is
descriptive. Don't assign an RFC number yet.
- Fill in the RFC. Put care into the details — RFCs that do not present
convincing motivation, demonstrate understanding of the impact of the design,
or are disingenuous about the drawbacks or alternatives tend to be
poorly received.
- Submit a pull request. The pull request will be assigned to a maintainer, and
will receive design feedback from the larger community; the RFC author should
be prepared to revise it in response.
- Build consensus and integrate feedback. RFCs that have broad support are much
more likely to make progress than those that don't receive any comments. Feel
free to reach out to the pull request assignee in particular to get help
identifying stakeholders and obstacles.
- The maintainers will discuss the RFC pull request, as much as possible in the
comment thread of the pull request itself. Offline discussion will be
summarized on the pull request comment thread.
- A good way to build consensus on a RFC pull request is to summarize the RFC on a
community contributor meeting. Coordinate with a maintainer to get on a contributor
meeting agenda. While this is not necessary, it may help to foster sufficient
consensus such that the RFC can proceed to final comment period.
- RFCs rarely go through this process unchanged, especially as alternatives and
drawbacks are shown. You can make edits, big and small, to the RFC to clarify
or change the design, but make changes as new commits to the pull request, and
leave a comment on the pull request explaining your changes. Specifically, do
not squash or rebase commits after they are visible on the pull request.
- At some point, a Fabric maintainer will propose a "motion for final comment
period" (FCP), along with a *disposition* for the RFC (merge, close, or
postpone).
  - This step is taken when enough of the tradeoffs have been discussed that
  the maintainers are in a position to make a decision. That does not require
  consensus amongst all participants in the RFC thread (which is usually
  impossible). However, the argument supporting the disposition on the RFC
  needs to have already been clearly articulated, and there should not be a
  strong consensus *against* that position. Fabric maintainers will use their
  best judgment in taking this step, and the FCP itself ensures there is ample
  time and notification for stakeholders to push back if it is made
  prematurely.
  - For RFCs with lengthy discussion, the motion to FCP is usually preceded by
  a *summary comment* trying to lay out the current state of the discussion and
  major trade-offs/points of disagreement.
  - Before actually entering FCP, *the majority* of maintainers must sign off;
  this is often the point at which many maintainers first review the RFC in
  full depth.
- The FCP lasts one week, or seven calendar days. It is also advertised widely,
e.g. in the [Fabric Mailing List](https://lists.hyperledger.org/g/fabric/topics).
This way all stakeholders have a chance to lodge any final objections before a
decision is reached.
- In most cases, the FCP period is quiet, and the RFC is either merged or
closed. However, sometimes substantial new arguments or ideas are raised, the
FCP is canceled, and the RFC goes back into development mode.

## The RFC life-cycle

[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes "active" then authors may implement it and submit the
change as a pull request to the corresponding Fabric repo. Being "active" is
not a rubber stamp, and it does not mean the change will ultimately be merged;
it does mean that in principle all the major stakeholders have agreed to the
change, and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a Fabric developer has been assigned the task
of implementing the feature. While it is not *necessary* that the author of the
RFC also write the implementation, it is by far the most effective way to see
an RFC through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to active RFCs can be done in follow-up pull requests. We strive
to write each RFC in a manner that it will reflect the final design of the
feature; but the nature of the process means that we cannot expect every merged
RFC to actually reflect what the end result will be at the time of the next
major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new RFCs, with a note added to the original RFC. Exactly what counts
as a "very minor change" is up to the maintainers to decide.

## Reviewing RFCs

[Reviewing RFCs]: #reviewing-rfcs

While the RFC pull request is up, the maintainers may schedule meetings with
the author and/or relevant stakeholders to discuss the issues in greater
detail, and in some cases the topic may be discussed at a contributors meeting.
In either case a summary from the meeting will be posted back to the RFC pull
request.

The Fabric maintainers make the final decisions about RFCs after the benefits
and drawbacks are well understood. These decisions can be made at any time, but
the maintainers will regularly issue decisions. When a decision is made, the
RFC pull request will either be merged or closed. In either case, if the
reasoning is not clear from the discussion in thread, the maintainers will add
a comment describing the rationale for the decision.

## Implementing an RFC

[Implementing an RFC]: #implementing-an-rfc

Some accepted RFCs represent vital changes that need to be implemented right
away. Other accepted RFCs can represent changes that can wait until a
developer feels like doing the work. Every accepted RFC has an associated
issue tracking its implementation in the [Fabric JIRA issue tracker](https://jira.hyperledger.org/projects/FAB/issues).

The author of an RFC is not obligated to implement it. Of course, the RFC
author, as any other developer, is welcome to post an implementation for review
after the RFC has been accepted. Use JIRA for this.

## License

[License]: #license

This repository is licensed under [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
([LICENSE](LICENSE)).

## Contributions

[Contributions]: #contributions

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be licensed as above, without any additional terms or conditions.
