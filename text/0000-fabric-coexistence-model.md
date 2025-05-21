---
layout: default
title: Fabric Ecosystem Coexistence Model
nav_order: 3
---

* Feature Name: Fabric Ecosystem Coexistence Model
* Start Date: 2025-05-20
* RFC PR: (leave this empty)
* Fabric Component: Ecosystem Architecture, Governance
* Fabric Issue: (leave this empty)

# Summary

This RFC proposes a strategic evolution for the Hyperledger Fabric ecosystem, transitioning from a "one-size-fits-all" architecture to a **Fabric Ecosystem Coexistence Model**. This model formally allows multiple, distinct Fabric-derived **Sub-Projects (referred to as implementations in this RFC for conceptual clarity)** (e.g., the current "Fabric Classic," a high-performance "Fabric-X" for CBDCs, or a future "Fabric-Y" for IoT) to be developed and recognized under the unified Hyperledger Fabric project umbrella. The adoption of this model **is supported by foundational updates to the Hyperledger Fabric Technical Charter and its governance structures** [Link to Technical Charter PR](https://github.com/hyperledger/fabric/pull/5223), empowering the Technical Steering Committee (TSC) to effectively govern this broader ecosystem of **Sub-Projects**.

# Motivation

Hyperledger Fabric's initial design provided a robust, general-purpose DLT platform. However, as the blockchain landscape matures and enterprise adoption demands more specialized solutions, a single, monolithic architecture faces inherent limitations in optimally serving an increasingly diverse set of requirements. While core Fabric concepts are highly reusable and proven, their specific instantiations and the surrounding platform features often need domain-specific adaptations.

Attempting to integrate all conceivable specializations into a single codebase risks making that codebase overly complex. The "one-size-fits-all" approach can become a bottleneck.

The primary motivations for this Ecosystem Coexistence Model are:

1.  **Strategic Adaptability & Specialization**: To formally enable the creation of distinct Fabric-based **Sub-Projects (implementations)** optimized for specific domains when their requirements are demonstrably and prohibitively difficult or architecturally impractical to implement within existing **Sub-Projects** without significant compromises.
2.  **Ecosystem Revival & Sustained Growth**: To invigorate the Fabric ecosystem by creating clear pathways for innovation.
3.  **Preventing Stagnation and Fragmentation**: Without a flexible model, the core project risks becoming less adaptable, leading to stagnation or uncontrolled fragmentation.
4.  **User Choice with Conceptual Consistency**: To offer users a family of **Sub-Projects** built upon familiar Fabric principles.
5.  **Leveraging and Reinforcing Core Strengths**: To ensure that all **Sub-Projects** within the ecosystem continue to leverage and reinforce the robust foundational concepts of Fabric (illustrative examples including execute-order-validate, MSP, KVV data model, and endorsement policy DSL, with the formal list of core principles to be maintained by the TSC as per the Technical Charter \[Placeholder for Link to Technical Charter PR\]), thereby maintaining a cohesive brand.

The expected outcome is a more vibrant, adaptable Hyperledger Fabric ecosystem. This RFC outlines a model that is enabled by [proposed updates to the Fabric Technical Charter](https://github.com/hyperledger/fabric/pull/5223).

# Guide-level explanation

Imagine the Hyperledger Fabric ecosystem evolving into a constellation of recognized **Sub-Projects (or implementations)**. Examples:

* **Sub-Project Classic**: The established, general-purpose Fabric.
* **Sub-Project X**: A new implementation optimized for extreme high-throughput, sharing core principles but perhaps with a different transaction structure or programming model.
* **Sub-Project Y**: A future implementation, perhaps for IoT, adhering to core principles but differentiated.

**How Fabric Developers and Users Should Think About This:**

Developers select the **Sub-Project (implementation)** that best aligns with their needs.

* **Conceptual Familiarity:** Core concepts will be transferable.
* **Expected Differences:** Be prepared for differences in:
    * Specific APIs and SDKs.
    * Available Features and Configurations.
    * Performance Characteristics.
    * Governance Models and Release Cadences: Each recognized **Sub-Project** will have its own maintainers and release cycle, operating under the overarching Fabric TSC, whose [**updated Technical Charter**](https://github.com/hyperledger/fabric/pull/5223) governs this model.
    * Operational Tooling and Deployment Models.

**Setting Expectations and Ensuring Clarity:**

* **Clear Documentation & Branding:** Comprehensive ecosystem documentation will delineate:
    * Specifics of each recognized **Sub-Project**.
    * Guidelines for choosing a **Sub-Project**.
    * Branding guidelines (e.g., "Hyperledger Fabric Classic," "Hyperledger Fabric-X"), as defined by the TSC, will ensure clarity.
* **Migration Paths:** Direct, automated migration paths between distinct **Sub-Projects** will **not be guaranteed** initially.
* **Burden of Proof for New Sub-Projects:** The creation of a new **Sub-Project** would be subject to its own rigorous technical RFC, demonstrating adherence to core principles and compelling justification.

**Impact on Security and Administration:**

Each coexisting **Sub-Project** would have its own dedicated maintainers.

* **Security considerations and patches** managed by respective maintainers.
* **Administrative procedures** specific to each.
* The **overarching Hyperledger Fabric Technical Steering Committee (TSC)**, operating under its [**updated Technical Charter**](https://github.com/hyperledger/fabric/pull/5223), would provide top-level governance, ensuring adherence to ecosystem-wide principles and managing the recognition process for new **Sub-Projects**.

# Reference-level explanation

This RFC introduces a policy and operational framework change, enabling a new working model. It aligns with the [**updated Hyperledger Fabric Technical Charter**](https://github.com/hyperledger/fabric/pull/5223) and the processes for recognizing and governing multiple, coexisting Fabric **Sub-Projects (implementations)**.

Core implications:

1.  **Clear Justification for New Sub-Projects:** A proposal for a new **Sub-Project** must provide comprehensive justification, including adherence to core principles (as defined by the TSC per the Technical Charter), purpose, evidence of necessity, architectural overview, and proposed initial maintainers, aligning with the governance structure in the [**updated Technical Charter**](https://github.com/hyperledger/fabric/pull/5223).
2.  **No Guaranteed Cross-Implementation Migration Paths (Initially):** Due to specialization, direct migration is not a default.
3.  **Distinct Governance, Maintenance, and Releases:** The governance model is detailed in the updated Hyperledger Fabric Technical Charter.

**Process for Introducing a New Coexisting Sub-Project:**

1.  **This RFC (Ecosystem Coexistence Model):** If accepted, it establishes the framework and aligns with the **proposed Fabric Technical Charter** that supports this model.
2.  **Community Review and TSC Approval (under current, updated Charter**): The specific **Sub-Project** RFC would undergo review and require TSC approval, based on merit, adherence to principles, justification, and alignment with ecosystem goals, as per the processes in the updated Charter.
3.  **Repository Creation and Development:** Upon approval, dedicated repositories would be created.

# Governance Model for the Fabric Ecosystem (as per the Updated Technical Charter)
The governance model for the Fabric ecosystem, including details on Sub-Project Maintainers, the Ecosystem-Level Governance by the Fabric TSC, and voting procedures, is detailed in the [proposed Hyperledger Fabric Technical Charter](https://github.com/hyperledger/fabric/pull/5223).

# Drawbacks
[drawbacks]: #drawbacks

* **Potential User Confusion**: Despite best efforts, having multiple "Fabric" implementations could initially confuse users regarding which to choose, their specific capabilities, and their differences.
    * *Mitigation*: Strong, clear, centralized documentation; distinct branding and naming conventions (e.g., "Hyperledger Fabric Classic," "Hyperledger Fabric-X") defined by the TSC; clear guidance on use-case suitability; active community support and education.
* **Fragmentation of Development Effort and Community Focus**: Developer attention, contributions, and community discussions could become spread across multiple implementations.
    * *Mitigation*: Fostering a sense of a unified Fabric *ecosystem* with shared core principles; encouraging cross-implementation learning and collaboration where appropriate; ensuring each implementation has a clear value proposition to attract a dedicated community; active stewardship by the Fabric TSC to promote overall ecosystem health and identify synergies.
* **Increased Governance Overhead**: Managing an ecosystem of multiple distinct implementations will introduce additional complexity for the overarching Fabric TSC.
    * *Mitigation*: A well-defined and amended Fabric TSC charter clearly outlining its role in stewarding the ecosystem, managing the lifecycle of implementations, and delegating appropriate autonomy to individual implementation maintainer groups. Effective processes and potentially dedicated sub-committees or working groups within the TSC structure.
* **Resource Contention**: Multiple implementations might compete for shared community resources (e.g., CI/CD infrastructure, marketing focus, general support channels).
    * *Mitigation*: Clear policies for resource allocation managed by the Fabric TSC; encouraging implementations to also seek their own dedicated resources or sponsors where appropriate.
* **Learning Curve for Users and Developers**: Users and developers may need to learn the nuances of different implementations if their needs span across them.
    * *Mitigation*: Emphasizing the shared core principles to provide a consistent conceptual foundation; excellent documentation highlighting differences and specific use cases; community training and educational materials.

# Rationale and alternatives
[alternatives]: #rationale-and-alternatives

**Why is this Ecosystem Coexistence Model the best approach?**

This federated, coexistence model, governed by an overarching TSC operating under an evolved Technical Charter, is deemed the most effective for the long-term health and relevance of Hyperledger Fabric because:

1.  **Fosters Targeted Innovation for Critical Use Cases**: It allows specialized solutions (like a Fabric-X for >100k TPS) to be developed and optimized rapidly when their specific, demanding requirements are demonstrably too difficult or disruptive to integrate into existing, general-purpose Fabric implementations without significant compromise.
2.  **Addresses Diverse Market Needs Strategically**: It directly responds to the reality that "one-size-fits-all" is insufficient for the breadth of high-value, specialized blockchain use cases emerging. This allows the Fabric *ecosystem* to cater to a wider market.
3.  **Maintains Conceptual Cohesion and Trust**: Unlike completely independent forks, this model *mandates* adherence to core Fabric principles (EOV, MSP, KVV, etc.), providing a familiar conceptual foundation and preserving the trust associated with the Fabric brand. It's about *managed diversity*, not uncontrolled divergence, under the stewardship of the Fabric TSC.
4.  **Prevents Stagnation and Encourages Renewal**: It offers a clear, governed path for the Fabric ecosystem to evolve, adapt to new technological paradigms (like microservices for performance), and remain relevant, rather than risking obsolescence by not being able to address new, demanding requirements.
5.  **Preserves Stability of Existing Implementations**: New, potentially disruptive architectural explorations (like Fabric-X) can occur in dedicated repositories without destabilizing the existing Fabric Classic codebase, which supports many production systems.
6.  **Attracts Specialized Talent and Communities**: Distinct implementations focused on specific problem domains can attract developers and organizations with specialized expertise and interest in those areas, enriching the overall Fabric talent pool.

**Alternatives Considered:**

1.  **Continue with a Single, Monolithic, Highly Configurable Fabric Implementation**: Attempt to evolve the single Fabric codebase to cater to all needs.
    * *Rationale for not choosing*: This approach becomes exponentially complex to design, test, maintain, and configure. For use cases with fundamentally distinct architectural needs or extreme performance demands (e.g., Fabric-X's microservice architecture for >100k TPS), attempting to shoehorn these into a single, evolving monolith is often impractical, leads to a "lowest common denominator" feature set, risks compromising existing functionalities, or makes the primary implementation too unwieldy and difficult to manage for everyone. As has been observed, "one size fits all" has not served well for pushing performance boundaries.
2.  **Encourage Independent Forks (No Coexistence Model / No Official Recognition):** Allow any group to fork Fabric and develop independently, without any formal relationship or shared ecosystem branding under the Hyperledger Fabric project.
    * *Rationale for not choosing*: This would lead to greater fragmentation, loss of shared identity and trust, potential brand confusion if forks still use "Fabric" in their name without oversight, and make it harder for users to identify solutions based on Fabric's proven concepts. It also misses the opportunity for shared learning, potential (even if not initially guaranteed) cross-pollination of ideas, and leveraging the established Hyperledger Fabric governance (even an evolved one) and community structures.
3.  **Attempt Extreme Modularity within a Single Fabric Implementation to Achieve "Personalities"**: Try to make the current Fabric codebase so modular that entirely different operational "personalities" (e.g., one for high-throughput, another for general use) can be compiled or enabled from the same core.
    * *Rationale for not choosing*: While modularity is a good design principle, achieving the level of modularity required to fundamentally alter core architectural patterns (e.g., monolithic vs. microservices), data storage models (e.g., embedded LevelDB/CouchDB vs. external distributed SQL DB per org), and consensus mechanisms (e.g., Raft vs. a sharded BFT like Arma) while maintaining a single, coherent, and manageable framework is exceptionally challenging, likely impossible without creating an internally fragmented and complex system. The distinct operational and governance needs of such different "personalities" would also be hard to manage under a single implementation's maintainer group.

**Impact of not doing this (Rejecting the Coexistence Model):**

* The Hyperledger Fabric ecosystem may stagnate, becoming unable to effectively compete for or support emerging, high-value use cases that require specialized platform characteristics (because modifying the existing Fabric implementation sufficiently is too difficult or risky for these specialized cases).
* This could lead to a gradual decline in relevance and adoption as users seeking such capabilities turn to other DLT platforms that are either designed for those niches or offer a more flexible ecosystem model.
* Valuable innovations and contributions (like the research underpinning Fabric-X) might occur outside the official Fabric project, leading to fragmentation and missed opportunities for the broader community.
* The Fabric brand might become associated only with its current capabilities, limiting its perceived potential for future growth areas.

# Prior art
[prior-art]: #prior-art

The concept of a technology ecosystem supporting multiple, related but distinct implementations or distributions, often sharing a common core or philosophy, is well-established and provides valuable precedents for this proposal. These examples demonstrate that allowing for specialized, semi-autonomous entities within a larger ecosystem is a proven model for fostering innovation, catering to diverse needs, and ensuring long-term vitality.

* **The Java Ecosystem:**
    * **Core Concepts/Shared Foundation:** The Java Language Specification, the Java Virtual Machine (JVM) Specification, and the Java Standard Edition (SE) APIs form a universal foundation, enabling the "Write Once, Run Anywhere" philosophy. This shared core ensures that fundamental language constructs, memory management principles, and core utilities are consistent across different Java environments.
    * **Distinct Implementations/Editions for Different Scales:**
        * **Java SE (Standard Edition):** Provides the core libraries and runtime for general-purpose desktop and server applications. It's the bedrock upon which other editions are built.
        * **Jakarta EE (formerly Java EE - Enterprise Edition):** A set of specifications (with multiple vendor implementations like WildFly, GlassFish, WebSphere Liberty, OpenLiberty) built atop Java SE. It is specifically tailored for large-scale, multi-tiered, server-side enterprise applications, offering rich APIs for web services (e.g., Servlets, JAX-RS), persistence (JPA), messaging (JMS), and distributed components (EJBs, CDI).
        * **Java ME (Micro Edition - historically significant):** A configuration of Java designed for resource-constrained environments like older mobile phones, PDAs, and embedded systems. It featured a much smaller API set (a subset of Java SE) and optimized, lightweight JVMs (like KVM).
    * **Ecosystem Strength & Migration Incompatibility:** The Java ecosystem's strength lies in this tiered approach. A complex Jakarta EE application, with its dependencies on application server infrastructure, enterprise beans, and extensive APIs, is **fundamentally incompatible for direct migration** to a Java ME environment, which lacks these APIs and operates under severe resource constraints. Developers choose the edition appropriate for their target deployment. Similarly, while Java SE forms the base, one doesn't simply "run" a full Jakarta EE application in a bare Java SE environment without an application server that implements the EE specifications. This specialization, while creating migration barriers between editions, allowed Java to be highly relevant across an enormous spectrum, from enterprise data centers to small consumer devices. The Fabric Ecosystem Coexistence Model aims for similar adaptability by allowing specialized Fabric implementations, acknowledging that direct migration between them may not be feasible or the primary goal.

* **Microsoft Windows Operating System Ecosystem:**
    * **Core Concepts/Shared Foundation:** A common Windows API heritage (Win32/WinRT at different levels), the NT kernel architecture (for most modern versions), and core operating system concepts like the file system (NTFS), security model (ACLs, user accounts), and process/thread management.
    * **Distinct Implementations/Editions for Different Scales:**
        * **Windows Server (e.g., Windows Server 2022, 2019):** Implementations optimized for server hardware and enterprise workloads. They include features for managing extensive networks, hosting large-scale applications, virtualization (Hyper-V), Active Directory services, and robust security and management tools designed for data center operations.
        * **Windows Desktop (e.g., Windows 10, Windows 11):** Implementations tailored for end-user PCs and workstations, focusing on user interface, broad application compatibility, multimedia, and personal productivity.
        * **Windows IoT (e.g., Windows IoT Enterprise, Windows IoT Core):** Specialized implementations for Internet of Things devices. IoT Enterprise is a full binary version of Windows Enterprise but licensed for dedicated, often fixed-function devices. Windows IoT Core is a minimal footprint version, lacking a traditional Windows shell, designed for resource-constrained devices like Raspberry Pi or industrial controllers, focusing on specific application hosting.
    * **Ecosystem Strength & Migration Incompatibility:** Microsoft maintains these distinct Windows implementations because the operational requirements, feature sets, and resource assumptions for a global datacenter server are vastly different from those of a personal computer or an industrial sensor. An application designed for the full feature set of Windows Server (e.g., relying heavily on Active Directory integration or specific server roles) **cannot be simply "migrated" or run** on Windows IoT Core, which lacks many of those services and APIs. This specialization, however, allows the Windows ecosystem to be relevant and functional across an incredibly broad spectrum of computing needs, all while sharing a common Windows identity and a degree of developer familiarity with core concepts. The Fabric Coexistence model seeks this same strength through specialized implementations, understanding that application portability between these distinct implementations will be limited.

* **Python Programming Language Ecosystem:**
    * **Core Concepts/Shared Foundation:** The Python language specification itself – its syntax, semantics, dynamic typing, and core data types (lists, dictionaries, etc.). The philosophy emphasizes readability and ease of use.
    * **Distinct Implementations/Editions for Different Scales:**
        * **CPython:** The reference implementation and most widely used version of Python. It's suitable for a vast range of applications on desktops, servers, and cloud platforms, featuring an extensive standard library and access to a massive ecosystem of third-party packages (e.g., Django, Flask for web; NumPy, Pandas, Scikit-learn for data science).
        * **MicroPython & CircuitPython:** These are lean and efficient implementations of the Python 3 programming language, specifically designed and optimized to run on microcontrollers (like ESP32, STM32, Raspberry Pi Pico) and other embedded devices. They include a carefully selected subset of the Python standard library and provide modules for direct hardware interaction (GPIO, I2C, SPI, ADC).
    * **Ecosystem Strength & Migration Incompatibility:** Python's adaptability allowed it to successfully penetrate the embedded systems and IoT world through these specialized implementations. However, one **cannot take a large Django web application**, with its complex ORM, templating engine, and numerous dependencies built for CPython, and run it directly on a microcontroller using MicroPython. The differences in available memory (kilobytes vs. gigabytes), processing power, the subset of the standard library available, and the absence of an operating system environment in many MicroPython contexts make direct migration impossible. The application logic must be fundamentally rethought and rewritten for the embedded target. By having these distinct implementations, the Python *language* and its programming paradigm can be applied effectively in both powerful server environments and highly resource-constrained embedded contexts. This is analogous to how Fabric-X aims to extend Fabric's core principles to new, demanding environments where the original implementation's architecture is not a fit.

These examples robustly demonstrate that allowing for specialized, semi-autonomous entities that share a common technological or philosophical heritage within a larger ecosystem is a proven and highly successful model for fostering innovation, catering to diverse and evolving needs, and ensuring long-term vitality and relevance. The Fabric Ecosystem Coexistence Model seeks to apply these lessons to Hyperledger Fabric.

# Testing
[testing]: #testing

As this RFC proposes a policy and operational framework change for the Hyperledger Fabric project, traditional software testing methodologies are not directly applicable to the RFC itself. The validation of this *model* will be an ongoing process based on its successful adoption and the health of the resulting ecosystem. Key indicators of success will include:

* **Clarity, Adoption, and Effectiveness of the Evolved Governance Model**: Successful establishment and community adoption of clear, fair, and efficient governance processes (within the Fabric TSC, guided by an amended Charter) for proposing, evaluating, recognizing, and overseeing coexisting implementations. This includes well-defined criteria for demonstrating adherence to core Fabric principles and justifying the need for a new implementation.
* **Active Community Engagement and Contribution**: Healthy levels of participation, contribution, and discussion across the various recognized Fabric implementations, indicating a vibrant overall ecosystem.
* **Positive User and Developer Feedback**: Evidence that users can clearly understand the distinctions between implementations, choose the appropriate one for their needs, and that developers find the ecosystem supportive and conducive to building solutions. This includes feedback on documentation, branding clarity, and the perceived value of specialization.
* **Successful Launch and Sustained Viability of New Implementations**: The successful launch, adoption, and continued development of new, specialized implementations (e.g., Fabric-X) that demonstrably meet their stated goals (e.g., performance, specific features) and attract their own user/developer communities.
* **Management of Drawbacks**: Effective mitigation of the identified drawbacks, particularly user confusion and resource fragmentation, through proactive governance and community efforts.

Each individual Fabric implementation (e.g., Fabric Classic, Fabric-X) will, of course, continue to require its own comprehensive software testing strategy, including unit, integration, performance, security, and end-to-end tests relevant to its specific architecture, features, and goals. This RFC does not alter that requirement for individual codebases.

# Dependencies
[dependencies]: #dependencies

The successful implementation of this Fabric Ecosystem Coexistence Model has the following key dependencies:

1.  **Community Buy-in and Consensus**: Broad acceptance and support from the Hyperledger Fabric community (maintainers, contributors, users, and organizations) for this evolutionary direction. This RFC itself is the primary mechanism for achieving this consensus.
2.  **Formal Amendment of the Hyperledger Fabric Technical Charter**: The acceptance of this Ecosystem Coexistence Model RFC **directly proposes the core principles that must guide the evolution of the Hyperledger Fabric Technical Steering Committee (TSC) and its Technical Charter**. These amendments are critical for effectively stewarding an ecosystem of multiple coexisting implementations:
    * **Principle 1: Expanded Mission and Scope for Ecosystem Governance:** The TSC's mission and scope, as would be defined in an amended Charter, must explicitly encompass the governance of a multi-implementation Hyperledger Fabric ecosystem. This includes the TSC's authority and responsibility for overseeing the lifecycle (recognition, ongoing adherence to principles, potential sunsetting) of distinct Fabric implementations.
    * **Principle 2: Defined TSC Responsibilities for Ecosystem Management:** An amended Charter (modifying current Section II.C, "Responsibilities of the TSC") must clearly delineate the TSC's responsibilities in this expanded role. These responsibilities shall include, at a minimum:
        * Establishing, maintaining, and evolving the 'Mandatory Core Fabric Principles'.
        * Defining and managing the formal process (including criteria for justification, technical review, and community consensus) for proposing, reviewing, recognizing, and potentially sunsetting distinct Fabric implementations.
        * Developing and enforcing ecosystem-wide branding, naming conventions, and communication strategies.
        * Serving as the ultimate point of mediation and resolution for ecosystem-level conflicts.
    * **Principle 3: Clarified Roles and Responsibilities (TSC vs. Implementation-Specific Maintainers):** An amended Charter (modifying current Section III, "Maintainers") must clearly distinguish between the overarching strategic and governance responsibilities of the TSC (acting as Ecosystem Stewards, potentially as an "Expanded Fabric TSC" as described in the "Proposed Governance Model" section) and the technical, operational, and day-to-day management responsibilities of the Implementation-Specific Maintainers for their respective codebases and communities.
    * **Principle 4: Adapted Governance Procedures for Ecosystem Decisions:** TSC procedures, including voting mechanisms (as per current Charter Section II.D), must be adapted within an amended Charter to address how decisions impacting the entire ecosystem (e.g., recognition of a new implementation, changes to core principles) are made, incorporating mechanisms such as those proposed in the "Proposed Governance Model" section of this RFC.
    * **Path Forward for Charter Amendment:** Following the acceptance of this Coexistence Model RFC, it is proposed that a dedicated working group, chartered by and including members of the current TSC, be formed. This working group will be tasked with drafting specific textual amendments to the Hyperledger Fabric Technical Charter based on these principles. This draft will then be presented to the broader Fabric community for review and feedback, and subsequently to the TSC for formal approval via the established charter amendment process (as per current Charter Section V).
3.  **Commitment from Proposers of New Implementations**: Teams proposing new implementations (like Fabric-X) must commit to:
    * Adhering to the core Fabric principles as defined and evolved by the TSC.
    * Providing thorough justification and a detailed technical RFC for their specific implementation, addressing the criteria set forth by the TSC.
    * Actively participating in the Fabric community and the evolved governance processes established under the amended Charter.
    * Clearly documenting their implementation's features, differences from other Fabric implementations, and intended use cases.
4.  **Future RFCs for Specific Implementations**: The actual creation and official recognition of any new coexisting implementation (e.g., Fabric-X, Fabric-Y) will be contingent upon:
    * The acceptance of this foundational Ecosystem Coexistence Model RFC.
    * The subsequent drafting and approval of the necessary amendments to the Fabric Technical Charter.
    * The submission and approval by the TSC (operating under its amended charter) of a separate, detailed technical RFC for that specific implementation.

# Unresolved questions
[unresolved]: #unresolved-questions

While this RFC proposes the overarching framework and the core principles for governance and charter evolution, the precise textual crafting of the Charter amendments and detailed operational policies for the Expanded TSC will be the work of the subsequently formed working group and the TSC itself. Key areas that the charter amendment process will need to formalize, based on the principles in this RFC, include:

1.  **Branding, Naming, and Communication Policies**: Establishing the definitive naming conventions and detailed communication strategies to guide users and differentiate implementations.
2.  **Cross-Implementation Collaboration and Resource Management Protocols**: Developing formal mechanisms for collaboration and clear policies for shared resource allocation.

This RFC provides the mandate and foundational principles for these subsequent detailed governance definitions and charter amendments.
