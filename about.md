---
layout: page
title: About
header: About
comments: False
---

Recently there has been growing interest in **policy technology** in the field of **cloud computing**.  While everyone means something different by *policy*, for our purposes *policy* means anything that describes how the cloud should behave.  Policy might dictate the ideal behavior for the cloud in terms of compute, networking, storage, and applications, or it might describe the cloud's ideal behavior in terms of cost, risk, and security.  The abstractions and form of the description are irrelevant; what matters for us is that a *policy* describes how the cloud should behave.

# Why Policy?

Why are policy languages important for the cloud?  In any cloud, different people and organizations have different ideas about how that cloud ought to behave.  For example:

* Each end user has different computing needs.
* The organization who owns the cloud wants it to operate efficiently and to obey service-level agreements.
* Industries as a whole agree on best-practices for computing resources.
* Governments pass laws that influence the cloud and the applications that run on it.

All of these real-world rules and regulations influence how the cloud must behave; yet, to achieve self-service (one of the basic goals of cloud computing), it is the cloud itself that must combine those different demands and work to obey them all.  Expecting the end-user to understand all of the other peoples' demands is simply unrealistic.

Policy-based cloud management systems address this issue head-on.  A policy language enables people to describe how they would each like the cloud to behave, using a language the computer system can understand.  The system then combines the demands from all those different people and takes action to ensure those demands are all met (or at least identifies when they cannot be met).


# Example

Here is an example of several different people all with different ideas about how the cloud ought to behave.  The goal of a policy-based cloud-management system is to accept a description of these different demands and ensure that whatever happens in the cloud, these demands are all simultaneously met (or to at least identify when  demands are not met).

1. **Application-developer**: My 2-tier PCI app (database tier and web tier) can be deployed either for production or for development.  When deployed for production, it needs
    * solid-state storage for the DB tier
    * all ports but 80 closed on the web tier
    * no network communication to DB tier except from the web tier
    * no VM in the DB tier can be deployed on the same hypervisor as another VM in the DB tier; same for the web tier
2. **Cloud operator**
    * Applications deployed for production must have access to the internet.
    * Applications deployed for production must not be deployed in the DMZ cluster.
    * Applications deployed for production should scale based on load.
    * Applications deployed for development should have 1 VM instance per tier.
    * Every application must use VM images signed by an administrator.
3. **Compliance officer**
    * No VM from a PCI app may be located on the same hypervisor as a VM from a non-PCI app.

<!--

The promise of policy technology is incredibly **attractive for customers** (organizations deploying cloud technology): people tell a cloud management system how the cloud ought to behave, and the computer system makes it happen.  Policy technology helps people cope with the staggering size and complexity of the infrastructure that makes up a cloud, the applications that run on that infrastructure, and peoples' disparate ideas about how the cloud should behave.

Policy technology is **attractive for vendors** because once a piece of technology has been given a description of what is supposed to happen, that technology knows precisely *what* its goals are and is free to implement those goals *how*ever it likes.  This separation of *what* and *how* gives vendors the freedom to implement the same API as competitors yet be innovative and differentiate their products.  And by supporting policy, vendors give customers the ability to manage technology the way their customers would like to manage it.
 -->

# Goals
The goal of this blog is to facilitate in-depth discussions about policy technology and how it can impact the world.  We plan to include tutorials, system descriptions, deployment experiences, applications, algorithms, position pieces, etc.  We will focus primarily on cloud computing, but we hope to include exciting advances in related areas as well.
