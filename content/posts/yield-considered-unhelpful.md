---
title: "Yield Considered Unhelpful"
date: 2021-04-04T14:29:57-04:00
---

I had the idea for this post after reading
[Linus's take on the  yield syscall](https://www.realworldtech.com/forum/?threadid=189711&curpostid=189752).
It's worth a read if you haven't seen it, although I will be summarizing his points.
I also think that the take missed a larger conversation about the APIs that a platform gives its users,
and how that shapes correct user behavior.

## What is 'yield'?

The function in question is [sched_yield](https://linux.die.net/man/2/sched_yield). It is a very simple function
that 'yields' the remainder of your thread's time back to the scheduler. In modern operating systems we have
"preemptive scheduling", which means that the scheduler will interrupt, or preempt, your thread when a certain
amount of time has passed. The scheduler preempts threads so that all of threads in the system can get a turn
to make progress. Preemption stops a CPU-intensive thread from hogging all of the resources in your system.

`yield` is a relic from a non-preemptive scheduler world where threads had to be 'nice' and let the scheduler
know when they are done. A thread that calls `yield` is basically saying "I'm good, there's no more work for
me to do right now. Put me in the back of the list and let all the other threads have a turn on the CPU".
If the scheduler can't interrupt, or preempt, a thread then calling `yield` is the only way that the thread
can return control to the scheduler. In modern OSes, schedulers only gives the thread a certain amount of
time before the scheduler steps in and takes control back. Modern schedulers don't rely on threads calling
`yield` anymore.

## Why is yield unhelpful?

Yield is unhelpful because it doesn't give the scheduler enough information to do its job correctly.

In a modern OS there should be no legitimate reason to call yield. If your thread is 'waiting' for something
to happen, then it should **block** on that thing happening. If your thread is extremely CPU-intensive then it
should **trust** that the scheduler will preempt you and give you and the rest of the system the correct priorities.

Yield is essentially saying "Put me to sleep for a random amount of time", because your thread is being moved
to the back of the scheduler's run-queue. Does being put to sleep for a random amount of time sound like
useful or understandable behavior? You will not be able to get consistent or replicable performance results
if you have `yields` all over your program.

### But I'm spin-waiting for something and I don't want to block the CPU.

I don't really believe that user-space code should use a spin-lock. I think that using a
[fast userspace mutex, or futex](https://en.wikipedia.org/wiki/Futex) will probably give you better performance
characteristics (Futex's don't have to enter the kernel in the "common case" of the lock being uncontested).

That aside, if you're convinced that you need a spinlock "for performance", then why are you yielding? The
scheduler prevents you from hogging all of the resources, so spin away on that lock until you get preempted.
If you check once and then yield, chances are that you will miss what you're waiting for while the
scheduler is running other threads. Given that the Linux scheduler's timeslice is 100ms, in the worst case
you will be late to your event by 100ms * the number of runnable threads on your CPU.

If you're spin-waiting in userspace, the kernel can't help you because the kernel doesn't know what you're
doing. The kernel's job is to help you, you should let it! If you use a futex, the kernel knows what you're
blocked on. The kernel will know immediately when your resource is free again, and if you're high enough
priority you might be run immediately after your resource is free. This shortens the worst-case scheduling
scenario tremendously and it should also give you **predictable** timing.

### But I'm CPU intensive and other things might be more important.

This is the example that Linus gave:

> You'll find various random GUI programs doing that because they are threaded, and one thread does things
> like update the screen, while another thread does calculations. The calculation loop (which is still important,
> just not latency-critical) might have "sched_yield() in it as a cheap way of saying
> "maybe there's a more important UI event going on".

I agree that many systems have been built like this, and doing this might be the simplest way to get a system
off the ground. However, this is not a predictable or understandable system. The problem with this approach is
that decisions about scheduling are being made at the wrong level.

As a process, you should **trust** the scheduler. If you are being run, you should trust that you are the highest
priority thing being run, and you should do your work. Please don't try and "guess" at the schedulers job, because
you don't have any of the information that the scheduler has. You don't know how much load the system is under.
You don't know what priorities the other threads have. You don't know who else is runnable at this exact moment.
The scheduler has all of this information! And the scheduler chose you to have the CPU! Please don't hand the
CPU back if you have work that you can do.

## In conclusion.

* Think of `yield` as saying "I would like to sleep for a random amount of time".
* Please don't use `yield` in your code to be 'nice'. Trust the scheduler.
* Please let the kernel know what you're waiting on by using a `futex` or other kernel primitive. Help the kernel help you.
