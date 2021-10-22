---
layout: default
title: Fabric_Website
nav_order: 3
---

- Feature Name: Fabric Website
- Start Date: (fill me in with today's  date, 2020-08-14)
- RFC PR: (leave this empty)
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC details the proposal for a new website for using Hyperledger Fabric. This website would serve as a "front door" for the project, and collate the various resources available to help users get started with Fabric.

The website has been deployed to Heroku for development purposes and is available to browse [here](https://warm-reaches-04614.herokuapp.com/).

# Motivation
[motivation]: #motivation

Presently, there are multiple different places to get started with Hyperledger Fabric. It can be difficult for a new user to know which resource is the best one for them to start with based on what they would like to achieve. The available documentation has excellent content for getting started and explains everything well, but there is also a large amount of additional content on other relevant topics (for example, performance testing) that is also useful to new users.

This website aims to surface this content and bring it with the existing documentation into one easy to consume place for users.

There is an existing website at https://www.hyperledger.org/use/fabric that serves as a landing page for people interested in Fabric. However this has several limitations - namely that it is based on a template that is used for all Hyperledger projects, and is unable to be extended by the fabric maintainers.

Other Hyperledger projects, such as [Sawtooth](https://sawtooth.hyperledger.org/), already have websites similar to what this RFC proposes, that collate all relevant resources into one place.  Adding a website with a similar purpose for Fabric will serve as a "front door" to the project, and give it a more approachable and inviting feel.

This website is for any user of Hyperledger Fabric, whether they are a developer of smart contracts or client applications, a network operator, or someone wishing to get involved in the Fabric community. It is for both new users wishing to get started with Fabric and existing users wanting to have a single place to access the resources they use most.

The expected outcome for this project is to have a fully functioning website for our users, which is continually being updated with the latest and best resources.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The current proposed structure for the website is that it will be made up of four primary sections, separated by tabs. These sections are based on the different types of Fabric user that could use this website as a resource. The sections are:

- An **Overview** tab, that will include general information about Hyperledger Fabric and what it is;
- An **Operators** tab, that will include an explanation of the role of a network operator and links to resources this type of user might find most useful, such as the operations guide and deployment guide;
- A **Developers** tab, that will include an explanation of the role of a developer and links to resources this type of user might find most useful, such as the VS Code extension and developer tooling, the commercial paper sample, and the latest chaincode and gateway SDK versions;
- A **Community** tab, that will include links to the most relevant resources for contributing to Fabric, such as the main Fabric gitHub repository, the Jira dashboard, and the contributing guide.

References and links common to all user types (StackOverflow, the Hyperledger Fabric wiki, social media profiles etc.) are included at the bottom of all tabs.

By laying the content out in this way, it is hoped that users will be able to easily locate the documentation they need to successfully achieve their Fabric use case. 


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The proposed website will be implemented using React. It will be deployed to GitHub pages, after which it will be accessible at the link hyperledger.github.io/fabric-website. After deployment, the URL https://fabric.hyperledger.org/ will also redirect to hyperledger.github.io/fabric-website, similar to the existing website for Sawtooth.

The website will not interact directly with other Fabric features or documentation. Instead, it is intended to serve as a "front door"; it should include links to all relevant resources, but not attempt to pull in or display their content directly.

The content that is included in the website will consist primarily of descriptions of use cases and links to appropriate tutorials or documentation for those use cases. It will include links to multiple versions of the documentation so that users have the correct information for the version of Fabric that they are using. These links will need to be updated/added to when new versions of these resources are made available.

