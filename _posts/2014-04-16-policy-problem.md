---
title:				On Policy in the Data Center - The policy problem
Project:			Congress
Author:				Tim Hinrichs and Scott Lowe
Contributors:       Martin Casado, Mike Dvorkin, Peter Balland, and Dennis Moreau
Affiliation:		VMware, Inc.
Date:				2014-05-08
Keywords:			Congress, OpenStack, Policy, Blog
Quotes Language:	english
layout:             post
---

Fully automated IT provisioning and management is considered by many to be the ultimate nirvana---people log into a self-service portal, ask for resources (compute, networking, storage, and others), and within minutes those resources are up and running. No longer are the people who use resources waiting on the people who are responsible for allocating and maintaining them. And, according to the accepted definitions of cloud computing (for example, the [NIST definition in SP800-145][1]), self-service provisioning is a key tenet of cloud computing.

However, fully automated IT management is a double-edged sword. While having people on the critical path for IT management was time-consuming, it provided an opportunity to ensure that those resources were managed sensibly and in a way that was consistent with how the business said they ought to be managed. In other words, having people on the critical path enabled IT resources to be managed according to *business policy*. We cannot simply remove those people without also adding a way of ensuring that IT resources obey business policy---without introducing a way of ensuring that IT resources retain the same level of policy compliance.

While many tools today (e.g. application lifecycle-management, templates, blueprints) claim they bring the data center into compliance with a given policy, there are often hidden assumptions underlying those claims. Before we can hope to understand how effective those tools are, we need to understand the problem of policy and policy compliance, which is the focus of this post. Future posts will begin laying out the space of policy compliance solutions and evaluating how existing policy efforts fit into that space.

## The Challenge of Policy Compliance

Policy compliance is challenging for several reasons. A business policy is often a combination of policies from many different sources. It's something in the heads of IT operators ("This link is flakey, so route network traffic around it"). It's written in English as organizational mandates ("We never put applications from different business units on the same production network"). It's something expressed in contracts between organizations ("Our preferred partners are always given solid-state storage"). It's something found in governmental legislation ("Data from Singapore's citizens can never leave the geographic borders of Singapore").

To complicate matters, all of those different sources of policy have competing objectives, cross multiple levels of system abstraction, are often mutually inconsistent, and are constantly changing. Every time an individual policy changes, significant work may be required to understand the repercussions of those changes on the IT resources. Furthermore, not all policy is created equal. Sometimes it is permissible (or even inevitable) that some policies are violated temporarily ("Ensure every operating system always has the latest security patches installed"). The prospect of policy violations leads to the problem of making choices based on the differing penalties for those violations and how difficult they are to rectify. For example, a violation of the policy ensuring up-to-date operating systems is easy to remedy (install the latest patch), but a violation of a self-stated privacy policy for protecting customer information may cause a public relations storm, which is much harder to remedy.

Despite its complexity, policy compliance is already being addressed today. People take high-level laws, rules, and regulations, and translate them into checklists that describe how the IT infrastructure must be architected, how the software applications running on that infrastructure must be written, and how software applications must be deployed on that infrastructure. Another group of people translate those checklists into configuration parameters for individual components of the infrastructure (e.g. servers, networks, storage) and into functional or non-functional software requirements for the applications deployed on that infrastructure. Network policy is implemented as a myriad of switch, router, and firewall configurations. Server policy is implemented by configuration management systems like Puppet/Chef/CFEngine, keeping systems up to date with the latest security patches, and ensuring that known vulnerabilities are mitigated with other mechanisms such as firewalls. Application developers write special-purpose code to address policy.

Hopefully it is clear that policy compliance is a hard, hard problem. Some of the difficulties are unavoidable. People will always need to deal with the ambiguity and contradictions of laws, rules, and regulations. People will always need to translate those laws into the language of infrastructure and applications. People will always have the job of auditing to ensure a business is in compliance with the law.

## Automating Policy Compliance

In this post, we focus on the aspects of the policy compliance problem that we believe are amenable to automation. We use the term *IT policy* from this point forward to refer to the high-level laws, rules, and regulations *after* they have been translated into the language of infrastructure and applications. This aspect of policy compliance is important because we believe we can automate much of it, and because there are numerous problems with how it is addressed today. For example:

1. An IT policy is often written using fairly high-level concepts, e.g. the information applications manipulate, how applications are allowed to communicate, what their performance must be, and how they must be secured. The onus for translating these concepts into infrastructure configuration and application code is left to infrastructure operators and application developers. That translation process is error-prone.

2. The results are often brittle. Moving a server or application from one network to another could cause a policy violation. A new application with different infrastructure requirements could force a network operator to remember why the configuration was the way it was, and to change it in a way that satisfies the IT policy for both the original and the new applications simultaneously.

3. When an auditor comes along to assess policy compliance, she must analyze the plethora of configurations, application code, architecture choices, and deployment options in the current system in order to understand what the IT policy that is being enforced actually is. Because this is so difficult, auditors typically use checklists that give an approximate understanding of compliance.

4. Whenever the IT policy changes (e.g. because new legislation is passed), people must identify the infrastructure components and applications causing (what are suddenly) policy violations and rework them.

## Key Components of Policy Compliance Automation

We believe that the same tools and technologies that helped automate IT management bring the promise of better, easier, and faster IT policy compliance through policy-aware systems. A policy-aware system has the potential to detect policy violations in the cloud and reconfigure automatically. A policy-aware system has the potential to identify situations where no matter what we do, there is no way to make a proposed change without violating policy. A policy-aware system has the potential to be told about policy updates and react accordingly. A policy-aware system has the potential to maintain enough state about the cloud to identify existing and potential policy violations and either prevent them from occurring or take corrective action.

To enable policy-aware systems, we need at least two things.

1. We must communicate the IT policy to the system in a language it can understand. English won't work; it is ambiguous and thus would require the computer to do the jobs of lawyers. Router and firewall configurations won't work because they are too low level---the true intent of the policy is lost in the details. We need a language that can unambiguously describe both operational and infrastructure policies using concepts similar to those we would use if writing the policy in English.

2. We need a policy engine that understands that language, can act to bring IT resources into compliance, can interoperate with the ecosystem of software services in a cloud or across multiple clouds, and can help administrators and auditors understand where policy violations are and how to correct them.

In a future post, we'll examine these key components in a bit more detail and discuss potential solutions for filling these roles. At that time, we'll also discuss how recent developments like Group Policy for OpenDaylight and OpenStack Neutron fit into this landscape.

## One More Thing: Openness

There is one more thing that is important for automated policy compliance: openness. As the "highest" layer in a fully-automated IT environment, policy represents the final layer of potential control or "lock-in." As such, we believe it is critically important that an automated policy compliance solution not fall under the control of any one vendor. An automated policy compliance solution that offers cloud interoperability needs to be developed out in the open, with open governance, open collaboration, and open code access. By leveraging highly successful open source methodologies and approaches, this becomes possible.

## Wrapping It Up

As we wrap this up and prepare for the next installation in this series, here are some key points to remember:

* IT policy compliance in cloud environments is critical. IT resource management *cannot* be fully and safely automated without policy compliance.
* Manually enforcing IT policy isn't sustainable in today's fast-moving and highly fluid IT environments. An automated IT policy compliance solution is needed.
* An automated IT policy compliance solution needs the right policy language---one that can express unambiguous concepts in a way that is consumable by both humans and software alike---as well as a policy engine that can understand this policy language and interact with an ecosystem of cloud services to enforce policy compliance.
* Any policy compliance solution needs to be developed using an open source model, with open governance, open collaboration, and open code access.

In the next part of this series on automated policy compliance in cloud environments, we'll take a deeper dive into looking at policy languages and policy engines that meet the needs described here.



[1]: http://csrc.nist.gov/publications/nistpubs/800-145/SP800-145.pdf
