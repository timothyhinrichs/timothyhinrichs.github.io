---
title:              On Policy in the Data Center - Congress
Project:            Congress
Author:             Tim Hinrichs and Scott Lowe
Contributors:       Alex Yip, Dmitri Kalintsev, and Peter Balland
Date:               2014-12-01
Keywords:           Congress, OpenStack, Policy
Quotes Language:    english
layout:             post
---

In the first few parts of this series, we discussed the [policy problem][1], we outlined dimensions of the [solution space][2], and we gave a brief overview of the existing [OpenStack policy efforts][3]. In this post we do a deep dive into one of the (not yet incubated) OpenStack policy efforts: Congress.


## Overview

Remember that to solve the policy problem, people take ideas in their head about how the data center ought to behave ("policy") and codify them in a language the computer system can understand. That is, the policy problem is really a programming languages problem. Not surprisingly Congress is, at its core, a policy language plus an implementation of that language.

Congress is a standard cloud service; you install it on a server, give it some inputs, and interact with it via a RESTful API. Congress has two kinds of inputs:

* The other cloud services you'd like it to manage (for example, a compute manager like OpenStack Nova and a network manager like OpenStack Neutron)
* A policy that describes how those services ought to behave

For example, we might decide to use Congress to manage OpenStack's Nova (compute), Neutron (networking), Swift (object storage), Cinder (block storage), and Heat (applications). We might write a geo-location policy:

>"Every application collecting personal information about its users from Japan must have all of its compute, networking, and storage elements deployed at a data center that resides within the geographic borders of Japan."

## Any Service

A cloud service gives Congress the ability to see and change the data center's behavior. The more services hooked up to Congress, the more powerful Congress becomes. Congress was designed to manage **any** collection of cloud services, regardless of their origin or locality (private or public). It does not matter if the service is provided by OpenStack, CloudStack, AWS, GCE, Cisco, HP, IBM, Red Hat, VMware, etc. It does not matter if the service manages compute, networking, storage, applications, databases, anti-virus, inventory, people, or groups. Congress is vendor and domain agnostic.

Congress provides a unified abstraction for all services, which insulates the policy writer from understanding differing data formats, authentication protocols, APIs, and the like. Congress does NOT require any special code to be running on the services it manages; rather, it includes a light-weight adapter for each service that implements the unified interface using the service's native API.

From the policy writer's point of view, each service is simply a collection of tables. A _table_ is a collection of rows; each _row_ is a collection of columns; each row-column entry stores simple data like strings and integers. When Congress wants to see what is happening in the data center, it reads from those tables. When Congress wants to change what is happening in the data center, it writes to those tables.

For example, the Nova compute service is represented as several tables like the _servers_ table below.

    +---------+----------+--------+-----------+----------+-----------+
    | id      | host_id  | status | tenant_id | image_id | flavor_id |
    +---------+----------+--------+-----------+----------+-----------+
    | <UUID1> | <UUID2>  | ACTIVE | alice     | <UUID3>  | <UUID4>   |
    | <UUID5> | <UUID6>  | ACTIVE | bob       | <UUID7>  | <UUID8>   |
    | <UUID9> | <UUID10> | DOWN   | bo        | <UUID7>  | <UUID8>   |
    +---------+----------+--------+-----------+----------+-----------+

At the time of writing, there are adapters (which we call "datasource drivers") for each of the following services, all but one of which are OpenStack.

* Nova
* Neutron
* Cinder
* Swift
* Keystone
* Ceilometer
* Glance
* Plexxi controller

Each adapter is a few hundred lines of code that (i) makes API calls into the service to get information about that service's behavior; and (ii) translates that information into tables. Just recently we added a domain-specific language (DSL) that automates the translation of that information into tables, given a description of the structure of the information.

For more information about connecting cloud services, see the [Congress cloud services documentation][4].

## Any Policy

A policy describes how a collection of cloud services ought to behave. Every organization's policy is unique because every organization has different services in its data center. Every organization has different business advantages they are trying to gain via the cloud. Every organization has different regulations that govern it. Every organization is full of people with different ideas about the right way to run the cloud.

Congress aims to provide a single policy language that every organization can use to express their high- and low-level policies. Instead of providing a long list of micro-policies that the user can mix-and-match, Congress provides a general purpose policy language for expressing policy: the well-known declarative language [Datalog][5].

**Datalog is domain-agnostic.**  It is just as easy to write policy about compute as it is to write policy about networking. It is just as easy to write policy about how compute, networking, storage, group membership, and applications interact with each other. Moreover, Datalog enables policy writers to define abstractions to bridge the gap between low-level infrastructure policy and high-level business policy.

Suppose our policy says that all servers should on average have a CPU utilization of at least 20% over a 2 day span. In Datalog we would write the a policy that leverages Nova for compute, Ceilometer for CPU utilization data, and some built-in tables that treat strings as if they were dates.

First, we declare the conditions under which there is a policy violation. We do that by writing a rule that says a VM is an error (policy violation) if the conditions shown below are true.  (Lines starting with // are comments.)

    error(vm, email_address) :-
        // myid is a server owned by owner
        nova:servers(id=myid, tenant_id=owner),
        // start_date is 2 days before end_date; end_date is today
        two_days_previous(start_date, end_date),
        // value is average cpu-utilization for last 2 days for server myid
        ceilometer:statistics(id=myid, start=start_date, end=end_date, meter="cpu-util", avg=value),
        // value is less than 20%
        arithmetic:less_than(value, 20),
        // email_address is owner's email
        keystone:user(id=owner, email=email_address)

We also define a helper table that computes the start and end dates for 2 days before today.

    two_days_previous(start_date, end_date) :-
        datetime:now(end_date),
        datetime:minus(end_date, "2 days", start_date)

Helper tables like `two_days_previous` are useful because they allow the policy writer to create higher-level concepts that may not exist natively in the cloud services. For example, we can create a helper table that tells us which servers are connected to the Internet---something that requires information from several different places in OpenStack. Or the compute, networking, and storage admins could create the higher-level concept "is-secure" and enable a higher-level manager to write a policy that describes when resources ought to be secured.

For more information about writing policy, see the [Congress policy documentation][6].

## Capabilities

Once we have connected services to Congress and written policy over those services, we've given Congress the inputs it needs carry out its core capabilities, which the user is free to mix and match.

* **Monitoring:** Congress watches how the other cloud services are behaving, compares that to policy, and flags mismatches (policy violations).

* **Enforcement:** Congress acts as a policy authority. A service can propose a change to Congress, and Congress will tell the service whether the change complies with policy or not, thus preventing policy violations before they happen. Congress can also correct some violations after the fact.

* **Auditing:** Congress gives users the ability to record the history of policy, policy violations, and remediations.

* **Delegation:**  Congress can offload the burden of monitoring/enforcing/auditing to other policy-aware systems.

When it comes to enforcement, a common question is why Congress would support both proactive and reactive enforcement. The implied question being, "Isn't proactive always preferred?"  The answer is that proactive is not always possible. Consider the simple policy "ensure all operating systems have the latest security patch."  As soon as Microsoft/Apple/RedHat releases a new security patch, the policy is immediately violated; the whole purpose of writing the policy is to enable Congress to identify the violation and take action to correct it.

The tip of [master][7] includes monitoring and a mechanism for proactive enforcement. In the Kilo release of OpenStack we plan to have a form of reactive enforcement available as well.

## Summary

In this post, we've talked about some of the key takeaways regarding Congress:

* Congress was designed to solve the policy problem and work with any cloud service and any policy.
* It is currently capable of monitoring and proactive enforcement. Reactive enforcement and delegation are currently underway.
* Congress is not yet incubated in OpenStack, but has contributions from half a dozen organizations and nearly two dozen people.

Please feel free to join our weekly [IRC meeting][8], check out the [wiki][9], and download and install the [code][10].


[1]: /2014/04/16/policy-problem.html
[2]: /2014/06/18/solution-space.html
[3]: /2014/09/17/openstack-policy-efforts.html
[4]: https://github.com/stackforge/congress/blob/master/doc/source/cloudservices.rst
[5]: https://en.wikipedia.org/wiki/Datalog
[6]: https://github.com/stackforge/congress/blob/master/doc/source/policy.rst
[7]: https://github.com/stackforge/congress
[8]: https://wiki.openstack.org/wiki/Meetings/Congress
[9]: https://wiki.openstack.org/wiki/Congress
[10]: https://github.com/stackforge/congress/blob/master/README.rst
