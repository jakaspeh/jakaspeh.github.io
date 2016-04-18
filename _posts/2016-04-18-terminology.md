---
layout: post
title:  "Terminology of concurrent/parallel programming"
date:   2016-04-18
categories: concurrency
---

I am a mathematician. This could be a blessing or a curse. In the world of
mathematics, everything is well defined and organized. But then, when a
mathematician gets into the real world, the world sometimes seems like a big
mess. It is not so nicely organized and there are contradictions everywhere.

Lately, I was writing a lot about concurrent programming. But there are some
things which annoyed the mathematician inside me. This article is an attempt to
clarify the terminology of parallel and concurrent programming

Parallel and concurrent programming
-----------------------------------

They both can be described as multiple processes executing during the same
time. But there are some differences between them.

Concurrent programming
======================

For concurrent programming is typical to have a non deterministic flow. It is
possible to run a concurrent program on only one-core (single processor)
machine. The concurrent programming deals with multiple threads but they do not
necessarily run on multiple cores/processors.

[Time-sharing](https://en.wikipedia.org/wiki/Time-sharing) is an example of
concurrent programming, which is not parallel.

Some people also understand a concurrent programming as dealing with shared
resources.

Parallel programming
====================

Typical workflow for parallel programming is to brake down a task in several
subtasks that can be processed independently. The results of the subtasks are
then merged together.

Parallel programming is not possible on only one core. Each computation is done
on a separate core/processor. 

An important aspect of the parallel programming is the execution of the
program. The program should execute as efficiently as possible.

Some people also understand the parallel programming as using extra
computational units to do more work per time.

Caveat
======

Of course people do not agree on the definitions. You should clarify the
terminology when speaking with someone about concurrency/parallelism, because
they might use a different vocabulary.

It looks a lot like defining a set of natural numbers in mathematics. Some
mathematicians include zero into the definition of natural numbers and others
define natural numbers without zero.


Shared and distributed memory
-----------------------------

There are two main memory models.

In a machine with **shared memory** each thread has access to all
memory. An example of such machine is a laptop with four cores. In the laptop,
each core can access all the memory.

The second memory model is the **distributed memory**. Let's say that we have
four one-core computers connected together over the internet. The core from one
computer can not access the memory from another computer. Therefore the cores do
not share the memory between each other. This is a simple example of a machine
with distributed memory.

There also exists computers with "mixed" memory. They are called computers with
**shared distributed model**. In this model the cores have a local memory, but
they can also access a region of memory which is shared between all cores.

How to write concurrent/parallel programs in a shared memory? We can use two
different programming paradigms.

Pure functional programming
---------------------------

Programming in pure functional style deals with functions which depend solely on
the input. Such functions are perfect for running in separate threads, because
the threads do not modify the shared state.

The workflow in such paradigm is simple. We spawn a thread with a task and its
inputs and then we collect the results.

Not pure functional programming (imperative programming)
--------------------------------------------------------

Threads in this programming paradigm modify the shared state. This might
potentially bring some problems such as data races...

How can we deal with this problems? Usually, we use some data structures to
manage the synchronization of the shared state. We can divide such data
structures into two groups.

Lock based data structures 
==========================

Lock based data structures use locks to synchronize threads between each
other. Here, we use the mutexes. The idea is to serialize parts of the code, by
blocking other threads.

Lock free data structures 
=========================

Lock free data structures do not use mutexes and do not block the threads. The
synchronization of the threads is done by low level atomic operations. The atomic
operations are operations which are not dividable. 

General
=======

The easiest way to program parallel programs is to write in the pure functional
style. Programming lock based data structures is harder, but still
manageable. The lock-free data structures are the hardest one to get them right.

The reason for such scale is that debugging/testing "concurrent data structures"
is harder then the "sequential data structures". The bugs might occur just in
some special cases, when some special ordering of operations take place. The
debugging/testing pure functional programs is easier, because the result is
based solely on the input of the program.

The decision on programming technique depends on the problem. There are
problems, which are nicely suitable for functional programming. And there are
problems where we can not use (or it is hard to use) the functional approach.

However, everything is not black and white. If the programming language permits,
you can write some parts of the program in the functional style and also use the
lock based/free data structures. I believe that the program can benefit from
using pure functions even if the whole program is not written in the functional
style.

The efficiency differs from example to example. We need to measure it and then
act accordingly. However, the rule of thumb is that the lock free data
structures are more efficient than the lock based data structures.

Summary
-------

We explained some terminology of the concurrent/parallel programming. We started
by stating the differences between concurrent and parallel programming. Then we
looked at distributed and shared memory models. We also explored the techniques
for programming in the shared memory model.

Thanks to [Aleš Bizjak](http://www.cs.au.dk/~abizjak/) for reading the draft of
this article!

Links:

* [Wiki: Concurrent computing](https://en.wikipedia.org/wiki/Concurrent_computing)
* [Wiki: Parallel computing](https://en.wikipedia.org/wiki/Parallel_computing)
* [Stackoverflow: difference between concurrent and parallel programming
1](http://stackoverflow.com/questions/1897993/difference-between-concurrent-programming-and-parallel-programming)
* [Stackoverflow: difference between concurrent and parallel programming
2](http://cs.stackexchange.com/questions/19987/difference-between-parallel-and-concurrent-programming)
* [Wiki: Disturbed memory](https://en.wikipedia.org/wiki/Distributed_memory)
* [Wiki: Shared memory](https://en.wikipedia.org/wiki/Shared_memory)
* [Great read about functional programming](http://blog.jenkster.com/2015/12/what-is-functional-programming.html)


