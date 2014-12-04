---
Title:				On Policy in the Data Center - Comparing policy efforts
Project:			Congress
Author:				Tim Hinrichs, Scott Lowe
Affiliation:		VMware, Inc.
Date:				2014-09-16
Keywords:			Congress, OpenStack, Policy
Quotes Language:	english
---

In the first two parts of this blog series we discussed the problem of [policy in the data center][1] and [the features that differentiate solutions to that problem][2]. In this post, we give a high-level overview of several policy efforts within OpenStack.

Remember that a _policy_ is a description of how (some part of) the data center ought to behave, a _service_ is any component in the data center that has an API, and a _policy system_ is designed to manage some combination of past, present, and future policy violations (auditing, monitoring, and enforcement, respectively).

The overview of OpenStack policy efforts talks about the features we identified in part 2 of this blog series. To recap, those features are:

* Policy language: how expressive is the language, is the language restricted to certain domains, what features (e.g. exceptions) does it support?
* Policy sources: what are the sources of policy, how do different sources of policy interact, how are conflicts dealt with?
* Services: which other data center services can be leveraged and how?
* Actions: what does the system do once it is given a policy: monitor (identify violations), enforce (prevent or correct violations), audit (analyze past violations)?

The one thing you'll notice is that there are many different policy efforts within OpenStack. Perhaps surprisingly there is actually little redundancy because each effort addresses a different part of the overall policy problem: enabling users to describe their desires in a way that an OpenStack cloud can act on them. Additionally, as we will point out again later in the post, domain independent and domain specific policy efforts are highly complementary.

## Congress

We begin with [Congress][3], our own policy effort within OpenStack. Congress is a system purpose-built for managing policy in the data center. A Congress policy describes the desired behavior of the data center by dictating how all the services running in that data center are supposed to behave both individually and in tandem. In the current release Congress accepts a single policy for the entire data center, the idea being that the cloud administrators are jointly responsible for writing and maintaining that policy.

A Congress policy is domain independent and can describe the behavior of any collection of data center services. The cloud administrator can write a policy about networking, a policy about compute, or a policy that about networking, compute, storage, antivirus, organizational charts, inventory management systems, ActiveDirectory, and so on.

The recent [alpha release of Congress][18] supports monitoring violations in policy: comparing how the data center is actually behaving to how policy says the data center ought to behave and flagging mismatches. In the future, Congress will also support enforcement by having Congress itself execute API calls to change the behavior of the data center and/or pushing policy to other policy-aware services better positioned to enforce policy.

## [Neutron Group-Based Policy (GBP)] [4]

GBP, which is similar to the policy effort in [OpenDaylight][5], utilizes policy to manage networking. A policy describes how the network packets in the data center are supposed to behave. Each policy ("contract" in GBP terminology) describes which actions (such as allow, drop, reroute, or apply QoS) should be applied to which network packets based on packet header properties like port and protocol. Entities on the network (called "endpoints") are grouped and each group is assigned one or more policies. Groups are maintained outside the policy language by people or automated systems.

In GBP, policies can come from any number of people or agents. Conflicts can arise within a single policy or across several policies and are eliminated by a mechanism built into GBP (which is out of scope for this blog post).

The goal of GBP is to enforce policy directly. (Both monitoring and auditing are challenging in the networking domain because there are so many packets moving so quickly throughout the data center.) To do enforcement, GBP compiles policies down into existing Neutron primitives and creates logical networks, switches, and routers. When new policy statements are inserted, GBP does an incremental compilation: changing the Neutron primitives in such a way as to implement the new policy while minimally disrupting existing primitives.

## Swift Storage Policy

[Swift][8] is OpenStack's object storage service. As of version 2.0, released [July 2014][6], Swift supports [storage policies][7]. Each storage policy is attached to a virtual storage system, which is where Swift stores objects. Each policy assigns values to a number of built-in features of a storage system. At the time of writing, each policy dictates how many partitions the storage system has, how many replicas of each object it should maintain, and the minimum amount of time before a partition can be moved to a different physical location since the last time it was moved.

A user can create any number of virtual storage systems---and so can write any number of policies---but there are no conflicts between policies. If we put an object into a container with 2 replicas and the same object into another container with 3 replicas, it just means we are storing that object in two different virtual storage systems, which all told means we have 5 replicas.

Policy is enforced directly by Swift. Every time an object is written, Swift ensures the right number of replicas are created. Swift ensures not to move a partition before policy allows that partition to be moved.

## Smart Scheduler/SolverScheduler

The [Smart Scheduler/SolverScheduler][9] effort aims to provides an interface for using different constraint solvers to solve optimization problems for other projects, [Nova][9] in particular. One specific use case is for Network Functions Virtualization (see [here][10] and [here][11]) For example, Nova might ask where to place a new virtual machine to minimize the average number of VMs on each server. This effort utilizes domain-independent solvers (such as linear programming/arithmetic solvers) but applies them to solve domain-specific problems. The intention is to focus on enforcement.

## Nova Policy-Based Scheduling Module

The [Nova policy-based scheduling module][12] aims to schedule Nova resources per client, per cluster of resources, and per context (e.g. overload, time, etc.). A [proof of concept was presented][13] at the Demo Theater at OpenStack Juno Summit.

## Gantt

Gantt aims to provide scheduling as a service for other OpenStack components (see [here][14] and [here][15]). Previously, it was a subgroup of Nova and focused on scheduling virtual machines based on resource utilization. It includes plugin framework for making arbitrary metrics available to the scheduler.

## Heat Convergence engine

The [Heat Convergence engine][16] represents a shift toward a model for Heat where applications are deployed and managed by comparing the current state of the application to the desired state of the application and taking action to reduce the differences. Each desired state amounts to a policy describing a single application. Those policies do not interact, logically, and can draw upon any service in the data center. Heat policies are concerned mainly with corrective enforcement, though monitoring is also useful ("how far along is my application's deployment?").

## Summary

The key takeaway is that OpenStack has a growing ecosystem of policy-aware services. Most of them are domain-specific, meaning they are systems tailored to enforcing a particular kind of policy for a particular service, but a few are domain-independent, meaning that they will work for any kind of policy.

The strength of a domain-specific policy system is *enforcing* policies within its domain, but its weakness is that policies outside the domain are not expressible in the language. The strength of a domain-independent policy system is *expressing* policies for any and every domain, but its weakness is that monitoring/enforcing/auditing those policies can be challenging.

For policy to live up to its expectations, we need a rich ecosystem of policy-aware services that interoperate with one another. Networking policies should be handled by Neutron; compute policies should be handled by Nova; storage policies should be handled by Swift and Cinder; application policies should be handled by Heat; cross-cutting policies should be handled by a combination of Congress, Gantt, and SolverScheduler. We believe it will be incredibly valuable to give users a single touch point to understand how all the policies throughout the data center interact and interoperate---to provide a dashboard where users ask questions about the current state of the data center, investigate the impact of proposed changes, enact and automate enforcement decisions, and audit the data center's policy from the past.

## Next Steps

To help coordinate the interaction and development of policy-aware services and policy-related efforts in OpenStack, the [OpenStack Mid-Cycle Policy Summit][17] intends to bring representatives from many different policy-minded companies and projects together. The aim of the summit is to discuss the current state of policy within OpenStack and begin discussing the roadmap for how policy will evolve in the future. The summit will start with some presentations by (and about) the various policy-related efforts and their approach to policy; it will wrap up with a workshop focused on how the different efforts might interoperate both today and in the future. Following this summit, which takes place September 18-19, 2014, we'll post another blog entry describing the experience and lessons learned.


[1]: /2014/04/16/policy-problem.html
[2]: /2014/06/18/solution-space.html
[3]: https://wiki.openstack.org/wiki/Congress
[4]:  https://docs.google.com/document/d/1ZbOFxAoibZbJmDWx1oOrOsDcov6Cuom5aaBIrupCD9E/edit?pli=1
[5]: https://wiki.opendaylight.org/view/Group_Policy:Main
[6]:  https://www.openstack.org/blog/2014/07/openstack-swift-2-0-released-and-storage-policies-have-arrived/
[7]: http://docs.openstack.org/developer/swift/overview_policies.html
[8]: https://wiki.openstack.org/wiki/Swift
[9]: https://blueprints.launchpad.net/nova/+spec/solver-scheduler
[10]:  http://openstacksummitmay2014atlanta.sched.org/event/44d44e392250173c4a0344bf27c58860#.U9aFP1hg7zY
[11]:  https://docs.google.com/a/vmware.com/document/d/1k60BQXOMkZS0SIxpFOppGgYp416uXcJVkAFep3Oeju8/edit#
[12]: https://blueprints.launchpad.net/nova/+spec/policy-based-scheduler
[13]:  http://openstacksummitmay2014atlanta.sched.org/event/b4313b37de4645079e3d5506b1d725df#.U9aNq1hg7zY
[14]: https://wiki.openstack.org/wiki/Gantt
[15]: https://github.com/openstack/gantt
[16]: https://review.openstack.org/#/c/95907/7/specs/convergence.rst
[17]: https://www.eventbrite.com/e/openstack-policy-summit-tickets-12642081807
[18]: https://github.com/stackforge/congress