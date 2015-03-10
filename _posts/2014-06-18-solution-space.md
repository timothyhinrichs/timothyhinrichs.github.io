---
title:              On Policy in the Data Center - The solution space
Project:            Congress
Author:             Tim Hinrichs and Scott Lowe
Contributors:
Date:               2014-09-16
Keywords:           Congress, OpenStack, Policy
Quotes Language:    english
layout:             post
---

In the [first part of this series][1] we described the policy problem: ensuring that the data center obeys the real-world rules and regulations that are pertinent to that data center. In this post, we look at the range of possible solutions by identifying some the key features that are important for any solution to the policy problem.  Those key features correspond to the following four questions, which we use to structure our discussion.

1. What are the policy sources a policy system must accommodate?
2. How do those sources express the desired policy to the system?
3. How does the policy system interact with data center services?
4. What can the policy system do once it has the policy?

Let's take a look at each of these questions one at a time.

## Policy Sources: The origins of policy

Let's start by digging deeper into an idea we touched on in the first post when describing the challenge of policy compliance: the sources of policy. While we sometimes talk about there being a single policy for a data center, the reality is that there are really many different policies that govern a data center. Each of these policies may have a different source or origin. Here are some examples of different policy sources:

* Application developers may write a separate policy for each application describing what that application expects from the infrastructure (such as high availability, elasticity/auto-scaling, connectivity, or a specific deployment location).
* The cloud operator may have a policy that describes how applications relate to one another. This policy might specify that applications from different business units must be deployed on different production networks, for example.
* Security and compliance teams might have policies that dictate specific firewall rules for web servers, encryption for PCI-compliant workloads, or OS patching guidelines.
* Different policies may be focused on different functionality within the data center. There might be a deployment policy, a billing policy, a security policy, a backup policy, and a decommissioning policy.
* Policies may be written at different levels of abstraction, e.g. applications versus virtual machines (VMs), networks versus routers, storage versus disks.
* Different policies might exist for different policy operations (monitoring, enforcing, and auditing are three examples that we will discuss later in this post).

The idea of multiple sources of policy naturally leads us to the presence of multiple policies. This is an interesting idea, because these multiple policies can interact with each other in many different ways. A policy describing where an application is to be deployed might also implicitly describe where VMs are to be deployed. A cloud operator's policy might require an application to be deployed on network A or B, and an application policy requiring high availability might mean it must be deployed on network B or C; taken together, this means the application can only be deployed on network B. An auditing policy that requires knowing provenance for data when applied to an application that supports a high transaction rate might require solid state storage to meet performance requirements.

Taking this a step further, it may be unclear how these policies should interact. If the backup policy says to have 3 copies of data, and an auditing policy requires keeping track of where the data originated, do we need 3 copies of that provenance information? Conflicts are another example. If the application's policy implies networks A or B, and the cloud operator's policy implies networks C or D, then there is no way to deploy that application so that both policies are satisfied simultaneously.

There are a couple key takeaways from this discussion. First, a policy system must deal with multiple policy sources. Second, a policy system must deal with the presence of multiple policies, and how those policies can or should interact with one another.

## Expressing Policy: Policy languages

Any discussion of policy systems has to deal with the subject of policy languages. An intuitive, easy-to-use syntax is critically important for eventual adoption, but here we focus on more semantic issues: how domain-specific is the language? How expressive is the language? What general-purpose policy features belong to the language?

A language is a domain specific language (DSL) if it includes primitives useful for policy in one domain but not another. For example, a policy language for networking might include primitives for the source and destination IP addresses of a network packet. A DSL for compute might include primitives for the amount of memory or disk space on a server. An application-oriented DSL might include elasticity primitives so that how different parts of the application grow and shrink can be the subject of policy.

Different parts of a policy language can be domain-specific:

* _Namespace:_ The objects over which we declare policy can be domain-specific. For example, a networking DSL might define policy about packets, ports, switches, and routers.
* _Condition:_ Policy languages typically have if-then constructs, and the "if" part of those constructs can include domain-specific tests, such as the source and destination IP addresses on a network packet.
* _Consequent:_ The "then" component of if-then constructs can also be domain-specific.  For networking, this might include allowing/dropping a packet, sending a packet through a firewall and then a load balancer, or ensuring quality of service (QoS).

Independent of domain-specific constructs, a language has a fundamental limitation on its expressiveness (its "raw expressiveness"). Language A is more expressive than language B if every policy for B can be translated into a policy for A but not vice versa. For example, if language A supports the logical connectives AND/OR/NOT, and language B is the same except it only supports AND/OR, then A is more expressive than B. However, it can be the case that language A supports AND/OR/NOT, language B supports AND/NOT, and yet the two languages are equally expressive (because OR can be simulated with AND/NOT).

It may seem that more expressiveness is necessarily better because a more expressive language makes it easier for users to write policies. Unfortunately, the more expressive a language, the harder it is to implement. By "harder to implement" we don't mean it's harder to get a 30% speed improvement through careful engineering; rather, we mean that it is provably impossible to make the implementation of sufficiently expressive languages run in less than exponential time. In short, every policy language chooses how to balance the policy writers' need for expressiveness and the policy system's need for implementability.

On top of domain-specificity and raw expressiveness, different policy languages support different features. For example, can we say that some policy statements are "hard" (can never be violated) while other statements are "soft" (can be violated if the only way to not violate is to violate a hard constraint). More generally, can we assign priorities to policy statements? Is there native support for exceptions to policy rules (maybe a cloud owner wants to manually make an exception for a violation so that auditing reflects why that violation was less severe than it may have seemed). Does the language have policy modules and enable people to describe how to combine those modules to produce a new policy? While such features might not impact domain-specificity or raw expressiveness, they have a large impact on how easy the policy language---and therefore the system using that language---is to use.

The key takeaway here is that the policy language has a significant impact on the policy system, so the choice of policy language is a critical one.

## Policy Interaction: Integrating with data center services

A policy system by itself is useless; to have value, the policy system must interact and integrate with other data center or cloud services. By "data center service" or "cloud service" we mean basically anything that has an API, e.g. OpenStack components like Neutron, routers, servers, processes, files, databases, antivirus, intrusion detection, inventory management.  Read-only API calls enable a policy system to _see_ what is happening; read/write API calls enable a policy system to _control_ what is happening.

Since a policy system's ability to do something useful with a policy (like prevent violations, correct violations, or monitor for violations) is directly related to what the service can see and do in the data center, it's crucial to understand how well a policy system works with the services in a data center. If two policy systems are equivalent except that one works with a broader range of data center services than the other, the one with a broader selection of data center services has the ability to see and do more; thus, it's better able to see and do things to help the data center obey policy.

One type of data center service is especially noteworthy: the policy-aware service. Such services understand policy natively. They may have an API that includes "insert policy statement" and "delete policy statement". Such services are especially useful in that they can potentially help the data center obey certain kinds of sub-policies. Distributing the work makes a policy system more robust, more reliable, and better performing.

The key point to remember here is that a policy system's "power" (knowledge of and control over what's happening in a data center or cloud environment) is driven by the nature of its interaction with the services running in that data center.

## Policy Action: Taking action based on policy

Having looked at three key aspects of a policy system---supporting multiple sources of policies and multiple policies, using a policy language that balances expressiveness with implementability, and providing the appropriate depth and breadth of integration with necessary data center services---we now come to a discussion of what the policy system does (or can do) once it knows what policy (or group of policies) is pertinent to the data center. It's compelling to think about the utility of policy in terms of the future, the present, and the past. We want the data center to obey the policy at all points in time, and there are different mechanisms for doing that.

* Auditing: We cannot change the past but we can record how the data center behaved, what the policy was, and therefore what the violations were.
* Monitoring: The present is equally impossible to change (by the time we act, that moment in time will have become the past), but we can identify violations, help people understand them, and gather information about how to reduce violations in the future.
* Enforcement: We can change the future behavior of the data center by taking action. Enforcement can attempt to prevent violations before they occur ("proactive enforcement") or correct violations after the fact ("reactive enforcement"). This is the most challenging of the three because it requires choosing and executing actions that affect the natural state of the data center.

The potential for any policy system to carry out these three functions depends crucially on two things: the policy itself (a function of how well the system supports multiple policies as well as the system's choice of policy language) and the controls the policy system has over the data center (driven by the policy system's interaction with and integration into the surrounding data center services). The combination of these two things impose hard limits on how well **any** policy system is able to audit, monitor, and enforce policy.

While we would rather prevent violations than correct them, it's sometimes impossible to do so. For example, we cannot prevent violations in a policy that requires server operating system (OS) instances to always have the latest patches. Why? As soon as Microsoft, Apple, or Red Hat releases a new security patch, the policy is immediately violated. The point of this kind of policy is that the policy system recognizes the violation and applies the patch to the vulnerable systems (to correct the violation).  The key takeaway from this example is that _preventing violations requires the policy system is on the critical path for any action that might violate policy._ Violations can only be prevented if such enforcement points are available.

Similarly, it's not always possible to correct violations. Consider a policy that says the load on a particular web server should never exceed 10,000 requests per second. If the requests to that server become high enough (even with load balancing), there may be no way reduce the load once it reaches 10,001 requests per second. The data center cannot control what web sites people in the real world access through their browsers. In this case, the key takeaway is that _correcting violations requires there be actions available to the policy system to counteract the causes of those violations._

Even policy monitoring has limitations. A policy dictating application deployment to particular data centers based on the users of that application assumes readily available information about where applications are deployed and the users of those applications. A web application that does not expose information about its users ensures even monitoring this policy is impossible. The key takeaway here is that _monitoring a policy requires that the appropriate information is available to the policy system._ Further, if we cannot monitor a policy, we also cannot audit that policy.

In short, every policy system has limitations. These limitations might be on what the policy system knows about the data center, or these limitations might be on what control it has over the data center. These limitations influence whether any given policy can be audited, monitored, and enforced. Moreover, these limitations can change as the data center changes. As new services (hardware or software) are installed or existing services are upgraded in the data center, new functionality becomes available, and a policy system may have additional power (fewer limitations). When old services are removed, the policy system may have less power (more limitations).

These limitations give us ceilings on how successful any policy system might be in terms of auditing, monitoring, and enforcing policy. It is therefore useful to compare policy system **designs** in terms of how close to those ceilings they can get. Of course, the true test is in terms of actual implementation, not design, and a comparison of systems in terms of what policies they can audit, monitor, and enforce at scale is incredibly valuable. However, we must be careful not to condemn systems for not solving unsolvable problems.

## Wrapping Up

This blog post has focused on laying out the range of possible solutions to the policy problem in the data center. In summary, here are some key points:

* Policies come from many different sources and interact in many different ways. Ideally the data center would obey all those policies simultaneously, but in practice we expect the policies to conflict. A solution to the policy problem must address the issues surrounding multiple policies.

* A policy language can be categorized in terms of its domain-specificity, its raw expressiveness, and the features it supports. Every solution must balance these three to meet the need for the policy writer to express policy and the need of the policy system to audit, monitor, and enforce policy.

* Every solution must interact with the ecosystem of data center services within the data center. The richer the ecosystem a solution can leverage, the more successful it can be.

* Once a policy system has a policy, it can audit, monitor, and enforce that policy. A solution to the policy problem is more or less successful at these functions depending on the policy and the data center.

In the next blog post, we will look at proposed policy systems like the OpenStack Group-Based Policy and the Congress project, and explain how they fit into this solution space.



[1]: /2014/04/16/policy-problem.html