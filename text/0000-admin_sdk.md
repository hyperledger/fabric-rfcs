---
layout: default
title: RFC Template
nav_order: 3
---

- Feature Name: Fabric admin sdk
- Start Date: 2022-11-11
- RFC PR: (leave this empty)
- Fabric Component: core
- Fabric Issue: I remember there is a plan for decouple peer cli in fabric 3.0 roadmap? ... but I didn't get the open issue on fabric repo for it.

# Summary
[summary]: #summary

As we are going to have new gateway sdk and deprecated sdk. Admin capabilities will been deprecated together with sdk. Hence, for BAAS/deployment tools as [fabric-operator](https://github.com/hyperledger-labs/fabric-operator) has to use peer cli to deal with admin capabilities, such as channel creation. We have a full list. [here](https://github.com/Hyperledger-TWGC/fabric-admin-sdk/issues/15)
Those features will be built on fabric proto and decouple with fabric code, if implemented with golang.

# Motivation
[motivation]: #motivation

admin capabilities sdk repos basing on coding language as golang, js, java to allow BAAS/deployment tools as [fabric-operator](https://github.com/hyperledger-labs/fabric-operator) able to use those repo instead of invoking peer cli.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

- Introducing new named concepts.
> fabric-admin-sdk or fabric-admin-sdk-{$language}(fabric-admin-sdk-java)
admin short for admin capabilities.

- Explaining the feature largely in terms of examples.
> [See detail lists here, link to avoid duplication](https://github.com/Hyperledger-TWGC/fabric-admin-sdk/issues/15)

- Explaining how Fabric programmers should *think* about the feature, and how
  it should impact the way they use fabric. It should explain the impact as
  concretely as possible.
> for deployment tool or BAAS developers, using this admin-sdk features instead of using CLI.

- If applicable, provide sample error messages, deprecation warnings, or
  migration guidance.
> 1st Done with the admin capabilities features.
> 2nd as it able to replace peer cli by SDK, decouple peer cli from fabric repo to this repo for build.(parallel for migration)
> 3rd house keeping in fabric repo.

- If applicable, describe the differences between teaching this to established
  and new Fabric developers.
> n/A, well as in general speaking, the admin sdk should reuse all fabric concepts(such as:local MSP) and replace peer cli implementations... at user level there is no difference. 
> But for BAAS/deployment tool developer, well, it is an alternative of peer cli.

- If applicable, describe any changes that may affect the security of
  communications or administration.
> N/A, not applicable.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
> Decouple peer CLI.

- It is reasonably clear how the feature would be implemented.
> base on fabric proto, build language based sdk repos, without fabric code as cross reference for golang.

- Corner cases are dissected by example.
https://github.com/Hyperledger-TWGC/fabric-admin-sdk

# Drawbacks
[drawbacks]: #drawbacks
n/A

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
> n/A
- For community proposals: Is this done by some other community and what were
  their experiences with it?
A sample from TWGC at [here](https://github.com/Hyperledger-TWGC/fabric-admin-sdk), and confirmed with maintainers at [here](https://github.com/Hyperledger-TWGC/fabric-admin-sdk/issues/40) we are all good to contribute this repo to fabric.

- For other teams: What lessons can we learn from what other communities have
  done here?
> n/A  
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.
>n/A

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your RFC with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an RFC.  Please also take into consideration
that Fabric sometimes intentionally diverges from common distributed
ledger/blockchain features.

# Testing
[testing]: #testing

- What kinds of test development and execution will be required in order
to validate this proposal, beyond the usual mandatory unit tests?
> https://github.com/Hyperledger-TWGC/fabric-admin-sdk/actions/workflows/golange2e.yml take test-network in fabric sample as a sample, the admin sdk should able to pass all admin capabilities feature tests.

- List integration test scenarios which will outline correctness of proposed functionality.
> Same with above.


# Dependencies
[dependencies]: #dependencies
n/A

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
