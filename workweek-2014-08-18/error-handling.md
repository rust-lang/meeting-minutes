# 2014/08/19 Error handling

- aturon: 2 RFCS in queue

RFC 1: https://github.com/rust-lang/rfcs/pull/201
RFC 2: https://github.com/rust-lang/rfcs/pull/204
Guidelines: http://aturon.github.io/errors/signaling.html

- aturon: more important has combination conventions and sugar.
- aturon: have guidelines for fail vs. result. difficult to capture. currently reflecting current practices, but not clear enough to resolve disputes.
- aturon: current says 'if there's a contract violation, designers can choose whether to fail or expose the Result'.
- wycats: in ffi cases, failures can make auditing difficult
- aturon: now failure is not marked. can't tell from looking at API.
- steveklabnik: docs indicate what 'failure' constitutes: does it mean 'task failure' or any error?
- aturon: table discussion on terminology
- aturon: proposal for extreme position on this question. gradually moving toward 'Result', away from 'fail'. let's discuss going to the extreme and making the guidelines clear cut. There are a small amount of instances that can fail:
asserts, `[]` notation (array indexing, hashmap lookup); everywhere else contract violations are signalled by returning 'Result'.
- aturon: collections would have sugar for indexing, but normally named methods don't
- wycats: right now very inconsistent on what has failure vs result semantics
- aturon: no room for interpretation with proposed rules
- niko: 1 important thing. non important thing. we could debate on RefCell/borrow/borrow_mut, but the important thing when there are cases where the API it comes down to testing validity of inputs vs internal state.
- aturon: assert is when I always expect it to succeed.
- aturon: it feels too noisy to have assert/unwrap everywhere.
- aturon: proposes 2 suffix operators: `?` and `!` sugar. Working with `Result` will be a lot more common. `unwrap` has a different flavor, it feels wrong and you should be doing something else. Contracts are more visible.
- wycats: prefer to check invariant by me, instead of by the API.
- brson: auditors have much less idea what would be guaranteed.
- aturon: those bangs are already happen.
- brson: harder to find `!` then to find `assert`.
- niko: easier to overlook `!`. `unwrap` is a poor name.
- brson: can a syntax highlighter highlight the bang?
- niko: probably not easily.
- pcwalton: what did swift use?
- niko: bang.
- niko: can we discuss the principle? does everybody agree?
- niko: worried that we don't understand the consequences. is there any way we can feel better about this?
- niko: I've been writing code that follows these principles and feel good about it.
- wycats: you could decauple !-sugar from ?-sugar. libs using try! become awckward
- niko: what?
- wycats: think this proposal means apps don't ever use !
- aturon: no, they can use it but they must expect
- wycats: in the normal case, you use `Result`. Every frame between you and the application code needs to propagate these results.
- acrichto: what fails right now?
- aturon: `RefCell`, `to_c_str`, `Path::new`, and channels, vector indexing
- pcwalton: some libraries you can't do this. For example, OpenGL for performance issues
- niko: good point. but it is a guideline, not an iron law.
- pcwalton: you aren't checking the contract when you call it, you check later and fail
- alex: we should change most users of fail to return `Result`. Putting `.assert()` everywhere does get wordy. Tough to adopt 

...

- erickt: where does monadic 'do' fit in?
- niko: poor  fit for rust because it doesn't cope with control flow. works in some cases.

(talking about monads...)

- aturon: even though return type of '!' and '?' are same, (??), typechecker will help you pick the right thing.
- aturon: other langs like swift make '?' more like 'do', whereas 'try!' returns
- wycats: `try!` is less-ergonomic exception, but it's ergonomic to 'intercede' and inject custom handling.

-brson: right now if you assert, it's easy to insert a breakpoint to debug. If you propagate the error, it pushes the error up the stack and it can be hard to debug.
- alex: we attach some state to errors, which can help with this, but we don't do it everywhere.
- aturon: `?` could be implemented as a trait so we can chain `None` results.

...

- wycats: would call some trait to convert to Result, then decide whether to propagate
- niko: makes it ergonomic to get into a Result. Custom errors don't work.
- wycats: with the gdb thing you could imagine having a mode that ..
- niko: problem is that when you return an Error for the first time there's no opportunity to capture that the error happened.
- acrichto: ?
- acrichto: every time you convert from error A -> B you tack on more info. a RefCell wouldn't return <T, ()>, but <T, RefCellError>.
- niko: Every type doesn't need its own error.
- wycats: Library boundaries are good places for new custom errors.
...
- brson: what's better than stack traces?
- wycats: every time you 'map error' you get a chained record of what happened.

- alex: this can include local variables and other information.
- wycats: with a normal exception, a lot of stack frames are unnecessary. This allows you to control the information that you'd store in an exception.
- niko: clear performance cost
- wycats: only on errors
- niko: it's nice to do error handling after a call. perhaps for these simple calls they could use Option, so they don't have to allocate as part of the 'error chain'.

- aturon: let's talk about the other RFC
- aturon: working with Result 'at scale': different libs with different notions of errors
- aturon: two aspects: 1) add trait that defines what an error is: desc, detail, cause.
                2) revision of `try!` to make propagation work ergonomically.
- aturon: if you are using e.g. I/O + HTTP, each has their own 'Error'. You also have an 'Error' type. Can't use `try!`, have to write shims to convert between error types.
- aturon: adds a trait to convert between diff types of errors. e.g. 'here's how to lift an IOError into my own Error'. Then you write `try!` and they'll do the chaining.
- wycats: cargo is doing this.

...

- wycats: when interoping with serializer code you get back an 'E'. Need some initial trait to convert to a box errors.
- niko: so Serializer would not work with generic E, but a boxed Error?
- wycats: no it's generic over <E: Error>
- niko: so when you are writing code that the serializer will call you have to ...
- wycats: anywhere code wants to be generic over error, having a generic error trait helps with interop.
- aturon: trick relies on multidispatch proposal.
- brson: is multidispatch attached to some other RFC?
- aturon: assoc items
- niko: some precedence (not the automatic nature), but this is what Java does more or less: chains of exceptions, declare your own type, write caught errors in your type.
- niko: more annoying to write out try/catch
- aturon: nothing about the trait forces you into any representation, or even having causes.
- wycats: in high level libraries you would want more rich errors
- niko: perf costs are less relevant.
- aturon: I/O can keep doing what it's doing
- wycats: I/O I added some things like what file you tried to open. Requires allocation, but worth it.
- erickt: is there anything in this proposal to help searching the error chain for specific errors?
- aturon: tricky issue. if i'm using lib 'A' and it uses 'B', I don't want to care about error types in B because that's encapsulated. what you want is 'A' to lift 'B's' errosr into its own type so clients of A only care about A.

- brson: people would want to inspect that data if lib authors fail to lift correctly. does this RFC require Error to implement Any?
...
- wycats: seems like Send should be a supertrait to make Error implement Any.
- niko: Can't return input pointer arguments in the Error...
- wycats: I think if we don't say it inherits from Send...
- niko: yeah, you want it to be sendable so you can let the error escape the task

- erickt: user might want to handle.
- acrichto: this is where low-leel libs they will return custom error types. everything implements this error trait. so if you to. ... everybody's going to be using Box<Error>. Frameworks don't care at all what error happened.
- acrichto: in general expect enums to be common low down in the stack, but higher in the stack.
- wycats: higher-level code makes it harder to recover
- erickt: worried people will be lazy and just box, not provide the semantic info
- wycats: can't stop people from being lazy
- niko: can't solve
- aturon: if we don't provide such a trait, others will. it's inevitable so we should standardize
- wycats: if not in std can't fix the std api's to use it
- erickt: like the idea of this Error type. if you don't use Box<Error> but T: Error, and have a concrete type...
- wycats: only type you have to use Box<Error>

(...)

- nmatsakis: `?` operator can attach filename/line-number
- pcwalton: there is a real cost to storing line numbers in the binary. It is on the order of the numer of lines in the code. It can add 10%+ to the binary size.
- wycats: I find this very surprising, but I totally believe it
- nmatsakis: could approximate in non-debug mode or omit

(...)

- niko: The `Error` trait seems pretty uncontroversial.
- wycats: with all the extra sugar, you shouldn't `.unwrap()`.
- niko: Want some time to think about this. What would help would be a full list of functions using fail. Find use cases that don't fit into this scheme. `Path::new` was brought up as a example that would be difficult.
- wycats: With the current design.
- acrichto: Maybe path could not care about errors only up to the point it converts a path to a c_str.
- wycats: It can also be hard for auditors to audit all users of `Path`.
- aturon: We also fail in atomics. Loads and Stores don't make sense. Release-Read will fail. You could imagine breaking up the enum, but that doesn't seem worthwhile. We could make atomic a special case. Or make all atomics return `Result`.
- wycats: I've had to deal with things that fail. When dealing with embedded coding, it's really painful.
- niko: if we had something like enum refinements (virtual structs), have enums with a subset of the orderings, would that help?
- wycats: If the only way to write this code is to misread the documentation.
- aturon: API designers can choose to give a contract. There may be things on our radar that would make it hard to convert over to Result.
- wycats: It seems like there are bad cases where you have essentially static errors, but you have to check for it dynamically. If you have an api that's designed to work with static values, it feels dubious to force it to check for dynamic errors.
- niko: do the llvm intrinsics work if the thing is not known?
- ???: there is a different intrinsic for every ordering.
- alex: task spawning is another place we can fail
- wycats: that is an out of memory error. It's okay to abort in this case.
- brson: there is a chance there is a case where we'll have failures.
- alex: divide by zero can also fail.
- wycats: for skylight we'd want to do try-catch. But if they're trying to use a C library in Rust. There could be cases where we have aborts, but that's also true in C.
- niko: we have a fallback path that you can choose to make or not.
- aturon: we've discussed moving away from unwinding. These conventions play into removing this.
- wycats: I'm arguing both cases. syklight wants to try to recover from errors, but also get dynamic exceptions if possible.
- pcwalton: should be able to write apps in rust that are reliable and can blow up on fail. This covers the kernel and embedded circumstances.
- niko: `fail!()` and a kernel panic are equivalent.
- pcwalton: optimization flag that disables unwinding might make sense.
- niko: I feel pretty good about this proposal. Macro syntax is a big open question.
- pcwalton: Don't like `#[foo]` is different.
- dave: `@` is common in java. Originally macros had a `#` prefix, but when it changed it became inconsistent. It's okay they aren't identical. Using `@` for metaprogramming things and `!` for error handling, we can point to strong traditions and not going down new road with syntax.
- brson: I'm concerned about burying this macro change. It should be it's own RFC.

(...)

niko: Summary. Generally like the proposal. Why do we use fail that don't fit this criteria. Update the RFC title

(...)

aturon: the word `fail` is problematic. It can become very awkward to talk about something failing. Proposing `abort`?
pcwalton: In C, `abort` sends a SIGABORT to the process.
aturon: It's a better ambiguity to use `abort` than `fail`.
nrc: Returning an error, i never use the word `fail`.
- wycats: it's confusing when talking about rust.
- pcwalton: Perhaps `panic`?
- dherman: Go's messaging is that you're only to use `panic` in the case where you can't recover.
- brson: You can't be guaranteed you have unwinding, so `panic` could be taking down the whole process.
- brson: we're going to have to eventually talk about process abort.

Action Items:
- Search the code base and find all the uses of `fail!()` and `.unwrap()` that don't fit in this use cases.
- Update the RFC title to mention the `@` macro change and mention changes to the annotation syntax.
- Write up an RFC on renaming `fail` to `panic`.

