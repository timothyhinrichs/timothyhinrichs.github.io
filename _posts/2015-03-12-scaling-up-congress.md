---
title:              Scaling up Congress
Project:            Congress
Author:             Alex Yip, Shawn Hargan, and Tim Hinrichs
Contributors:       Scott Lowe
Date:               2015-03-10
Keywords:           Congress, OpenStack, Policy
Quotes Language:    english
layout:             post
---

The OpenStack [Congress][1] project aims to provide policy as a service for the data center. Congress is still in the early stages, but quite soon we expect to have our first production deployment. Consequently, we've spent the last couple of months improving its performance. This blog post describes a few crucial optimizations we made to achieve the following performance and memory improvements.

* Query performance: **500x faster**
* Data import speed: **20,000x faster**
* Memory overhead: **6x reduction**

# Speeding up Query Performance

Our earliest experience with Congress policy performance was with a policy we use in VMware’s internal OpenStack cloud. This policy finds underutilized virtual machines, and highlights the VM name, its owner, and its owner’s email address. The policy consists of the following two rules: the first finds VMs that have an average CPU utilization over the last two months that is less than 1; the second takes those VMs, looks up their owner, and reports the owner’s username and email address.

    reclaim_server(vm) :-
        ceilometer:stats("cpu_util",vm, avg_cpu),
        lessthan(avg_cpu, 1)

    error(user_id, email, vm_name) :-
        reclaim_server(vm),
        nova:servers(vm, vm_name, user_id),
        keystone:users(user_id, email)

When we started running this policy in the VMware OpenStack cloud, we found that reading the content of the error table would take around 45 minutes, with the Congress server process using 100% of a CPU. Looking at the log output, we saw that Congress spent most of its time iterating over the `nova:servers` and `keystone:users` tables just to find the email address of the owner of one of a VM already known to be underutilized.

For each VM in `reclaim_server` (there were about 500 entries), Congress would iterate through every entry in `nova:servers` (3,000 entries) to find the ID of the VM's owner, and then it would iterate through every entry in `keystone:users` (70,000 entries) to find that owner's email. All told, Congress was inspecting 73,000 table rows for each entry in `reclaim_server`. In complexity terms, if "n" is the number of rows in `reclaim_servers`, "vm" is the number of rows in `nova:servers`, and "u" is the number of rows in `keystone:users`, this query would take O(n * (vm + u)) time.

[Adding indexing to the Congress policy engine][11] (using a hash table instead of a list), enabled Congress to reduce the complexity from O(vm + u) time per reclaimed server to O(1). Indexing requires more memory, but produces a total complexity of O(n), where "n" is the number of Ceilometer statistics.

In a simple test, indexing produced a **speed up of 530x** (from 680 seconds without indexing to 1.3 seconds with indexing). Because this improvement actually changes the overall complexity, this speedup could be bigger or smaller depending on the dataset, but larger datasets will show larger speedups.

# Speeding Up Data Import

We'd like Congress to operate reasonably well on a dataset with 10 million rows and 1 gigabyte of raw data. To make sure Congress can handle workloads of this size, we benchmarked data insert, policy execution, and change simulation in Congress with that much data. The first thing we found was that inserting data into Congress took a very long time and used a lot of CPU time.

    for i in range(10000):
        congress.insert('r(%d)' % i)

After doing a little digging, we found that the main source of slowness was iterating over all the policy statements (data and rules) each time a new statement was inserted. There were 2 reasons to look at all the rules/data on every insert:

1. Eliminate duplicates
2. Compute a dependency graph over the rules to do syntax checking

To speed up duplicate elimination, we decided to
[use an OrderedDict instead of a Python list][7] so that we could maintain the original ordering of policy statements while quickly eliminating duplicates. We estimate this change produced at least a **200x improvement in performance** (2000 rows took 10s to insert and 20,000 rows took over 30 minutes to insert).

To speed up the graph computation, we modified the algorithm to [*update* the existing dependency graph][6] instead of recomputing it from scratch on each insert. With the graph optimization, 10,000 inserts took just 13 seconds compared to 115 seconds, roughly a **10x performance improvement**.

Next we increased the size of the benchmark to 100,000 facts to exaggerate the expensive aspects of the benchmark and used the cProfile package to get the CPU profile shown below, which says a bulk of the test time is spent in the parser:

<pre>
Wed Feb  4 11:51:08 2015    prof-fast-100k
        148872569 function calls (146161114 primitive calls) in 223.626 seconds
  Ordered by: cumulative time
  ncalls  tottime  percall  cumtime  percall filename:lineno(function)
       1    0.002    0.002  223.633  223.633 nosetests:3(< module >)
...
       1    0.540    0.540  222.385  222.385 test_runtime_performance.py:67(test_insert_nonrecursive)
  100001    0.301    0.000  221.844    0.002 runtime.py:338(insert)
  100001    0.602    0.000  221.377    0.002 runtime.py:435(insert_string)
  100001    0.285    0.000  185.391    0.002 runtime.py:798(parse)
  100001    0.196    0.000  185.107    0.002 compile.py:1566(parse)
  100001    1.045    0.000  184.911    0.002 compile.py:1588(get_compiler)
  100001    0.598    0.000  130.992    0.001 compile.py:1153(read_source)
  100001    2.560    0.000  116.483    0.001 compile.py:1222(parse_file)
  100001    2.731    0.000  106.859    0.001 CongressParser.py:123(prog)
  100000    1.640    0.000   73.005    0.001 CongressParser.py:280(formula)
  100000    1.313    0.000   65.629    0.001 CongressParser.py:450(bare_formula)
  100000    3.465    0.000   57.232    0.001 CongressParser.py:907(atom)
  100002    0.947    0.000   46.594    0.000 optparse.py:1190(__init__)
  200007    0.383    0.000   38.950    0.000 gettext.py:580(gettext)
  200007    0.749    0.000   38.567    0.000 gettext.py:542(dgettext)
  200023    0.584    0.000   37.708    0.000 gettext.py:476(translation)
  200023    6.501    0.000   37.124    0.000 gettext.py:421(find)
  100001    1.436    0.000   35.249    0.000 runtime.py:465(update_obj)
</pre>

When a caller passes a string representation of a fact or rule to `congress.insert()`, Congress needs to parse the string into the python objects Congress uses internally to represent policies statements like `Rule`, `Literal`, etc., and this parsing process is expensive. Instead of passing a string, the caller can pass in a `Literal` object to `insert()`. Changing the benchmark to insert a `Literal` object instead of a string reduces the benchmark time from 130 seconds to 26 seconds, a **speedup of 5x**. Knowing this, we removed parsing from the data-load critical path.

After eliminating the unnecessary parsing and increasing the number of rows to 1,000,000, insertion takes 272 seconds, which is still too long. Looking at the CPU profile shows a few surprisingly expensive functions: UUID creation costs, string operations, computing the hash, and logging.

<pre>
Wed Feb  4 14:25:10 2015    prof-1000k
        250439265 function calls (224427810 primitive calls) in 423.668 seconds
  Ordered by: internal time
  ncalls  tottime  percall  cumtime  percall filename:lineno(function)
 2000004   35.449    0.000   44.624    0.000 uuid.py:101(__init__)
 4000003   28.108    0.000   58.340    0.000 log.py:302(process)
 2000000   26.635    0.000   80.440    0.000 uuid.py:531(uuid4)
 1000000   22.400    0.000   25.162    0.000 compile.py:305(is_ground)
8006191/6005385   17.581    0.000   52.147    0.000 {method 'join' of 'str' objects}
 4000001   14.637    0.000  102.491    0.000 base.py:41(log)
28000274/4000274   13.554    0.000   53.212    0.000 {repr}
 1000001   13.126    0.000  345.882    0.000 runtime.py:465(update_obj)
 4000000   12.764    0.000   49.461    0.000 compile.py:278(__repr__)
 4000003   11.562    0.000   82.679    0.000 __init__.py:1406(debug)
 4000003   11.110    0.000   11.110    0.000 local.py:23(__getattribute__)
10000238   10.374    0.000   10.374    0.000 {method 'format' of 'str' objects}
19016037    8.961    0.000    8.961    0.000 {isinstance}
 2000000    8.094    0.000   48.623    0.000 compile.py:566(__hash__)
 4003958    8.076    0.000   19.186    0.000 {getattr}
</pre>

To fix these issues, we now:

* [Compute the hash using a `tuple`][4] of an object’s contents instead of a string representation
* [Cache the hash value][4] after computing it
* Have [no ID field in the Rule][5] object

With these optimizations in place, inserting 1,000,000 facts took 135 seconds, down from 272 seconds, a **speed up of 2x**. We tried removed the logging in the `insert()` code path, which brings the time down to 66 seconds (**another 2x speedup**), but we will make those changes later since they make it harder to debug.

# Reducing Memory Use

Next we ramped up the benchmark to insert 10,000,000 facts. Unfortunately, 10,000,000 facts used more memory than our machine had, which caused Congress to use swap space and run several hours without finishing. Thus the next step in improving performance was understanding and reducing the memory overhead for simple facts. Here's what we found:

* 1M facts of the form `p(1)` used 2.4 GB of memory
* 10M facts of the form `p(1)` used 24 GB of memory, which was more than our machine had
* 1M facts of the form `p(1, 2, 'foo', 'bar', i, ‘a’ * 100 + str(i))` used 4.5 GB of memory
* 1 fact of the form `p(1, 2, 'foo', 'bar', i, ‘a’ * 100 + str(i))` has a payload of 110 bytes
* Memory overhead for `p(1, 2, 'foo', 'bar', i, ‘a’ * 100 + str(i))` was 41x

The memory overhead was caused by the decision to represent a simple fact (of which there are many) using the same data structures it used to represent a policy rule (of which there are few).

Congress stored each fact as a `Rule` object, and each `Rule` object contained a `Literal` object, which in turn contained a `Term` object for each element in the fact. Each `Rule` object has five fields for things like head, body, and the cached hash value we added above. Similarly, `Literal` has 6 fields, and `Term` has 4 fields. Each object of type `Rule`, `Literal`, and `Term` uses 64 bytes, so for a fact with 6 elements the scaffolding uses 64 + 64 + 64 * 6 == 512 bytes, not including the payload data in the fact itself.

To reduce the memory overhead, we [introduced a new data structure][3] `Fact` just for simple facts and began storing facts separately from rules. A `Fact` object is a subclass of the Python `tuple`, so it has no overhead except for the overhead of a tuple plus a field storing the table’s name. We also streamlined the interface for inserting facts so that it does not log each fact, and uses generators to move facts into the policy engine to avoid unnecessary memory allocation.

After changing Congress to use the `Fact` class, we saw the following results:

* 1M facts of the form `p(1, 2, 'foo', 'bar', i, ‘a’ * 100 + str(i))` take 11 seconds to load and 775 MB of memory
* 10M facts of the form `p(1, 2, 'foo', 'bar', i, ‘a’ * 100 + str(i))` take 113 seconds to load and 7.3 GB of memory, a memory overhead of 6.6x
* Memory overhead was reduced from 41x to 6.6x, which is a **6x reduction in memory overhead**
* While we could not measure the time required to load 10M facts without our optimization, we estimate (conservatively) a **100x data load speedup** (3+ hours reduced to 113 seconds)

# Summary

All told, we made several different optimizations that led to a 500x query speedup, 20,000x data load speedup, and a 6x memory reduction.

| Optimization | Result |
| ---- | ---- |
| Added indexing to Datalog engine | 500x query speedup |
| OrderedDict to store rules instead of List | 200x data load speedup |
| Incremental dependency graph computation | 10x data load speedup |
| Avoid parsing strings | 5x data load speedup |
| Improved hashing and UUID elimination | 2x data load speedup  |
| Introduction of `Fact` | 6x memory reduction and 100x data load speedup |

{::comment}
In the future we may optimize further:

* Congress currently converts each `Fact` object to a `Rule` object when answering queries, which can be eliminated.
* We can eliminate the table field from each `Fact` to further reduce memory consumption.
{:/comment}

Our next project is designing and implementing a scale-out architecture to ensure Congress is highly-available and can meet the throughput demands of a deployment environment. The blueprint is [here][2]; the spec should be available soon. Suggestions are welcome!

Please feel free to join our weekly [IRC meeting][8], drop by the #congress channel on IRC, check out the [wiki][9], or download and install the [code][10].

[1]: /2014/12/09/congress-overview.html
[2]: https://blueprints.launchpad.net/congress/+spec/query-high-availability
[3]: https://review.openstack.org/#/c/150593/
[4]: https://review.openstack.org/#/c/147365/
[5]: https://review.openstack.org/#/c/147320/
[6]: https://review.openstack.org/#/c/146237/
[7]: https://review.openstack.org/#/c/135469/
[8]: https://wiki.openstack.org/wiki/Meetings/Congress
[9]: https://wiki.openstack.org/wiki/Congress
[10]: https://github.com/stackforge/congress/blob/master/README.rst
[11]: https://review.openstack.org/#/c/142874/
