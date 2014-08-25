# libgreen / TLS redux

- ...some discussion omitted...

- aturon: we should rename task to thread if we do this
- acrichto: agree

# Introduce scoped TLS API

```
set_value(key, &new_value, || { ... })
with_value(key, |value| { ... }) // but see below...
```

it would be nice to use RAII rather than `with_value`, but it must be tied to local stack frame. We can use a macro but it will have to invoke unsafe methods. It would be better if we had privacy hygiene but unsafe is ok for now.

Macro would expand to something like `&get_value()` such that result cannot ever be moved.

# Conclusions

- TLS will always be native TLS (thread local storage)
- Library to have scoped TLS on all platforms

Current definition of std::rt::task::Task

```
pub struct Task {
    pub heap: LocalHeap,
    pub gc: GarbageCollector,
    pub storage: LocalStorage,
    pub unwinder: Unwinder,
    pub death: Death,
    pub name: Option<SendStr>,

    state: TaskState,
    imp: Option<Box<Runtime + Send>>,
}
```

Replacement for all these fields:

- heap - TLS variable for Gc<T> pointers, initialized on task spawning
- gc - gone
- storage - gone
- unwinder - private TLS variable to some try function
- death - some private TLS variable or some similar thing
- name - set while spawning or set manually 
- state - go away, not necessary if there is no Task
- imp - gone, it's all native

aturon: Can introduce TLS channel for communicating failure/success, have the fail macro send data along this channel -- this is one way of replacing the current `Death` field

- Major conclusion: we no longer need a Task, there truly is no runtime
- acrichto: the try-catch block is now safe and super lightweight. Very easy to deal with embedded case
- pnkfelix: Re-entrant?
- acrichto: Not at first, but probably in the future.
- erickt: Can a drop impl tell if you're failing?
- acrichto: yes, and you'd still be able to do that via the private TLS variable
- nmatsakis: We could add a separate drop method for failure unwinding -- or even two drop traits
- nmatsakis: In the compiler, there's often stuff you want to do only in the "bad" case, where you want to clean up an intermediate result
- luqman: Need to make clear the TLS story for platforms without native TLS. Emulation layer?
- acrichto: Yes, we will require some form of TLS 

- acrichto: There are some organizational questions here. I think liblibc should be libsys with bindings to all OS definitions. Totally platform specific. Basically take all the scattered platform APIs and put them there
- acrichto: Anyone can use/contribute here

- brson: What would go into this crate?
- acrichto: liblibc already has a lot of bindings, otherwise we just grow over time
- brson: This library goes in libstd. You really want to open the floodgates?
- nmatsakis: who can blame us if system libraries change and/or variations on unix don't agree
- acrichto: I've been adding tons of bindings in libnative, and no one else is benefiting from them
- erickt: An alternative would be ... (essentially to keep this stuff private)
- acrichto: libstd depends on a large subset of this functionality; i'd like to centralize it. If you feel like adding your own stuff, others can benefit
- nmatsakis: Would you rather have everyone writing their own bindings?
- erickt: I'm just not sure rust devs should have to maintain this
- nmatsakis: It could live in Cargo -- libbindings
- acrichto: The problem is, rustc depends on it
- nmatsakis: I want to understand better the concern here. Will system APIs really change?
- brson: No.
- nmatsakis: There will be bugs in the bindings, of course.
- brson: We put a lot of effort in removing the C surface area from Rust.
- acrichto: The alternative is to hide them
- erickt: Or we could follow Go's path and use raw syscalls. We'd have to be careful...
- acrichto: FFI in Rust is a lot nicer
- brson: Is this part of the spec?
- nmatsakis: If it's part of our std distro, then yes
- brson: We could put it in Cargo
- nmatsakis: The only real downside there is that we have duplicate the ones we use in the compiler
- nmatsakis: wycats has asked: If Rust is a systems language, why is it harder to get access to systems API than it is in Ruby
- steveklabnik: But the Windows story on Ruby is bad
- acrichto: We have to make sure that Windows stays at some level of parity, at least for libstd
- acrichto: The proposal, again, is: rename liblibc to libsys, move all of our extern bindings there, distribute the crate but don't re-export from libstd
- acrichto: Not really an issue for other Rust implementations -- they would just copy it, it's basically just a bunch of header files
- brson: Ok. As long as not exported in std
- brson: What about libc types? We've said before they can show up in libstd
- acrichto: We might want to expose an even lower level library that exposes some primitive C types
- brson: Certainly some Rust APIs need C types
- acrichto: One worry is, a lot of Windows and Unix APIs require linking to RT....
- nmatsakis: Note that "sys" fits into Jack's cargo conventions
...
- brson: We could eventually support linking to the full menu of C libs
... 




















