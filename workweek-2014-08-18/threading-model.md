# 2014-08-19 Threading model

- nmatsakis: I feel like the only thing we can call stable for 1.0 is libnative, the system-level threading model
- nmatsakis: It would make sense for us to say, this is what we're offering for now, and we'd like to push the development of efficient green-threading, but they will come later/as a community effort.
- nmatsakis: It'd be really cool if we could expose the coroutine aspect of our green-threading infrastructure, but I'm not sure we want to keep/expose libgreen
- acrichto: My current perspective: libnative, librustuv, libgreen are unstable and people shouldn't use them directly
- acrichto: I'm fine saying that our model is task = thread unless you explicitly opt-out
- pcwalton: We could put libgreen into Cargo
- acrichto: But libgreen has a huge amount of testing with libstd. That'd be lost in Cargo
- acrichto: It's super-useful to have these tests
- wycats: We could still test against the cargo lib. Leave all the traits that allow libgreen to exist
- nmatsakis: If, on the release channel you can't use libgreen...
- wycats: But you'd want people to be able to use it from Cargo
- acrichto: Yes. It's mature, but not really stable.
- nmatsakis: You've said before that our current channels work across both kinds of schedulers, but it's not clear this is extensible to other schedulers or would work in the Cargo version -- or whether we even want that
- nmatsakis: Maybe we just have to give up that special status for channels, and you have to figure out a different way to interface green and native threads
- acrichto: Basically we have just de- and re-scheduling hooks
- nmatsakis: So in principle, channels should work with other schedulers?
- acrichto: I think it probably assumes that these are actually concurrent
- brson: The bigger problem is IO. The maintenance burden here is astronomical, and as you add other IO models, this is going to get worse. I think we should just deprecate these publically. There's also a lot of overhead -- including vtable and allocations
- nmatsakis: I was envisioning that writing a scheduler in userland means supplying the scheduler, and base reader/writers for files
- acrichto: That's basically what we have today
- acrichto: The problem is that every IO operation is a virtual call, and every IO object creation is a box, and every program in the world has all of the native io, because there's a single global vtable. That's part of why our executable sizes are growing, all of the IO stuff gets included
- acrichto: Has anyone complained about the performance?
- pcwalton: People complain about binary sizes
- nmatsakis: I feel like IO is a canonical example where OO abstraction works nicely. What's the counter-proposal?
- pcwalton: Link to libnative
- acrichto: The bigger problem is, because it's a vtable, you always get all of it -- can't optimize away
- brson: Basically, the interface includes everything that can block
- nmatsakis: It seems like there's room for a trait with a smaller interface, just providing reader/writers basically
- wycats: So you have to use a different IO library if you use a different scheduler?
- acrichto: If I create an IO object on a native thread and send that to a green thread that uses it, the green thread gets blocked

... missed ...

- acrichto: The change I'd make if we remove libgreen is to move libnative under libstd. We'd have no virtual calls, no boxing, no nothing. Even tasks don't need a trait; we'd have to bake in mutexes and condvars.
- nmatsakis: I think that's kind of what I want
- nmatsakis: When I write a scheduler, I might want to expose readers/writers and other blocking primitives that will work properly with my scheduler
- brson: I don't see how you can have a single reader/writer impl per task model
- nmatsakis: Maybe I mean IO Streams
- wycats: I want nonblocking IO
- pcwalton: Doesn't work on Windows
- nmatsakis: Why isn't this just exposing underlying system stuff, as we discussed earlier?
- acrichto: If we did drop libgreen, it's probably more feasible
- wycats: Also, you can do Windows if you do your own buffering
- nmatsakis: Does this have to go into libstd now?
- pcwalton: I think there shouldn't be an abstraction for nonblocking IO. 
- wycats: What about Java's NIO?
- pcwalton: What does it do for Windows?
- wycats: Not sure
- pcwalton: I would much rather have people writing Apache-style apps and only do nonblocking if they know they need the performance and they're on Unix
- nmatsakis: It seems like the right answer isn't clear; why not just let community libraries explore this space for now?
- wycats: Seems OK, as long as enough is exposed to do this
- pcwalton: Here's my worry. We expose select. We bless it. Lots of unixy people buy into it an write lots of libraries using it. Then, a bunch of game devs come along and try to use this ecosystem -- they're all on Windows, and things won't work for them. Games are very network heavy these days.
- pcwalton: I'd rather have the ecosystem be built on thread-per-connection
- nmatsakis: How can we force people to do this? When you say "bless" select, you just mean expose a binding? That's all I'm proposing...
- pcwalton: The question is, do we have the power to influence the community via library design? I think Rust would be ill-served if we encourage things that are bad on Windows
...
- wycats: libevent is mega buggy. libuv just doesn't care about Windows
- nmatsakis: I'm confused; I thought we were *NOT* providing an abstraction
- wycats: Maybe Linux threads are awesome now and people are happy to do Apache-style servers. That doesn't seem the case right now
- pcwalton: That's what Go does
- nmatsakis: But they have lightweight threads
- pcwalton: I'm objecting to having non-blocking (select) IO as the preferred way to do IO in Rust. We should expose it, but not prefer it; we don't want to shut Windows out in the cold
- wycats: We definitely need to care about Windows. It's totally fine if the FAQ recommends blocking IO. But I think that high performance networking code wants to multiplex. It's not a 1.0 concern, but I don't want our official long-term stance to be blocking only
- pcwalton: The problem is you can't write a single app that works great on both models
- wycats: I think emulating nonblocking on Windows is plausible; I think the code you end up with is not so far away from what you do with iocp anyway.
- pcwalton: I'd be happy if there was an efficient way to implement select on top of iocp
- wycats: I think you want that programming model because you lose little on Windows, and gain a lot on Unix
- pcwalton: If that is true, I'm all in favor. I just want to confirm
- wycats: The point for right now is just that our policy shouldn't be restricted to blocking io
- nmatsakis: It seems our policy should be: we're exposing the system level right now. We haven't blessed a higher-level abstraction yet, and we'll work on it after 1.0
- acrichto: Are we headed toward just giving raw syscall access? Right now, we have a very safe Rust interface over things like TCP sockets
- wycats: Once you're trying to coordinate across multiple things, synchronous mode gets a lot harder
... missed ...
- wycats: I think Ruby has a good story here
- acrichto: A lot of the current complexity is for allowing access to readers/writers across tasks. Being able to clone things, to close things remotely, etc
... missed ...

# Summarizing action items:

- We leave open the door to exploring async IO in the future
- We continue to have a convenient/nice sync abstraction
- Cut away libgreen, tie std::io to libnative
- Expose nicer interfaces on top of libc, refactor std::io to use that

