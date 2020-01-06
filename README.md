---
description: >-
  Understanding atomics and the memory ordering options when dealing with them
  can help us better understand multithreaded programming and why Rust helps us
  write safe and performant multithreaded code.
---

# Explaining Atomics in Rust

Trying to understand atomics by just reading random articles and the documentation in Rust \(or C++ for that matter\) feels like trying to learn physics by reverse engineering`E=MC^2`.

I'll give it my best try to explain this for myself and for you in this article. If I succeed the ratio should be `WTF?/AHA! < 1`. Let me know in the [issue tracker of the repository for this article](https://github.com/cfsamson/articles-atomics-in-rust) how we did.

## Multiprocessor programming

When writing code for multiple CPUs there are several subtle things we need to consider. You see, both compilers and CPU's reorder the code we write if they think it will lead to faster execution. In single threaded programs, this is not something we need to consider, but once we start writing multithreaded programs the compiler reordering can get us into trouble.

However, while the compiler ordering is possible to check by looking at the disassembled code, things get much more difficult on systems with multiple CPUs.

When threads are run on different CPUs, the internal reordering of instructions on the CPU can lead to some very hard to debug problems since we mostly observe the side effects of CPU reordering, speculative execution, pipelining and caching. 

I don't think the CPU knows in advance exactly how it's going to run your code either.

{% hint style="warning" %}
The problem atomics solve are related to memory loads and stores. Any reordering of instructions which does not operate on shared memory has no impact we're concerned about here.

**I'll be using one main reference here unless I state otherwise:** [Intel® 64 and IA-32 Architectures Software Developer’s Manual Volume 3A](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf). So when I refer to Chapter x in the Intel Developer manual, I refer to this document.
{% endhint %}

Let's start at the bottom and work our way up to a better understanding.

_I'll skip compiler reordering since it's pretty easy to understand. Just know that your code most likely will not be compiled chronologically the way you write it. However, this only applies to code that can correctly be reordered. Code that depends on previous steps will of course not be reordered at random._

## Memory ordering vs Atomic instructions <a id="cpu-caches"></a>

A typical atomic operation in rust looks like this:

```rust
atomic_val.fetch_add(1, Ordering::Relaxed);
```

There are two things stand out as different from a regular add operation.

1. `fetch_add`is a special method on the atomic type
2. `Ordering::Relaxed`specifies the memory ordering

Now, to make things clear from the start. Atomic instructions are always atomic. The memory ordering is primarily information to the compiler indicating what it can and can't do.

There is also one more thing to note.

There are two main models for a CPu to treat memory. Either strongly ordered, or weakly ordered. 

Most current desktop CPUs from AMD and Intel uses strong ordering. That means that the CPU itself gives some guarantees about not reordering certain operations. Some examples of this guarantee can be found in Intels Developer Manual, chapter 8.2.2:

> * Reads are not reordered with other reads.
> * Writes are not reordered with older reads.
> * Writes to memory are not reordered with other writes \(with some exceptions\)

There are more guarantees as well, however there is also one important non-guarantee mentioned:

> Reads may be **reordered with older writes to different locations** but not with older writes to the same location

_For now, just remember this non-guarantee._

A weakly ordered CPU can choose to reorder instructions much more loosely. Since the code we write might be compiled to such platforms, the semantics in Rust needs to accommodate for that. Also, the chapter regarding memory ordering does especially apply to weakly ordered platforms since there is more for us to consider on such a system.



## CPU Caches <a id="cpu-caches"></a>

Normally, a CPU has three levels of caching: L1, L2, and L3. While L2, and L3 are shared between cores, L1 is a per core cache. Our challenges start here.

The L1 cache uses a sort of [MESI caching protocol](https://en.wikipedia.org/wiki/MESI_protocol). While the name might sound mysterious, it's actually pretty simple. It is an acronym for the different states an item in the cache can find themselves in:

```text
These states apply to each cache line in the L1 cache:

(M) Modified - modified (dirty). Need to write back data to main memory.
(E) Exclusive - only exists in this cache. Doesn't need to be synced (clean).
(S) Shared - might exist in other caches. Is current with main memory (clean).
(I) Invalid - cache line is invalid. Another cache has modified it.
```

{% hint style="info" %}
See chapter 11.4 in [Intels Developer Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf) for more information about the cache states.
{% endhint %}

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

So, if we really do need to access and change memory which is `Shared` how does the other cores know that their L1 cache is invalid? 

Well, a cache line is invalidated if the data exists in the L1 cache on a different core \(remember, it's in the`Shared`state\) and is modified there. To actually inform the other cores that their cached data is invalid there must be some way for the cores to communicate with each other, right?

Yes there is, however, it's pretty hard to find documentation about the exact details. Each CPU has what we can think of as a _mailbox_.

This mailbox is what's important for us when we're going to talk about atomics later on. 

This mailbox can buffer a certain number of messages. Each message is buffered here to avoid interrupting the CPU all the time and force it to handle every message sent from other cores immediately.

 Now, at some point the CPU checks this mailbox \(I've also seen talks about designs which issues an interrupt to the CPU when it needs to process messages\) and updates its cache accordingly.

Let's take an example of a cache line which is marked as `Shared`. 

If a CPU modifies this cache line, it is invalidated in the other caches. The core that modified the data sends a message to rest of the CPU's. When the other cores check their _mailbox_ they see that this cache line is now invalid and the state is updated accordingly in each cache on the other cores.

The L1 cache on each core then fetches the correct value from main memory \(or L2/L3 cache\) and sets its state to `Shared`again.

{% hint style="info" %}
It's useful to know that none of these operations by default happens immediately. Both the sending of messages and reading the mailbox can be delayed for an arbitrary amount of time. The processor will by default first focus on performance, secondly on synchronization and cache coherence.
{% endhint %}

## Memory ordering

Now that we have some idea of how the CPUs are designed to coordinate between them, we can talk about different memory orderings and what they mean.

In Rust the memory ordering is represented by the [std::sync::atomic::Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) enum, which has 5 possible values.

{% hint style="info" %}
On strongly ordered platforms, many of these orderings will emit the same exact code depending on the situation since the CPU has some guarantees we can rely on. However, they will prevent the compiler from doing unexpected reordering of our code which is the important part to know. The memory ordering we want to have is information we give the compiler to make our intentions clear. 

The atomic _instructions_ we use on this memory on the other hand does instruct the CPU to treat operations on this memory in a specific way.

On a weakly ordered system, these semantics might cause different explicit memory fences to be set up.
{% endhint %}

### A mental model

While this is pretty hard to do in Rust in practice due to it's type system \(we have no way to get a pointer to an `Atomic`for example\), I find it useful to imagine an observer-core. This observer core is interested in the same memory as we perform atomic operations on for some reason, so we'll divide each model i two. How it looks on the core it's running, and how it looks from an observer-core.

Remember that Rust inherits its memory model for atomics from C++ 20, and C copies their model from C++ as well. In these languages, mixing atomic operations and "normal" memory access on the same memory is perfectly doable.

### Relaxed

**On the current CPU:**  
Relaxed memory ordering on atomics will prevent the compiler from reordering these instructions themselves but on weakly ordered CPUs it might reorder all other memory access. It's OK if you only increment a counter but might get you into trouble if you use a flag to implement a spin-lock for example since you can't trust that "normal" memory access before and after the flag is set, is not reordered.

**On the observer CPU:**  
Any [locked](./#the-lockcpu-instruction-prefix) atomic operation on any memory location triggers the cache coherency mechanism and will force the observer cores L1 cache to update this memory location as well so at that point, the observer core will see the new value. However, a `Atomic::store`on itself is not a locking operation so unless the observer core has processed it's "mailbox" it is perfectly possible to fetch an outdated value from the cache. Since the compiler might reorder any other memory as well, the observer thread might observe operations in a different order than the "program order" \(as we wrote them\).

This is therefore the weakest of the possible memory orderings. It implies that the operation does not do any specific synchronization with the other CPUs.

{% hint style="info" %}
One caveat is that on strongly ordered systems, these operations will most likely be run in the same manner as with `Acquire/Release`. Proving or observing any differences on such a system might be impossible. However, on weakly ordered systems, this will matter. 

Take a [look at this article](https://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu/) where they run the same code on both a strongly ordered CPU and a weakly ordered CPU to see the differences.
{% endhint %}

### Acquire

**On the current CPU:**  
Any memory operation \(that means anything manipulating memory locations, not only `Atomics`in Rust\) that is written after the `Acquire`access stays after it. It's meant to be paired with a `Release`memory ordering flag. Any locking operation will still be visible on all other threads.

**On the observer CPU:**  
Any locking operation on the `Atomic`type in Rust will be visible due to the cache coherency mechanism. However, as with `Relaxed`, `Atomic::store`is not a locking operation. However, the observing core will never see any memory operation written after a locking `Acquire`operation happen before it. 

`Acquire` is often used to write locks, where some operations need to stay after the successful acquisition of the lock. For this exact reason, `Acquire`only makes sense in _load_ operations. **Most** **`atomic`methods in rust which involves stores will panic if you pass inn** **`Acquire`as the memory ordering of a** **`store`operation.**

### Release

**On the current CPU:**  
In contrast to `Acquire`, any memory operation written before the `Release`memory ordering flag stays before it. It's meant to be paired with an `Acquire`memory ordering flag. As with Atomics. Any locking operation will still be visible on all other threads, but often a `Release`store will be non-locking, on weakly ordered CPUs it might however insert a [memory fence](https://doc.rust-lang.org/std/sync/atomic/fn.fence.html) to assert that the CPU doesn't reorder these instructions since they're non-locking \(which means reordering is something the CPU might do\). This is partially what makes it weaker than `SeqCst`but also more performant.

**On the observer CPU:**  
Any locking operation on the `Atomic`type in Rust will be visible due to the cache coherency mechanism. However, as with `Relaxed`, `Atomic::store`is not a locking operation. However, the observing core will never see any memory operation written before a locking `Acquire`operation happen after it. 

`Release` is often used together with `Acquire`to write locks. For a lock to function some operations need to stay after the successful acquisition of the lock, but before the lock is released.  **For this reason, and opposite of the** **`Acquire`ordering, most** **`load`methods in Rust will panic if you pass in a** **`Release`ordering.**

### AcqRel

This is intended to be used on operations which both loads and stores a value. On strongly ordered systems, this is most likely the default behavior, but on weakly ordered systems a memory fence might be used to prevent the compiler from reordering the instructions. This operation acts more or less like a fence in Rust. Memory operations that is written before it will not me reordered across this boundary, and memory operations which is written after it will not be reordered before it.

{% hint style="info" %}
It's worth noting that `Acquire/Release`often comes for free on strongly ordered systems, so there is absolutely no extra cost of using this kind of memory ordering.
{% endhint %}

### SeqCst

Same as `AcqRel`but it also preserves a sequentially consistent order between operations that are marked with `SeqCst`. It's a bit hard to understand, but it's a good segway to our next topic concerning `atomic instructions`.

Let's consider `SeqCst` in contrast to an `Acquire/Release`operation.

**I'll use this playground example to explain:**

{% embed url="https://play.rust-lang.org/?version=stable&mode=release&edition=2018&gist=7548a9c7c4e59c0e8d6fafd46e093e67" %}

{% hint style="info" %}
Make sure you have selected `Release`as build option and the choose `Show Assembly`. The relevant part of the assembly is outputted in section `LBB0_2`using the current compiler \(1.40\) so you can search for that.
{% endhint %}

The code is pretty simple. We acquire a flag and change it atomically, and when successful we increase a counter \(un-atomically\), and then we change the flag back. The complexity here stems from having a smart compiler that insists on pre-calculating everything on `release`builds and basically change the program dramatically.

If we first load our flag using a locking instruction \(which `compare_and_swap`is\) we know that we have an updated view on **all** memory and that any modifications we do are visible in the L1 caches where it's referenced \(this memory location is invalid now, so it will be fetched from memory on next access\).

{% hint style="info" %}
The observer thread will also see this as the first thing that happens.
{% endhint %}

When we later on use a store operation to change this flag back \(which doesn't require us to read the most current memory value\), the `store`operation doesn't need to happen atomically at all since we know that the next time this value will be read, the locking instruction will make sure we read the updated value.

**The outputted assembly using `Acquire/Release`will look like this:**

```rust
lock		cmpxchgb	%r14b, playground::LOCKED(%rip)
jne	.LBB0_2
movq	playground::COUNTER(%rip), %r12
addq	$1, %r12
movq	%r12, playground::COUNTER(%rip)
movb	$0, playground::LOCKED(%rip)
```

If this means nothing to you, just note that `lock cmpxchgb`is an [_locking_ ](./#the-lockcpu-instruction-prefix)operation, it reads a flag and changes it's value if a certain condition is met. 

This operation will involve the CPU's [cache coherency mechanism](https://en.wikipedia.org/wiki/Cache_coherence) to make sure no other CPU accesses this data in their caches and once it's updated, all other instances in other caches is `Invalidated`. This instruction also makes sure our cache is current when we start the operation \(all messages processed\).

{% hint style="info" %}
This can be derived from chapter 8.2.5 in [Intels Developer Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf) which states:

> Synchronization mechanisms in multiple-processor systems may depend upon a strong memory-ordering model. Here, a program can use a locking instruction such as the XCHG instruction or the LOCK prefix to ensure that a read-modify-write operation on memory is carried out atomically. Locking operations typically operate like I/O operations in that they wait for all previous instructions to complete and for all buffered writes to drain to memory \(see Section 8.1.2, “Bus Locking”\).
{% endhint %}

Now the store operation which used `Release`memory ordering is `movb $0, playground::LOCKED(%rip)` which is not an atomic operation at all. We know that the next time this value is read on another CPU which use `Acuire/Release`on this memory they will see the change since they use a [locked](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf) instruction which makes sure the result of that write operation is visible in all caches.

{% hint style="info" %}
The observing core \(or any other core for that matter\) might not get a perfectly chronological view of events. If two cores are acquiring and releasing this flag. The observer core might see thread 1 change the flag to `true`, then core 2 set the flag to `true`and then both core 1 and core 2 setting the flag to false.

There is no agreement on which order these events happen since the `mov`instruction is a normal store and thereby, each core will only get notified through their "mailbox". The mailbox remember, will in time give information about what has happened, but not in any order we can rely on.
{% endhint %}

If we change our `test` function in the code I linked to in the playground to:

```rust
pub fn test(inc: usize) -> usize {
    while LOCKED.compare_and_swap(false, true, Ordering::SeqCst) {}
    unsafe { COUNTER += inc };
    LOCKED.store(false, Ordering::SeqCst);
    unsafe { COUNTER }
}
```

We get the following assembly output:

```rust
xorl	%eax, %eax
lock		cmpxchgb	%r14b, playground::LOCKED(%rip)
jne	.LBB0_2
movq	playground::COUNTER(%rip), %r12
addq	$1, %r12
movq	%r12, playground::COUNTER(%rip)
xorl	%eax, %eax
xchgb	%al, playground::LOCKED(%rip)    # notice this
```

The interesting change here is the store operation `xchgb %al, playground::LOCKED(%rip)`. In contrast to the `mov`operation we had previously, this is an atomic operation \(`xchg`has an [implicit `lock`prefix](https://en.wikibooks.org/wiki/X86_Assembly/Data_Transfer)\).  

Since the `xchg`instruction is a [locked ](./#the-lockcpu-instruction-prefix)instruction when it refers to memory \(see chapter 8.1.2.1 in [Intels Developer Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf)\),  it will make sure all cache lines on other cores referring to the same memory is invalidated.

{% hint style="info" %}
For our observing core this is a big change. Since the store operation is also a `locking` instruction in this case, the cache coherence mechanism will invalidate the value we have in the L1 cache, so the next time we read it will fetch the updated value from memory.

So the observer will see core 1 change the flag to `true`, then later on change it back to `false`, and the same for core 2. In fact, all threads that uses this memory location will observe the changes in a sequentially consitstent order.
{% endhint %}

One important thing to note is that if there is any `Acquire`, `Release`or `Relaxed`operations on this memory, the sequential consistency is lost since there is no way to know when that operation happened and thereby agree on a total order of operations.

**SeqCst is the strongest of the memory orderings, it also has a slightly higher cost than the others.**

{% hint style="info" %}
You can see an example of why above since every atomic instruction has an overhead of involving the CPUs [cache coherency mechanism](https://en.wikipedia.org/wiki/Cache_coherence) and locking the memory location in the other caches. The fewer such instructions we need while still having a correct program, the better performance.
{% endhint %}

## The `lock`CPU instruction prefix

In addition to the memory fences discussed above, using the atomic types in the `std::sync::atomic`module gives access to some important CPU instructions we normally don't see in Rust:

From [Implementing Scalable Atomic Locks for Multi-Core Intel® EM64T and IA32 Architectures](https://software.intel.com/en-us/articles/implementing-scalable-atomic-locks-for-multi-core-intel-em64t-and-ia32-architectures):

_User level locks involve utilizing the atomic instructions of processor to atomically update a memory space. The atomic instructions involve utilizing a lock prefix on the instruction and having the destination operand assigned to a memory address. The following instructions can run atomically with a lock prefix on current Intel processors: ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG..._

Ok, so when we use the methods on atomics like `fetch_add`on [AtomicUsize](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html) it actually changes the instructions we use to add the two numbers on the CPU. The assembly would instead look something like \(an AT&T dialect\) `lock addq ..., ...`instead of `addq ..., ...`which we'd normally expect.

{% hint style="info" %}
Its a good time to remind yourself now that `atomic`types is not something the CPU knows about. An `AtomicUsize`will be represented by a regular `usize`in the CPU. It's the methods and types on the `atomic`which emits different CPU instructions that matter.
{% endhint %}

### Now what does this `lock`instruction prefix do?

It can quickly become a bit technical, but as far as I understand it, an easy way to model this is that it sets the cache line state to `Modified`already when the memory is fetched from the cache. 

This way, from the moment it's fetched from a cores L1 cache it's marked as `Modified` . The processor uses its [cache coherence mechanism](https://en.wikipedia.org/wiki/Cache_coherence) to make sure the state is updated to `Invalid`on all other caches where it exists - even though they've not yet process all of their messages in their mailboxes yet.

If message passing is the normal way of synchronizing changes, the `locked`instruction \(and other memory-ordering or serializing instructions\) involves a more expensive and more powerful mechanism which bypasses the message passing, locks the cache lines on the other caches \(so no load or store operation can happen when it's in progress\) and sets them as invalid accordingly which forces the cache to fetch an updated value from memory.

{% hint style="info" %}
If you're interested in reading more about this then take a look at chapter 8 of the [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf).
{% endhint %}

{% hint style="info" %}
A cache line is most often 64 bytes on a 64 bit system. This can vary based on the exact CPU, but what is important to consider is that the locking mechanisms used if a memory crosses two cache lines is much more expensive and might involve bus locking and other hardware techniques.

Also, atomic operations crossing cache line boundaries have a very varying support based on different architectures. **Presumably this is the major reason that all `atomic`operations is limited to `usize`sized data.**
{% endhint %}

## Conclusion

Are you still there? If so, relax now, we're done for today. Thanks for staying with me and reading through, I do sincerely hope you enjoyed it and got some value out of it.

I fundamentally believe that getting a good mental model around problems you actually work hard to deal with has numerous benefits for both your personal motivation and how you write your code even though you never venture into the `std::sync::atomic`module at all in your daily life.

Until next time!







