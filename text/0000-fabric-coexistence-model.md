---
layout: default
title: Fabric Ecosystem Coexistence Model
nav_order: 3
---

- Feature Name: Fabric Ecosystem Coexistence Model
- Start Date: 2025-05-20
- RFC PR: (leave this empty)
- Fabric Component: Ecosystem Architecture, Governance, and Core Principles
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a strategic evolution for the Hyperledger Fabric ecosystem, transitioning from a "one-size-fits-all" architecture to a **Fabric Ecosystem Coexistence Model**. This model formally allows multiple, distinct Fabric-derived **implementations** (e.g., the current "Fabric Classic," a high-performance "Fabric-X" for CBDCs, or a future "Fabric-Y" for IoT) to be developed and recognized under the unified Hyperledger Fabric project umbrella. While all such implementations **must mandatorily adhere to core, non-negotiable Fabric principles**—most notably the **execute-order-validate architectural pattern**, the **Membership Service Provider (MSP) framework for identity**, and the **Key-Value-Version data model**—they can be tailored with specific optimizations, transaction formats, programming models, and governance nuances to best serve distinct domains or high-value use cases with unique requirements. This approach aims to enhance Fabric's adaptability, foster innovation, and broaden its applicability to next-generation DLT challenges.

# Motivation
[motivation]: #motivation

Hyperledger Fabric's initial design provided a robust, general-purpose DLT platform. However, as the blockchain landscape matures and enterprise adoption demands more specialized solutions, a single, monolithic architecture faces inherent limitations in optimally serving an increasingly diverse set of requirements. While core Fabric concepts like the execute-order-validate flow, MSPs, and its data model are highly reusable and proven, their specific instantiations and the surrounding platform features often need domain-specific adaptations to achieve peak effectiveness or meet unique operational needs (e.g., extreme throughput, novel privacy mechanisms, or distinct programming paradigms).

Attempting to integrate all conceivable specializations into a single codebase risks making that codebase overly complex, difficult to maintain, and potentially suboptimal for all use cases. The "one-size-fits-all" approach, while valuable initially, can become a bottleneck to innovation and adoption in highly specialized or demanding new fields.

The primary motivations for this Ecosystem Coexistence Model are:

1.  **Strategic Adaptability & Specialization**: To formally enable the creation of distinct Fabric-based implementations optimized for specific domains (e.g., high-throughput Central Bank Digital Currencies, specialized digital asset exchanges, or unique IoT data integrity solutions) when their requirements are demonstrably and prohibitively difficult or architecturally impractical to implement within existing Fabric implementations without significant compromises to their design, performance, or established user base.
2.  **Ecosystem Revival & Sustained Growth**: To invigorate the Fabric ecosystem by creating clear pathways for innovation that address new, active use cases as blockchain technology evolves. This model acknowledges that diverse problems often require diverse, yet related, solutions, thereby expanding Fabric's overall relevance and market reach.
3.  **Preventing Stagnation and Fragmentation**: Without a flexible, officially recognized model for specialized implementations, the core Fabric project risks becoming less adaptable to emerging demands. This could lead to either stagnation (if new needs are unmet) or uncontrolled fragmentation (if valuable innovations occur outside the recognized Fabric ecosystem, leading to a loss of shared identity and community).
4.  **User Choice with Conceptual Consistency**: To offer users and organizations a family of implementations built upon familiar and trusted Fabric principles. This allows them to select the specific implementation best suited to their particular use case, benefiting from a common conceptual understanding (e.g., of identity, endorsement, and the EOV flow) while accessing specialized capabilities.
5.  **Leveraging and Reinforcing Core Strengths**: To ensure that all implementations within the ecosystem continue to leverage and reinforce the robust foundational concepts of Fabric (especially execute-order-validate, MSP, KVV data model, and endorsement policy DSL), thereby maintaining a cohesive, high-quality, and trusted brand.

The expected outcome is a more vibrant, adaptable, and resilient Hyperledger Fabric ecosystem, capable of supporting a wider and more demanding range of current and future enterprise blockchain applications, solidifying its position as a leading DLT platform.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Imagine the Hyperledger Fabric ecosystem evolving into a well-defined **constellation of recognized implementations**. Each star in this constellation represents a specific Fabric implementation, sharing common DNA (the core Fabric principles) but shining with its own unique characteristics tailored for certain tasks. Instead of a single "Fabric," an organization might choose from:

* **Fabric (Classic)**: The established, general-purpose Fabric implementation, continuing to serve its wide array of existing production use cases with its known feature set and operational model.
* **Fabric-X (Example)**: A new implementation (as proposed in a separate, detailed technical RFC) specifically designed and optimized for extreme high-throughput digital asset use cases like CBDCs. It would mandatorily share the Execute-Order-Validate model, MSP framework, KVV data model, and endorsement policy concepts with Fabric Classic. However, it might feature a different transaction structure, an evolved programming model (e.g., 'Flows' instead of traditional chaincode), omit certain features not critical for its target domain (e.g., initially, channels or PDCs as currently known), and possess unique performance optimizations and a microservice-based architecture.
* **Fabric-Y (Future Example)**: Another potential future implementation, perhaps tailored for IoT data integrity with specialized consensus mechanisms or lightweight client requirements, again adhering to the core Fabric principles but differentiated for its specific domain.

**How Fabric Developers and Users Should Think About This:**

When embarking on a new project, developers and architects would first analyze their core requirements (performance, specific features, operational environment, programming model preferences). Based on this, they would select the Fabric ecosystem implementation that best aligns with those needs.

* **Conceptual Familiarity:** Developers familiar with one Fabric implementation will find many core concepts (identity via MSP, endorsement policies, the EOV lifecycle, KVV state) immediately recognizable and transferable when exploring another implementation within the ecosystem. This shared foundation is a key strength.
* **Expected Differences:** However, they should also expect and plan for differences in:
    * **Specific APIs and SDKs:** Tailored to the programming model and features of that implementation (e.g., Fabric-X might use an SDK for 'Flows' rather than a chaincode shim).
    * **Available Features and Configurations:** Some features (like channels or PDCs) might be implemented differently or not be present in all implementations, especially in their initial versions if they are highly specialized.
    * **Performance Characteristics:** Implementations will have different performance profiles based on their architecture and optimizations.
    * **Governance Models and Release Cadences:** Each recognized implementation will have its own set of maintainers and a release cycle appropriate for its community and stability requirements, though all operating under the overarching Fabric TSC.
    * **Operational Tooling and Deployment Models:** A microservice-based Fabric-X will have different deployment and operational needs than a monolithic Fabric Classic.

**Setting Expectations and Ensuring Clarity:**

It is paramount to set clear expectations to avoid user confusion. This model proposes **distinct, recognized implementations built on shared foundational principles, not merely configurable modules of a single entity.**

* **Clear Documentation & Branding:** Comprehensive documentation for the overall Fabric ecosystem will clearly delineate:
    * The common core principles shared by all implementations.
    * The specific intended use cases, features, architectural highlights, and operational differences of each recognized implementation (e.g., Fabric Classic, Fabric-X).
    * Guidelines on how to choose the most appropriate implementation for a given project.
    * Branding guidelines (e.g., "Hyperledger Fabric Classic," "Hyperledger Fabric-X") will be established to ensure clarity and maintain the integrity of the Fabric brand.
* **Migration Paths:** Initially, direct, automated migration paths between distinct implementations (e.g., from Fabric Classic to Fabric-X) will **not be guaranteed** due to potentially significant architectural and feature differences. The focus of new implementations like Fabric-X is often to enable use cases that are infeasible with existing ones. However, for specific, highly demanded transitions, the community for a given implementation might develop tool-assisted migration strategies (e.g., state snapshotting and transformation) as a future effort, subject to their own RFCs.
* **Burden of Proof for New Implementations:** This RFC proposes the *framework and policy* to allow such a model. The actual creation, detailing, and acceptance of a specific new implementation (like Fabric-X or Fabric-Y) would be subject to its own, separate, rigorous technical RFC. This subsequent RFC must clearly articulate its adherence to core Fabric principles and provide compelling justification (see "Reference-level explanation") for why its specific requirements cannot be reasonably met by extending existing implementations.

**Impact on Security and Administration:**

Each coexisting implementation (Fabric Classic, Fabric-X, etc.) would have its own dedicated set of maintainers and a defined release cycle. Consequently:

* **Security considerations, patches, and updates** would be managed by the maintainers specific to each implementation, according to its architecture and codebase.
* **Administrative procedures and operational best practices** would also be specific to each implementation.
* The **overarching Hyperledger Fabric Technical Steering Committee (TSC)** would provide top-level governance, ensure adherence to ecosystem-wide principles (like the mandatory core concepts), manage the process for recognizing new implementations, and facilitate cross-implementation collaboration where beneficial. The TSC's role would evolve to steward the entire constellation.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC introduces a policy and operational framework change for the Hyperledger Fabric project, enabling a new working model for the ecosystem. It does not propose specific code modifications to any single existing Fabric component but rather defines the principles and processes for recognizing and governing multiple, coexisting Fabric implementations.

The core technical and governance implications of this model are:

1.  **Mandatory Adherence to Core Fabric Principles:** Any new implementation seeking recognition under the Fabric Ecosystem Coexistence Model **must** demonstrably implement and adhere to a defined set of non-negotiable core Hyperledger Fabric principles. These initially include:
    * The **Execute-Order-Validate (EOV)** architectural pattern for transaction lifecycle.
    * The **Membership Service Provider (MSP)** framework for identity management and permissioning.
    * The **Key-Value-Version (KVV)** data model for ledger state.
    * The use of **Endorsement Policies** (utilizing Fabric's policy DSL) to govern transaction validity.
    * The concept of **Read-Write Sets (RWSets)** for transaction proposals and **Multi-Version Concurrency Control (MVCC)** for validation.
    This list of core principles can be refined over time by the Fabric TSC. Implementations may extend or optimize how these are realized (e.g., Fabric-X using a distributed DB for KVV state but preserving the model) but cannot fundamentally deviate from them.

2.  **Clear Justification for New Implementations:** A proposal for a new coexisting implementation must provide a comprehensive justification, including:
    * **Demonstration of Adherence:** Explicitly detailing how it meets all mandatory core Fabric principles.
    * **Statement of Purpose and Target Use Cases:** Clearly defining the problem space it aims to address.
    * **Evidence of Necessity:** Providing compelling evidence (e.g., architectural analysis, performance benchmarks of attempted adaptations on existing implementations, or proof-of-concept results) that its target use case has requirements that are **prohibitively difficult, architecturally infeasible, or would lead to unacceptable compromises (e.g., to performance, security, or complexity) if implemented by modifying existing Fabric implementations.** The burden of proof lies with the proposers of the new implementation.
    * **Architectural Overview:** Describing its high-level architecture and key differentiating features.
    * **Proposed Governance:** Outlining its proposed initial maintainers and how it will operate within the broader Fabric governance structure.

3.  **No Guaranteed Cross-Implementation Migration Paths (Initially):**
    * Due to the specialized nature of different implementations, there will be **no default or guaranteed automated migration paths** for applications, data, or assets directly from one coexisting implementation (e.g., Fabric Classic) to another (e.g., Fabric-X), or vice-versa. Each implementation is distinct and optimized for its specific purpose.
    * *Example*: An application built for Fabric Classic cannot be assumed to run on Fabric-X without significant rework (e.g., adapting from chaincode to 'Flows'), and ledger data may not be directly transferable due to differences in transaction structures or underlying storage optimizations.
    * *Future Possibility*: While direct, seamless migration paths are not an initial goal of this overarching model, specific communities for coexisting implementations *might* choose to develop tools, best practices, or specific bridges for transitioning applications or data between certain implementations if there's a strong, well-defined need. Such efforts would be community-driven and subject to their own review processes.

4.  **Distinct Governance, Maintenance, and Releases within an Overarching Framework:**
    * Each recognized implementation within the Fabric ecosystem (e.g., "Fabric Classic," "Fabric-X") will operate with:
        * A **dedicated set of maintainers** responsible for its codebase, security, and releases.
        * An **independent development lifecycle and release cadence** appropriate for its specific goals and community.
        * Its own set of **GitHub repositories** (e.g., `hyperledger/fabric`, `hyperledger/fabric-x-orderer`).
    * All implementations will fall under the **ultimate governance of the Hyperledger Fabric TSC**. The TSC will:
        * Define and maintain the criteria for recognizing new Fabric implementations.
        * Approve proposals for new coexisting implementations.
        * Oversee adherence to core principles across the ecosystem.
        * Manage branding and naming conventions.
        * Facilitate collaboration and address potential conflicts or resource contention between implementation communities.
        * The Fabric TSC charter will be amended (via a separate RFC) to reflect this expanded stewardship role.

**Process for Introducing a New Coexisting Implementation (e.g., Fabric-X):**

1.  **This RFC (Ecosystem Coexistence Model):** If accepted, it establishes the *possibility* and *framework* for having multiple recognized implementations.
2.  **Specific Implementation RFC:** Proposers of a new implementation (e.g., Fabric-X) would then submit a detailed technical RFC to the Fabric project. This RFC would need to:
    * Meet the "Clear Justification for New Implementations" criteria outlined above.
    * Describe its architecture, features, differences from other implementations, initial feature set, and development roadmap.
    * Propose its initial set of maintainers.
3.  **Community Review and TSC Approval:** The specific implementation RFC would undergo community review and require approval from the Fabric TSC, based on its technical merit, adherence to core principles, justification of necessity, and alignment with the ecosystem's strategic goals.
4.  **Repository Creation and Development:** Upon approval, dedicated repositories would be created under the Hyperledger Fabric GitHub organization, and development would proceed under the guidance of its designated maintainers.

This structured approach ensures that the Fabric ecosystem can evolve in a controlled, transparent, and principled manner.

# Drawbacks
[drawbacks]: #drawbacks

* **Potential User Confusion**: Despite best efforts, having multiple "Fabric" implementations could initially confuse users regarding which to choose, their specific capabilities, and their differences.
    * *Mitigation*: Strong, clear, centralized documentation; distinct branding and naming conventions (e.g., "Hyperledger Fabric Classic," "Hyperledger Fabric-X"); clear guidance on use-case suitability; active community support and education.
* **Fragmentation of Development Effort and Community Focus**: Developer attention, contributions, and community discussions could become spread across multiple implementations.
    * *Mitigation*: Fostering a sense of a unified Fabric *ecosystem* with shared core principles; encouraging cross-implementation learning and collaboration where appropriate; ensuring each implementation has a clear value proposition to attract a dedicated community; active stewardship by the Fabric TSC to promote overall ecosystem health.
* **Risk of Brand Dilution**: If the quality, security, or adherence to core Fabric principles varies significantly and negatively between implementations, it could dilute the overall "Hyperledger Fabric" brand.
    * *Mitigation*: Strict TSC oversight on the adherence to mandatory core principles for all recognized implementations; clear quality and security standards; transparent governance for each implementation.
* **Increased Governance Overhead**: Managing an ecosystem of multiple distinct implementations will introduce additional complexity for the overarching Fabric TSC.
    * *Mitigation*: A well-defined and amended Fabric TSC charter clearly outlining its role in stewarding the ecosystem, managing the lifecycle of implementations, and delegating appropriate autonomy to individual implementation maintainer groups.
* **Learning Curve for Users and Developers**: Users and developers may need to learn the nuances of different implementations if their needs span across them.
    * *Mitigation*: Emphasizing the shared core principles to provide a consistent conceptual foundation; excellent documentation highlighting differences and specific use cases; community training and educational materials.

# Rationale and alternatives
[alternatives]: #rationale-and-alternatives

**Why is this Ecosystem Coexistence Model the best approach?**

This federated, coexistence model, governed by an overarching TSC, is deemed the most effective for the long-term health and relevance of Hyperledger Fabric because:

1.  **Fosters Targeted Innovation for Critical Use Cases**: It allows specialized solutions (like a Fabric-X for >100k TPS) to be developed and optimized rapidly when their specific, demanding requirements are demonstrably too difficult or disruptive to integrate into existing, general-purpose Fabric implementations without significant compromise.
2.  **Addresses Diverse Market Needs Strategically**: It directly responds to the reality that "one-size-fits-all" is insufficient for the breadth of high-value, specialized blockchain use cases emerging. This allows the Fabric *ecosystem* to cater to a wider market.
3.  **Maintains Conceptual Cohesion and Trust**: Unlike completely independent forks, this model *mandates* adherence to core Fabric principles (EOV, MSP, KVV, etc.), providing a familiar conceptual foundation and preserving the trust associated with the Fabric brand. It's about *managed diversity*, not uncontrolled divergence.
4.  **Prevents Stagnation and Encourages Renewal**: It offers a clear, governed path for the Fabric ecosystem to evolve, adapt to new technological paradigms (like microservices for performance), and remain relevant, rather than risking obsolescence by not being able to address new, demanding requirements.
5.  **Preserves Stability of Existing Implementations**: New, potentially disruptive architectural explorations (like Fabric-X) can occur in dedicated repositories without destabilizing the existing Fabric Classic codebase, which supports many production systems.
6.  **Attracts Specialized Talent and Communities**: Distinct implementations focused on specific problem domains can attract developers and organizations with specialized expertise and interest in those areas, enriching the overall Fabric talent pool.

**Alternatives Considered:**

1.  **Continue with a Single, Monolithic, Highly Configurable Fabric Implementation**: Attempt to evolve the single Fabric codebase to cater to all needs.
    * *Rationale for not choosing*: This approach becomes exponentially complex to design, test, maintain, and configure. For use cases with fundamentally distinct architectural needs or extreme performance demands (e.g., Fabric-X's microservice architecture for >100k TPS), attempting to shoehorn these into a single, evolving monolith is often impractical, leads to a "lowest common denominator" feature set, risks compromising existing functionalities, or makes the primary implementation too unwieldy and difficult to manage for everyone. As has been observed, "one size fits all" has not served well for pushing performance boundaries.
2.  **Encourage Independent Forks (No Coexistence Model / No Official Recognition):** Allow any group to fork Fabric and develop independently, without any formal relationship or shared ecosystem branding under the Hyperledger Fabric project.
    * *Rationale for not choosing*: This would lead to greater fragmentation, loss of shared identity and trust, potential brand confusion if forks still use "Fabric" in their name without oversight, and make it harder for users to identify solutions based on Fabric's proven concepts. It also misses the opportunity for shared learning, potential (even if not initially guaranteed) cross-pollination of ideas, and leveraging the established Hyperledger Fabric governance and community structures.
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

* **Clarity, Adoption, and Effectiveness of the Evolved Governance Model**: Successful establishment and community adoption of clear, fair, and efficient governance processes (within the Fabric TSC) for proposing, evaluating, recognizing, and overseeing coexisting implementations. This includes well-defined criteria for demonstrating adherence to core Fabric principles and justifying the need for a new implementation.
* **Active Community Engagement and Contribution**: Healthy levels of participation, contribution, and discussion across the various recognized Fabric implementations, indicating a vibrant overall ecosystem.
* **Positive User and Developer Feedback**: Evidence that users can clearly understand the distinctions between implementations, choose the appropriate one for their needs, and that developers find the ecosystem supportive and conducive to building solutions. This includes feedback on documentation, branding clarity, and the perceived value of specialization.
* **Successful Launch and Sustained Viability of New Implementations**: The successful launch, adoption, and continued development of new, specialized implementations (e.g., Fabric-X) that demonstrably meet their stated goals (e.g., performance, specific features) and attract their own user/developer communities.
* **Management of Drawbacks**: Effective mitigation of the identified drawbacks, particularly user confusion and resource fragmentation, through proactive governance and community efforts.

Each individual Fabric implementation (e.g., Fabric Classic, Fabric-X) will, of course, continue to require its own comprehensive software testing strategy, including unit, integration, performance, security, and end-to-end tests relevant to its specific architecture, features, and goals. This RFC does not alter that requirement for individual codebases.

# Dependencies
[dependencies]: #dependencies

The successful implementation of this Fabric Ecosystem Coexistence Model has the following key dependencies:

1.  **Community Buy-in and Consensus**: Broad acceptance and support from the Hyperledger Fabric community (maintainers, contributors, users, and organizations) for this evolutionary direction. This RFC itself is the primary mechanism for achieving this consensus.
2.  **Evolution of Fabric TSC Governance**: The Hyperledger Fabric Technical Steering Committee (TSC) will need to adapt its charter, scope, and operational processes to effectively steward an ecosystem of multiple coexisting implementations. This will likely involve:
    * Defining and ratifying the precise list of "Mandatory Core Fabric Principles."
    * Establishing the formal process and detailed criteria for proposing and approving new coexisting implementations.
    * Developing branding and naming guidelines for different implementations.
    * Creating mechanisms for managing shared resources and resolving potential conflicts.
    * A separate PR on `hyperledger/fabric` detailing these proposed governance amendments will follow if this foundational Coexistence Model RFC is accepted.
3.  **Commitment from Proposers of New Implementations**: Teams proposing new implementations (like Fabric-X) must commit to:
    * Adhering to the core Fabric principles.
    * Providing thorough justification and a detailed technical RFC for their specific implementation.
    * Actively participating in the Fabric community and governance processes.
    * Clearly documenting their implementation's features, differences, and intended use cases.
4.  **Future RFCs for Specific Implementations**: The actual creation of any new coexisting implementation (e.g., Fabric-X, Fabric-Y) will be contingent upon the acceptance of this foundational Ecosystem Coexistence Model RFC, followed by the submission and approval of a separate, detailed technical RFC for that specific implementation.

# Unresolved questions
[unresolved]: #unresolved-questions

While this RFC proposes the overarching framework, several specific details will need to be defined and resolved through subsequent discussions, community consensus, and potentially further targeted RFCs, particularly concerning the evolution of the Fabric TSC's governance role. Key areas for future resolution include:

1.  **Precise Governance Model for the Fabric "Constellation"**:
    * How will voting rights within the TSC be structured concerning decisions that impact the entire ecosystem versus those specific to one implementation?
    * What is the process for escalating disputes or making binding decisions if consensus cannot be reached within an individual implementation's maintainer group on matters affecting core principles or ecosystem integrity?
3.  **Branding, Naming, and Communication Guidelines**:
    * What specific naming conventions will be used for recognized implementations (e.g., `Hyperledger Fabric Classic`, `Hyperledger Fabric-X`, `Hyperledger Fabric [Codename]`) to ensure clarity and prevent market confusion?
    * How will the Hyperledger Fabric website, documentation portals, and communication channels be structured to clearly present the different implementations and guide users?
4.  **Mechanisms for Cross-Implementation Collaboration and Shared Learning**:
    * While implementations may be distinct, are there forums or processes that could encourage sharing of non-differentiating innovations, security best practices, or common tooling where beneficial?
    * How can the ecosystem leverage shared learning without imposing undue homogenization?
6.  **Lifecycle Management of Implementations (including De-chartering/Sunsetting)**:
    * What is the process for an implementation to be officially recognized?
    * What are the criteria and process for an implementation to become "emeritus," be de-chartered, or be officially sunsetted if it becomes inactive, unmaintained, or no longer strategically relevant?

These questions are expected to be addressed through focused efforts led by the Fabric TSC, with active community participation, once this foundational Ecosystem Coexistence Model is accepted. This RFC aims to provide the mandate and direction for that subsequent work.
