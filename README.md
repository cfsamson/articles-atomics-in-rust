---
description: >-
  Understanding atomics and the memory ordering options when dealing with them
  can help us better understand multithreaded programming and why Rust helps us
  write safe and performant multithreaded code.
---

# Explaining Atomics in Rust

Trying to understand atomics by just reading random articles and the documentation in Rust \(or C++ for that matter\) feels like trying to learn physics by reverse engineering`E=MC^2`. Potentially doable with enough grit, but the ratio of `WTF?/AHA!` is enormous.

I'll give it my best try to explain this for myself and for you in this article. If I succeed the ratio should be `WTF?/AHA! < 1`. Let me know in the [issue tracker of the repository for this article](https://github.com/cfsamson/articles-atomics-in-rust) how we did.

## Multiprocessor programming

When writing code for multiple CPUs there are several subtle things we need to consider. You see, both compilers and CPU's reorder the code we write if they think it will lead to faster execution. In single threaded programs, this is not something we need to consider, but once we start writing multithreaded programs the compiler reordering can get us into trouble.

However, while the compiler ordering is possible to check by looking at the disassembled code, things get much more difficult on systems with multiple CPUs.

When threads are run on different CPUs, the internal reordering of instructions on the CPU can lead to some very hard to debug problems since we mostly observe the side effects of CPU reordering, speculative execution, pipelining and caching. 

I don't think the CPU knows in advance exactly how it's going to run your code either.

{% hint style="warning" %}
The problem atomics solve are related to memory loads and stores. Any reordering of instructions which does not operate on shared memory has no impact we're concerned about here.

One more thing to note is that I consciously use multi processor or multi core programming, instead of multi `threaded`programming. While related in practice on most systems today, _most_ of this book will **not** apply on a _multithreaded_  but _single core_ system.
{% endhint %}

Let's start at the bottom and work our way up to a better understanding.

_I'll skip compiler reordering since it's pretty easy to understand. Just know that your code most likely will not be compiled chronologically the way you write it. However, this only applies to code that can correctly be reordered. Code that depends on previous steps will of course not be reordered at random._

## CPU Caches <a id="cpu-caches"></a>

Normally, a CPU has three levels of caching: L1, L2, and L3. While L2, and L3 are shared between cores, L1 is a per core cache. Our challenges start here.

The L1 cache uses a sort of [MESI caching protocol](https://en.wikipedia.org/wiki/MESI_protocol). While the name might sound mysterious, it's actually pretty simple. It is an acronym for the different states an item in the cache can find themselves in:

```text
These states apply to each cache line in the L1 cache:

(M) Modified - modified (dirty). Need to write back data to main memory.
(E) Exclusive - only exists in this cache. Doesn't ned to be synced (clean).
(S) Shared - might exist in other caches. Is current with main memory (clean).
(I) Invalid - cache line is invalid. Another cache has modified it.
```

Ok, so we can model this for ourselves by thinking that every cache line in the CPU has an enum with four states attached to them.

{% hint style="info" %}
**Does this sound familiar?**

In Rust we have two kind of references `&`shared references, and `&mut`exclusive references. 

It will help you alot if you stop thinking of them as `mutable`and `immutable`references, since this is not true all the time. Atomics and types which allow for interior mutability does indeed break this mental model. Think of them as `exclusive`and `shared`instead.

This does indeed map very well to the `E` and `S`, two of the possible states data can have in the L1 Cache. Modelling this in the language can \(and does\) provide the possibility of optimizations which languages without these semantics can't do.

In Rust, only memory which is `Exclusive`can be modified by default. 

_This means, as long as we don't break the rule and mutate `Shared`references, all Rust programs can assume that the L1 cache on the core they're running on is up to date and does not need any synchronization._

Of curse, many programs needs to share memory between cores to work, but doing so explicitly and with care can lead to better code for running on multiple processors.
{% endhint %}

## Inter-processor Communication

So, if a cache line is invalidated if the data exists in the L1 cache on on a different core \(`Shared`state\) and is modified there, there must be some way for the cores to communicate with each other if they're going to know if the state of this cache line has changed?

Yes there is, however, it's pretty hard to find documentation about the exact details. Each CPU has what we can think of as a _mailbox_.

This mailbox is what's important for us when we're going to talk about atomics later on. 

This mailbox can buffer a certain number of messages. Each message is buffered here to avoid interrupting the CPU all the time and force it to handle every message sent from other cores immediately.

 Now, at some point the CPU checks this mailbox \(I've also seen talks about designs which issues an interrupt to the CPU when it needs to process messages\) and updates it's cache accordingly.

Let's take an example of a cache line which is marked as `Shared`. 

If a CPU modifies this cache line, it is invalidated in the other caches. The core that modified the data sends a message to rest of the CPU's. When the other cores check their _mailbox_ they see that this cache line is now invalid and the state is updated accordingly in each cache on the other cores.

The L1 cache on each core then fetches the correct value from main memory \(or L2/L3 cache\) and sets its state to `Shared`again.

{% hint style="info" %}
It's useful to know that none of these operations by default happens immediately. Both the sending of messages and reading the mailbox can be delayed for an arbitrary amount of time. The processor will by default first focus on performance, secondly on synchronization and cache coherence.
{% endhint %}

## Memory fences

Now that we have some idea of how the CPUs are designed to coordinate between them, we can talk about different memory orderings and what they mean:

In Rust the memory ordering is represented by the [std::sync::atomic::Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) enum, which has 5 possible values:

### Relaxed

No messages are forced to be sent or retrieved & processed from the inbox before this operation. This is therefore the weakest of the possible memory orderings. It implies that the operation does not do any specific synchronization with the other CPUs.

### Acquire

All messages in the inbox of the current CPU are read and processed before the next operation. This means that all cache lines any other CPU has modified, is marked as `Invalid`meaning that we'll need to fetch new values from memory if our operation involves reading such a value. For this exact reason, `Acquire`only makes sense in _load_ operations. **Most** **`atomic`methods in rust which involves stores will panic if you pass inn** **`Acquire`as the memory ordering of a** **`store`operation.**

### Release

All pending messages on the CPU are flushed and sent to the other CPUs mailboxes before we perform the nest operation. This is most likely values which was `Shared`but we have been modified, so they're now marked as `Modified`in the cache of the current CPU. Before we proceed with any operation, we flush the changes made to main memory and sends a message to all the other cores that they'll need to mark this cache line as `Invalid`. `Release`memory ordering only makes sense on `store`operation. **For this reason, and opposite of the** **`Acquire`ordering, most** **`load`methods in Rust will panic if you pass in a** **`Release`ordering.**

### AcqRel

First process all messages in the inbox \(as we do in `Acquire`\), then flush changes and notify the other CPUs, so they can update their caches \(as we do in `Release`\).

### SeqCst

Same as `AcqRel`but it also preserves a sequentially consistent order between operations that are marked with `SeqCst`. It's a bit hard to understand, but think of it like special messages which are marked with a timestamp.

The timestamp is only attached with messages sent to us from a CPU which did an `AcqRel`as a part of `SeqCst`. We only read this timestamp if we are performing a `AcqRel`as a part of a `SeqCst`operation ourselves, and we order these messages chronologically based on their timestamp.

If we have three cores, A, B and C. All performing `SeqCst`operations on the same memory. Since all of them sort the messages chronologically , they will all read the messages in the order from first to last. The three cores can thereby agree on what happened in which order.

One important thing to note is that if there is any `Acquire`, `Release`or `Relaxed`operations on this memory, the sequential consistency is lost since there is no way to know when that operation happened and thereby agree on a total order of operations.

**SeqCst is the strongest of the memory orderings, it also has a slightly higher cost than the others.**

I have a hard time coming up with a good example where this ordering is the only one that solves a certain problem. Most synchronization can be solved by using the weaker orderings.

_However, reasoning about_ _`Acquire`and_ _`Release`in complex scenarios can be hard. If you only use_ _`SeqCst`on a part of memory you'll know that you have the strongest memory ordering and you're likely on the "safe side". It makes working with atomics a lot more convenient._

{% hint style="info" %}
Since these synchronizations happen before the nest operation, and we force the core we're currently running on to synchronize it's cache with the other cores we call these operations `memory fences`or `memory barriers`.
{% endhint %}

## The `lock`CPU instruction prefix

In addition to the memory fences discussed above, using the atomic types in the `std::sync::atomic`module gives access to some important CPU instructions we normally don't see in Rust:

From [Implementing Scalable Atomic Locks for Multi-Core Intel® EM64T and IA32 Architectures](https://software.intel.com/en-us/articles/implementing-scalable-atomic-locks-for-multi-core-intel-em64t-and-ia32-architectures):

_User level locks involve utilizing the atomic instructions of processor to atomically update a memory space. The atomic instructions involve utilizing a lock prefix on the instruction and having the destination operand assigned to a memory address. The following instructions can run atomically with a lock prefix on current Intel processors: ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG..._

Ok, so when we use the methods on atomics like `fetch_add`on [AtomicUsize](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html) it actually the the assembly would instead look something like \(an AT&T dalect\) `lock addq ..., ...`instead of `addq ..., ...`which we'd normally expect.

{% hint style="info" %}
Its a good time to remind yourself now that `atomic`types is not something the CPU knows about. An `AtomicUsize`will be represented by a regular `usize`in the CPU. It's the methods and types on the `atomic`which emits different CPU instructions that matter.
{% endhint %}

### Now what does this `lock`dinstruction prefix do?

It can quickly become a bit technical, but as far as I understand it, an easy way to model this is that it sets the cache line state to `Modified`already when the memory is fetched from the case. This way, from the moment it's fetched from a cores L1 cache it's marked as `Modified`and a message to `invalidate`it in the other caches is sent.

If you consider a "normal" add operation, the `load`operation does not change the state of the cache line. First when the "add" operation is completed the state of this cache line in the L1 cache is changed.

If you see this in relation to the memory fences discussed above you can understand why this subtle change matters.

{% hint style="info" %}
A cache line is most often 64 bytes on a 64 bit system. This can vary based on the exact CPU, but what is important to consider is that the locking mechanisms used if a memory crosses two cache lines is much more expensive and might involve bus locking and other hardware techniques.

Also, atomic operations crossing cache line boundaries have a very varying support based on different architectures. **Presumably this is the major reason that all `atomic`operations is limited to `usize`sized data.**
{% endhint %}

## Conclusion

Are you still there? If so, relax now, we're done for today. Thanks for staying with me and reading through, I do sincerely hope you enjoyed it and got some value out of it.

I fundamentally believe that getting a good mental model around problems you actually work hard to deal with has numerous benefits for both your personal motivation and how you write your code even though you never venture into the `std::sync::atomic`module at all in your daily life.

Until next time!







