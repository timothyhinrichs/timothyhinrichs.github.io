---
title:              2014 Research Papers Relevant to Congress
Project:            Congress
Author:             Tim Hinrichs
Contributors:
Affiliation:        VMware, Inc.
Date:               2015-01-10
Keywords:           Congress, OpenStack, Policy
Quotes Language:    english
layout:             post
fullview:           false
---

While perusing the 2014 research conference proceedings, I found a number of papers relevant to [Congress][1].  In this post I've cited each paper, given a quick overview of the main ideas, and described why it is relevant to Congress.  The hope is that some of the core ideas in these papers will be springboards to help us address similar issues in Congress.

Here are the conferences I surveyed.

* Computer Security: [Oakland][oakland2014], [Usenix Security][usenix2014], [NDSS][ndss2014], [CCS][ccs2014], [CSF][csf2014]
* Programming languages: [POPL][popl2014], [PLDI][pldi2014]
* Databases: [VLDB][vldb2014], [PODS/SIGMOD][pods2014]
* Operating Systems: [OSDI][osdi2014], [SOSP 2013][sosp2013], [SIGCOMM][sigcomm2014]
* Artificial Intelligence/Logic: [AAAI][aaai2014], [IJCAI 2013][ijcai2013], [ICLP][iclp2014]


## Datasources

**Weimo Liu, Saravanan Thirumuruganathan, Nan Zhang, Gautam Das:
[Aggregate Estimation Over Dynamic Hidden Web Databases][2]. PVLDB 7(12): 1107-1118 (2014)**

In the context of Congress, Liu's "hidden web database" is a datasource with large amounts of data with an interface that only allows us to periodically poll information using a limited query language.  Liu addresses the problem of computing aggregates (such as count/sum/average) using such datasources.  The main challenge is that aggregates are often not allowed in the query interface, and hence the best we can hope to do is approximate the aggregate.  Liu's approach first estimates a signature that represents how volatile the datasource is and then constructs two classes of queries based on the results: (i) queries that update the current aggregate estimate (for when the data is changing frequently) and (ii) queries that compute a new aggregate from scratch (for when the data is changing infrequently).

For Congress, the idea of estimating the volatility signature for an arbitrary datacenter service seems too difficult.  But if the operator supplied that information somehow (or could at least edit it), then trying to minimize the number of pollings that we do to compute aggregates is attractive.

## User interface

**Angela Bonifati, Radu Ciucanu, Slawek Staworko:
[Interactive Join Query Inference with JIM][3]. PVLDB 7(13): 1541-1544 (2014)**

Bonifati describes a user interface for laymen to construct joins without writing SQL/Datalog or even knowing the schema.  In Congress, a join is basically the right-hand side of a Datalog rule, which dictates how precisely to combine existing tables to construct a new table.  Bonifati's key idea is to present tuples that could potentially be in the new table, ask the user to label each tuple positively or negatively, and iterate until they have identified the proper join.

It is unclear to me how the user knows whether to label a tuple positively or negatively.  In Congress's case the point of writing the rules is to discover something about the state of the datacenter.  For example, the user might want to discover which VMs are  connected to the internet.  Without knowing the schema there's no real way I can see the user having any idea what the tuples ought to look like (either of the result or of the intermediate join).

## Policy languages

**Petar Tsankov, Srdjan Marinovic, Mohammad Torabi Dashti, David A. Basin: [Fail-Secure Access Control][5]. ACM Conference on Computer and Communications Security 2014: 1157-1168**

Tsankov studies the problem of failures of information sources for access control systems.  If an access control policy depends on information from an external system, and that external system fails to return the needed information, how should the access control system respond to access control requests?  Tsankov proposes giving administrators the ability to write a policy that dictates what should happen in such circumstances using BelLog, which is a variant of Datalog that uses 4 truth values.  BelLog can also be used to describe how conflicts in different policies ought to be combined.

In Congress, datasources may fail or become unreachable, and we are left deciding how Congress should operate.  (Access control is implemented via the 'simulate' functionality in Congress.)  Providing a Datalog-like language for describing conflict resolution as well as error-handling is an attractive idea.

**Thomas Lukasiewicz, Maria Vanina Martinez, Gerardo Ignacio Simari: [Preference-Based Query Answering in Datalog+/- Ontologies][7]. IJCAI 2013**

Lukasiewicz describes an approach of adding preferences to a Datalog-based ontology language.

Congress's error tables are an ontology, so this paper should enable us to add preferences to Congress's policy language as well.


**Werner KieBling, and Gerhard Kostler: [Preference SQL - Design, Implementation, Experiences][6]. VLDB 2002: 990-1001.**

This paper, cited by Lukasiewicz, details how we can add preferences to SQL (where a preference is a partial order).  The implementation compiles Preferences + SQL to raw SQL.  The approach has been used in industrial settings.

This is relevant to Congress if we decide to add preferences to our policy language.  SQL and Datalog are close cousins, and the same ideas described in this paper should enable us to compile Preferences + Datalog to Datalog.

**Christos Dimoulas, Scott Moore, Aslan Askarov, Stephen Chong: [Declarative Policies for Capability Control][7]. CSF 2014: 3-17**

Dimoulas extends the notion of capability-safe programming languages to include declarative policies for access control and integrity.  A capability-safe programming language is one in which a software component must have a token before using another component.  They are one way of implementing an access control policy (which dictates which components can use which other components).  An integrity policy is one that dictates which component can influence the use of which other component.  This project gives rigorous semantics to these concepts and shows how to use high-order contracts to enforce the access control policy and a security type system to enforce the integrity policy.

In the context of Congress, we can think of all the services running in the datacenter as components of one large software system.  The approach taken in this paper, however, seems to assume there is one programming language (runtime, compiler) used to write all of the components--something we will never have in Congress.  Nevertheless, the basic concepts employed in this paper may be useful.

**Yuanzhong Xu, Alan M. Dunn, Owen S. Hofmann, Michael Z. Lee, Syed Akbar Mehdi, Emmett Witchel: [Application-Defined Decentralized Access Control][9]. USENIX Annual Technical Conference 2014: 395-407**

Xu presents a modification of Linux that enhances its usual discretionary access control model with one that makes it easier for Linux users and developers to autonomously implement the principle of least priviledge.  The approach obviates the need for root access to create new principals, define new access control policies, and assign privileges to Linux processes.

Xu describes a mechanism for enhancing policy support at the operating system level--a capability that would be nice once Congress begins to enforce policy at that layer.

## Policy queries and performance

**Meghyn Bienvenu, Riccardo Rosati: [Tractable Approximations of Consistent Query Answering for Robust Ontology-based Data Access][10]. IJCAI 2013**

Bienvenu introduces generalizations of approximations for inconsistency-tolerant semantics for query answering (implication/entailment) in first-order logic that have polynomial-time data complexity.  This paper includes a good overview of past work and enough preliminaries to be mostly self-contained.

In the context of Congress, we expect there to be policy violations, which amount to inconsistencies between policy and the state of the datacenter.  In addition, the fragments of first-order logic addressed in Bienvenu's paper are a fragment of Datalog.
When we begin to introduce assistive enforcement (where external services can ask Congress to provide values that are consistent with policy), we must be able to answer those queries even though inconsistencies already exist.  There are two ways to answer such queries: find values that are consistent with *every* possible repair of the policy violations and find values that are consistent with *some* repair.  Bienvenu addresses queries about *implication* instead of *consistency*, but the two are tightly connected (one is the negation of the other, classically).


**Meikel Poess, Tilmann Rabl, Brian Caufield: [TPC-DI: The First Industry Benchmark for Data Integration][11]. PVLDB 7(13): 1367-1378 (2014)**

Poess describes the rationale behind a benchmark for data integration systems.  It includes the task of loading large amounts of data and the task of incrementally receiving data updates.  The benchmark can be downloaded directly from the TPC organization's website [here][12].

Congress integrates data from many different data sources, and hence this benchmark is useful for testing scalability.

**Yannis Klonatos, Christoph Koch, Tiark Rompf, Hassan Chafi: [Building Efficient Query Engines in a High-Level Language][13]. PVLDB 7(10): 853-864 (2014)**

Klonatos describes an approach to building an in-memory database query optimizer by writing Scala code that is then compiled into highly-efficient C code that is customized for each query.  This deviates from the traditional wisdom that says query optimizers ought to be written in a low-level programming language to achieve efficiency at the cost of developer productivity.  The resulting query optimizer is competitive with a commercial, in-memory database optimizer.

In Congress, enumerating policy violations is accomplished by evaluating a (Datalog) database query.  At the time of writing, Congress performs no query optimization but rather answers a query in the order the user has specified.  Klonatos's approach may enable us to build optimizer sooner rather than later.


## Declarative Distributed Systems

Declarative networking is an approach to building distributed systems where the programmer writes a declarative program that is executed on all nodes of the system and nodes communicate using declarative statements (often just data).

**Tom J. Ameloot, Bas Ketsman, Frank Neven, Daniel Zinn: [Weaker forms of monotonicity for declarative networking: a more fine-grained answer to the calm-conjecture][16]. PODS 2014: 64-75**

Ameloot investigates a conjecture that says the nodes carrying out some computation can do so without coordination exactly when the computation is monotonic (e.g. can be expressed in a Datalog program without negation).  Ameloot provides a more fine-grained computational model for coordination-freeness than was previously available and proves it equivalent to a new definition of monotonic programs.

In Congress, we are planning to incorporate the ability to delegate monitoring, enforcement, and auditing to other policy engines.  Certainly in the context of monitoring this means distributing the evaluation of a query to multiple nodes in the system and exchanging data when necessary.  Work on declarative networking is therefore pertinent to Congress.  Ameloot's paper in particular aims to understand the fundamental nature of this problem, which is valuable by itself.  However, the paper's practical value is limited since there are no (even approximation) algorithms for checking if a given query is in one of the classes investigated, or how coordination-freeness impacts the design of the system.

**Serge Abiteboul, Meghyn Bienvenu, Alban Galland, Emilien Antoine: [A rule-based language for web data management][17]. PODS 2011: 293-304**

Abiteboul introduces a Datalog-based networking language (called Webdamlog) that enables nodes in a distributed system to communicate using both rules and data.  The Datalog programmer explicitly writes rules that describe when data should be sent to another node.  The Datalog programmer also has control over when rules are sent to another node.  The rules sent from one node to another are simplified instances of existing rules.  In particular, node A sends to node B every instance of its own rules whose head tables are located at B after evaluating the portion of the rule body local to A.  Whether delegation adds any expressiveness depends on the precise definitions of the system.

For Congress, this is relevent because the Congress will delegate policy to other policy engines in the system.  The fundamental problem of delegation seems to be choosing which rules to send to which node, which makes Webdamlog pertinent.  But the Congress problem is slightly different than the Webdamlog problem.  The biggest difference is that we want to hide the details of delegation from the policy writer (who may not be a programmer).  A less important but noteworthy difference is that Congress cannot assume that all of the other policy engines are using Datalog; they may have fundamentally less expressive languages.


## Policy explanations
**Alexandra Meliou, Sudeepa Roy, and Dan Suciu, [Causality and Explanations in Databases][14], PVLDB 7(13): 1715-1716 (2014)**

Meliou writes a 2-page overview of a tutorial on finding explanations of why a database query answer is true, where the database answer may include aggregates.  This paper includes a handful of references to other papers that do deeper work on the topic.

In Congress, a policy violation amounts to a row belonging to a particular database table, and hence explaining a policy violation to a user means explaining a database query.  Enabling cloud administrators to understand the root causes of violations is an important feature for the usability of the system.

**Luciano Caroprese, Irina Trubitsyna, Miroslaw Truszczynski, Ester Zumpano: [A Measure of Arbitrariness in Abductive Explanations][15]. TPLP 14(4-5): 665-679 (2014)**

Caroprese looks at the problem of measuring the quality of different abductive explanations.  An abductive explanation is a modification to a set of ground facts that make a given (database) query true (perhaps in the presence of integrity constraints).  Caroprese measures different explanations by the number of ways a given explanation's arguments can be replaced with other arguments; the higher this measure, the more arbitrary the explanation actually is.  Caroprese proves complexity results on finding an explanation with a given amount of arbitrariness along several different dimensions of Datalog systems: recursion, negation, integrity constraints.

For Congress, abduction queries may help cloud administrators understand how they might correct a policy violation.  By asking to make the negation of a policy violation true, each abductive explanation of that query eliminates the violation; finding a handful of the most important abductive explantions (under some metric of importance) would help guide the administrator toward eliminating a violation.  This paper introduces the first such metric I am aware of and details the basic algorithmic ideas for how to identify important explanations.

## Policy Enforcement

**Wen Zhang, You Chen, Thaddeus Cybulski, Daniel Fabbri, Carl A. Gunter, Patrick Lawlor, David M. Liebovitz, Bradley Malin: [Decide Now or Decide Later?: Quantifying the Tradeoff between Prospective and Retrospective Access Decisions][4]. ACM Conference on Computer and Communications Security 2014: 1182-1192**

Zhang develops a framework for deciding whether to ignore an access control decision and rely on audit to catch malicious behavior.  In the context of Congress, this work seems relevant as a way of thinking about the tradeoffs of proactive and reactive enforcement (assuming we had separate policies for each).  The key idea is to utilize machine learning to represent the proactive as well as reactive policies, incorporate costs for violations in each policy, and thereby determine for any new access control request whether to allow or deny it.

Assuming in Congress we had one policy for proactive enforcement and a separate policy for reactive enforcement, we could imagine this line of thinking being used to decide whether to enforce policy proactively or reactively for any given request.




[1]: blog entry on Congress
[2]: http://www.vldb.org/pvldb/vol7/p1569-liu.pdf
[3]: http://www.vldb.org/pvldb/vol7/p1541-bonifati.pdf
[4]: http://seclab.illinois.edu/wp-content/uploads/2014/08/Zhang+14.pdf
[5]: http://www.inf.ethz.ch/personal/basin/pubs/failsecure-ccs14.pdf
[6]: http://www.vldb.org/conf/2002/S30P03.pdf
[7]: http://ijcai.org/papers13/Papers/IJCAI13-155.pdf
[8]: http://people.seas.harvard.edu/~chong/pubs/csf14_capflow.pdf
[9]: https://www.cs.utexas.edu/~adunn/pubs/xu14atc-dcac.pdf
[10]: http://ijcai.org/papers13/Papers/IJCAI13-121.pdf
[11]: http://msrg.org/publications/pdf_files/2014/VLDB2014TPCDI-TPCDI:_The_First_Industry_Bench.pdf
[12]: http://www.tpc.org/tpcdi/digen-download-request.asp
[13]: http://www.vldb.org/pvldb/vol7/p853-klonatos.pdf
[14]: http://www.vldb.org/pvldb/vol7/p1715-meliou.pdf
[15]: http://arxiv.org/pdf/1405.2494v1.pdf
[16]: http://alpha.uhasselt.be/~lucg5503/papers/pods2014.pdf
[17]: https://hal.inria.fr/inria-00582891/document

[oakland2014]: http://www.ieee-security.org/TC/SP2014/
[usenix2014]: https://www.usenix.org/conference/usenixsecurity14
[ndss2014]: http://www.internetsociety.org/events/ndss-symposium-2014
[ccs2014]: http://www.sigsac.org/ccs/CCS2014/
[csf2014]: http://csf2014.di.univr.it/
[popl2014]: http://popl.mpi-sws.org/2014/
[pldi2014]: http://conferences.inf.ed.ac.uk/pldi2014/
[vldb2014]: http://www.vldb.org/2014/
[pods2014]: http://www.sigmod2014.org/

[osdi2014]: https://www.usenix.org/conference/osdi14
[sosp2013]: http://sigops.org/sosp/sosp13/
[sigcomm2014]: http://conferences.sigcomm.org/sigcomm/2014/
[aaai2014]: http://www.aaai.org/Conferences/AAAI/aaai14.php
[ijcai2013]: http://ijcai13.org/
[iclp2014]: http://users.ugent.be/~tschrijv/ICLP2014/


