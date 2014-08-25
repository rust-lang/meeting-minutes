# 2014-08-19 Library Experience Report

- wycats: talking about cargo + skylight
- wycats: we're building a tool for people who have rails apps to tell what's going slow in their application.
- wycats: typically, there are a bunch of processes with a load balancer. We need to collect info from each of these processes and send them to our server.
- wycats: The traditional way to do this is, you put an agent on each of the processes, and have each agent talk to the server. But then there's a lot of overhead (making http requests, etc)
- wycats: Plus, Ruby is single-threaded, so you're eating up valuable CPU
- wycats: Instead, we have one process that collects from small agents locally, and then batches that information to the server.
- wycats: Initially, all of this was written in Ruby, which it's worth noting has pretty good Unix support (forking, etc) and we only support Linux, so we were able to take a lot of advantage of that.
- wycats: Initially, we replaced the embedded agents with Rust code. Our goal was to cut down on memory footprint so that we didn't bloat Ruby code we were instrumenting.
- wycats: But we still had the batching process in Ruby, which still had a high footprint.
- wycats: In the last couple of months, we rewrote that batching process in Rust, and that ended up being much harder.
- wycats: The batching process doesn't have to worry about FFI -- more like a traditional socket server. But the Rust IO story isn't as complete, and that's where the pain was.

## Moving the embedded agents to Rust

- wycats: For the embedded agents, the problem was with FFI. The FFI boundary doesn't work well with failure. You have to be running Rust code within a task. You have to take some extra care to wrap the FFI code with a task boundary, so that you don't abort on failure. And you definitely don't want each FFI call to spawn a thread.
- wycats: We ended up hacking around this by using low-level Rust runtime APIs to hijack the current thread, and then use libnative etc to catch task failure and report it as an exception within Ruby
- wycats: Before things became Result-oriented, we've been able to report better errors on the Ruby side. We really want rich error reporting, since this code runs in a wide variety of environments.
- wycats: Largely, we solved this problem early, and our solution has been working well. Basically, what we'd like is just to make this kind of FFI API to be more official and useable without exposing the internals of Rust.
- pcwalton: I very much agree. There should be just a line of code that you write to get into a Rust native thread
- wycats: One thing people said to me early on is: "what you want is no-standard-lib mode" But actually, that's not a problem; I just want to avoid spawning threads. I want the standard library.
- wycats: Any embedding in a dynamic language wants libstd, but not threading.
- pcwalton: Agreed
... some details about exactly how the FFI setup works ...
- wycats: We encode Rust's ownership discipline by nulling out internal pointers, with dynamic checking that gives you a Ruby exception if you try to use it
- wycats: For FFI to dynamic langs, you really want C macros that build up Rust's ownership discipline
- wycats: A recent issue we've hit is: there's no skipping API for readers, which has been a problem as we've tried to optimize. Today, you have to read all the bytes into a buffer, which is a waste. But that's minor.

## Moving the batching server to Rust

- wycats: The original reason I started using Rust came out of trying to lower the memory footprint of Ruby code.
- wycats: Initially, we thought it'd be really easy to move the batching server to Rust
- wycats: Basically, we do some (racy) initialization, and then we start listening on a socket. The agents are every so often sending traces. Then you just batch. In principle, it's really simple, but it turned out to be painful in Rust.
- wycats: For one, there's no "flock" in Rust. There's no way to have an IPC that has some kind of lock -- you have to drop down into C code. Carl wrote a library that implements enough of flock to get the job done.
- wycats: Essentially, this is a pretty simple server that needs to do some multiplexing, including timeouts, and also allow live updates. But Rust's IO makes multiplexing very difficult
- pcwalton: Go has the same limitations, but people like its IO. Why is Rust different?
- wycats: There were a couple of problems. For one, you want to use a bounded channel here. But we lost a day due to a bug -- basically because these channels are not heavily used and therefore not being tested. Overall, bugs are a problem
- wycats: I want to select on a few channels with a timeout. I suspect they're easier in Go.
- pcwalton: In Go, select is built-in and you have to use channels. So you'd have to spawn a task with a timeout that sends a message.
- wycats: The problem is that the lack of select with a timeout leaks into the rest of your program. Whereas with Ruby, we just use IO.select and everything just works.
- pcwalton: We could add select with timeouts -- that won't even need language changes
- nmatsakis: Selecting on ports and channels or IO?
- wycats: You can convert freely in Rust.
- nmatsakis: What I"m getting at is: select over file descriptors is built in to the OS, is that what you want?
- wycats: If we could've ported it directly (by getting the fd out), that's what we would've done.
- acrichto: Channels in isolation work well, channels together do not.
- wycats: There are a lot of workarounds for individual problems, but when you try to compose these workarounds, it gets very complex.
- wycats: In general, timeouts are a struggle. There's not a clear rule in Rust that if you make some IO abstraction, you need to provide a timeout.
- wycats: We ended up finding a solution that's probably what you'd do in Go. But basically, the port took a lot longer than expected, and not because of ownership but rather because of IO
- wycats: I just want to do select over channels with a timeout.
- wycats: It seems like this is a systems language, and we should be exposing the underlying system capabilities like select
- wycats: Maybe we can do something like Java's solution, which is cross-platform but works extremely well, but we're not there yet. Because the high-level abstraction isn't done, you're forced to work low-level, but you can't get at those low-level details, so you're stuck.
- dherman: We were talking about this for Parallel JS is that you work your way up to abstractions by starting with low-level primitives and then building up abstractions.
- pcwalton: This has been tried many times for IO and failed. libevent, etc
- wycats: We should probably just expose enough of the unix guts so that people can get what they need
- pcwalton: I think for web servers, what we have today is fine
- wycats: We don't have signal handling, we don't have timeouts
- pcwalton: We should fix timeout, and Windows doesn't have signal handling.
- wycats: Maybe, but web servers should handle ctrl+C...
- pcwalton: I'm not opposed to fileno. When talking about a systems language here, we're doing what C did. There's stdio.h that provides functionality like opening and reading files. We've extended this with TCP sockets etc. Then there's another C standard called POSIX (basically Unix only), it's different, but there's a bridge between them, called fileno, where you can convert from a stdio to posix. But we should focus on having an abstraction.
- wycats: Agreed. I want fileno, and I want more bindings. We don't have a thing that compiles against .h files, and it's expensive to do the bindings. We need a libposix and liblinux, etc
- pcwalton: I'm in agreement. We already have liblibc. It needs to be better and more Rust-y
- dherman: It's a bridge between high-level and low-level IO on your platform
- nmatsakis: So, if you want it, you can use the low-level routines, but if you can get away with it you can work cross-platform
- pcwalton: I think it's amazing that we've gotten as far as we have in the cross-platform case
- nmatsakis: I don't think anyone is arguing that we should drop that. But we need a platform bridge.
- wycats: I think the readiness versus completion models of IO are very hard to bridge. You have to expose the raw models
- pcwalton: Basically, we allow you to create things that work like Apache where you spawn a thread. We've done a lot of work to make this work well in most cases, it's competitive with Go, but it's not maximally performant. We should allow you to drop down when you need the maximal perf
- wycats: Abstractions over the Unix ecosystem works OK because they all know how to work on the readiness model of IO
- wycats: Basically, we should re-open the floodgates for PRs with platform-dependent bindings. You should be able to accept a socket, get the actual socket and give it to a low-level library

## Limitations on traits

- wycats: Basically, the problems outlined in the original trait reform RFC
- wycats: I just want to make a pitch for specialization. I'll use Path as an example.
- wycats: Path does not implement Show, and sometimes you want to write userland code that can print it for debugging. We wanted a new trait called Inspect, where if you implement Show you get it, and we can spot fix the few missing Show cases. Right now, it's impossible to do it, and it's even impossible to do it with the current trait work

```
impl<T:Show> Inspect for T { ... }
impl Inspect for Path { ... }
```

- pcwalton: Perhaps we could do this with some sort of qualified defaulting
- nmatsakis: I think the big difference would be that, if you had nameable defaults, you'd still have to declare the impl for every type that might participate. You wouldn't get an impl of Inspect automatically, but you wouldn't have to write the definitions.
- wycats: This doesn't work for a general purpose library. You don't want to force all consumers to implement Inspect
- pcwalton: I understand the justification for specialization, but I'm nervous about losing reasoning principles. Right now, if an impl applies, I know it's what will be called
- wycats: Another example is if we provided a skip method on readers, with an inefficient default impl but then optimized for other cases
- wycats: You can do this within core, of course, but you can't do it yourself externally.
- wycats: So in general, you need something like this for extensibility.
- pnkfelix: The path example is disjoint
- wycats: Yeah, but in general it might not be. Sometimes you really want to do something custom.
- wycats: For the skip example, couldn't you just define a subtrait with a default impl?
- nmatsakis: You'd still have to explicitly opt-in to implementing it
- wycats: Right, which means that you're leaking things
- nmatsakis: Basically, specialization lets you hide distinctions behind an abstraction
- pnkfelix: It seems like there's a tension here between hiding the distinction and pcwalton's desires
- pcwalton: Yes, we're fundamentally opposed
- nmatsakis: I think we can both be satisfied
- dherman: But this is post-1.0 anyway?
- nmatsakis: Yes, I want to defer this
- pcwalton: It's crucial that you can figure out what code will be executed without checking every impl in the program
- dherman: What bothers me is that sounds exactly like the argument against OO or higher-order functions
- pcwalton: That's why I made an analogy to final. If you're designing an OO language from scratch, generally you don't want all methods to be virtual. In modern Java programming, style is to do final by default
- wycats: I think dynamic languages are loose enough that people have made peace with this issue, so maybe things are different there. You can always use a REPL to help figure things out.
- zwarich: To me the problems with specialization are: the error messages (cf C++ template error messages). Also, can you still do specialization with separate typechecking?
- nmatsakis: Certainly the proposal I have in mind is separately checkable and keeps today's errors. We would guarantee at typecheck time that there is an impl, but we may not know exactly what it is until monomorphization. 
- nmatsakis: The other problem is that .clone() and trait objects are completely incompatible. 

```
trait Clone {
    fn clone(&self) -> Self; // <-- not legal in trait objects
}
```

```
let r = Box<Reader+Clone>;
r.clone(); 
```

```
trait CloneableReader : Clone  { ... }

#[deriving(Clone)]
struct Foo {
    reader: Box<CloneableReader>
}
```

- wycats: it's painful because cloning is used for general-purpose workarounds to certain ownership problems, which can't be used at all with trait objects.
- acrichto: Today, a by-value self method builds a shim; maybe we could do something like that?
- nmatsakis: I've thought about something like that, but it winds up being pretty weird; it makes the out-pointer that we use under the hood more visible, which seems problematic.

```
trait CloneInto for Sized? {
    unsafe fn clone_into(m: &mut Self);
}
```

- nmatsakis: You'd pass in an out-pointer, and you could implement this for all Clone types. But you could also implement it for unsized types. Basically, the type system can't guarantee appropriate sizing, hence unsafety
- nmatsakis: Some people have proposed auto-boxing, but with DST it's really unclear how to make this work.
- nmatsakis: Maybe there isn't a great solution, but we should be on the lookout.
- nmatsakis: This is not unique to clone -- probably comes up in combinator libraries. But probably in the latter case you can impl for Box<Foo>
- pcwalton: Maybe you could specialize a clone for Any.
- nmatsakis: Sortof a Java solution. You have to use a downcast.
- pcwalton: It'd be horribly unsafe, of course
- acrichto: But Any requires the 'static bound
- nmatsakis: Yeah, but that's probably OK
