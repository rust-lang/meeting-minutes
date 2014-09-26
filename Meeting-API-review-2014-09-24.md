# 2014-09-24

# Attending

brson, nmatsakis, huon, acrichto, aturon, pcwalton

# Failure APIs to discuss

- channels http://doc.rust-lang.org/std/comm/
- RefCell http://doc.rust-lang.org/std/cell/struct.RefCell.html
- Vec
- String
- ToCStr (fails on interior nulls)
- Path (Path::new and all related methods fail on null bytes)
- fmt (non-utf8 contents)
- Mutex/RWLock (poisoning)
- println (what an I/O error happens?)

General issues to consider:
- how clear can we make the guidelines?
    (e.g. Niko would like to say "Prefer Result unless a good reason to do otherwise")
- when to allow both panic and Result variants?
- what changes if we have ! sugar?
- Option<T> vs Result<T, ()>


- nmatsakis: On the one hand, ! makes it easy to return Result, but can make it harder to know what deserves scrutiny

# Channels

http://doc.rust-lang.org/std/comm/

- aturon: Before we said that channels/mutexes, synchronization primitives that coordinate tasks, is a special case with regard to panicking. We said we want panic propagation by default. The idea was that tasks would have panics propagate throughout automatically. That led to the default mode being panicking. On the other hand you need a way to stop the propagation at some point which ends up with a different api.
- aturon: You arguably only ever need the result-producing one, but the pushback is that it's not ergonomic. Channels in specific are seen as so core.
- brson: bang syntax would make this so easy. Just send()!
- niko: I'm not a big fan of the panic propagation by default.
- acrichto: We're so heavily rearchitecting the runtime that it might make sense to move away from that. The most important thing is that someone knows the failure has occurred.
- nmatsakis: Propagation makes it sound like we want to encourage panics
- acrichto: It seems like a no-brainer where, if the default is the failing API, you *have* to have the Result API
- aturon: If you there's no other way to check
- brson: but recv().unwrap() is so bad
- niko: we renamed unwrap?
- aturon: haha not yet. Lots of RFCs and a bit entangled
- nmatsakis: does recv().ensure(), recv.expect(), or recv.assert() feel different?
- nmatsakis: what makes channels special to me is not so much whether they're fundamental, but that they're a showcase point
- brson: spawning and message passing were supposed to be sweet-looking
- aturon: Sounds like we're happy with Result only if we have !
- acrichto: What if we never get !, though?
- brson: Would probably still want to be consistent and only provide Result variant
- brson: recv().ensure() is better than unwrap() at least
- niko: ensure()/expect() still don't invoke too much...
- niko: I agree with brian, but I seem to be somewhat more tolerant of longer lines, particularly here where I don't expect to write `foo(port.recv())` so much as `let msg = port.recv().expect()`
- aturon: We've also pretty much decided to move to the @ syntax for macros/attributes? We'll at least have the option to use ! later.
- brson: maybe send().bang() ?
- niko: Maybe we should look at some other apis?
- aturon: We feel fairly comfortable in the sense you should not have failing and result-based variants?
- acrichto: I object a little
- aturon: this is one of only two apis that provide the variants listed (refcell/channel)
- nmatsakis: I think I feel differently about RefCell

# RefCell

http://doc.rust-lang.org/std/cell/struct.RefCell.html

- aturon: Ok so refcell provides borrow/borrow_mut which panic if it's invalid. Also provides try_borrow and try_borrow_mut which return options. The argument here was much weaker in the sense that the ref cell doesn't give you a lot of ability to inspect its state otherwise. There's no way to check to see if a borrow will succeed. Unsure how often you want to inspect the state, but we could instead just provide a method that tells you whether it's been borrowed.
- niko: I'm of two minds on this. I feel that inspecting the state is a bit of an orthogonal design choice. In this case the state is simple so it may not be so bad. In more complex cases you may have expensive predicates or atomic concerns. From an optimization point of view a result based variant may be more performant.
- niko: Here I want it to fail by default because it's so overwhelmingly in favor and I often nest it in some other expression and it would generate very long lines. map.borrow_mut().unwrap().insert() would be sad
- brson: Channels are much more dynamic, ref cells are much more static.
- niko: Certainly not cross task.
- aturon: Presumably if we had bang syntax that would change the calculus?
- niko: somewhat. I'm probably willing to add one more character, just not a whole nother function call
- aturon: I feel that this really does depend on bang. I would be happy with Result if we had it, but otherwise I would want to deprecate the try_ versions and expose a state inspecting version. If we did that and did what we said with channels then we wouldn't have the two variants in the stdlib.
- niko: I wouldn't object for ref cell being on the "known short list" of things that would fail. It's better if you don't use refcell but if you're using it already it's already a bit painful to work with
- niko: I would be ok taking your solution and if we end up with bang this method still implicitly fails.
- aturon: At this point I think we can revise the guidelines to very strongly discourage having both variants.
- niko: I'd consider having multiple variants in the SiegeLord example (different sets of errors). The range of errors that is covered is distinct.
...
- brson: planning on removing the try variants and adding a method to query what the result would be?
- aturon: That's plan B. Plan A is making the main versions return result and have the bang.
- brson: Presumably option is the wrong type here?
- aturon: That's a question, when to use option vs result?
- niko: Let's come back to that question.

# Vec/String

http://doc.rust-lang.org/std/vec/
http://doc.rust-lang.org/std/string/

- aturon: We've had a nuanced failure strategy here. To recap, we've got sugary indexing notation with Vec which fails. We have a get variant that returns an Option on out of bounds. There are slice functions which panic on out of bounds. There are things like head and tail which in some cases panic and in some cases return option. With string there's all of this plus the question of bad utf-8 boundary issues. To me this feels like a mess. The simplest thing is to say move everything to Result/Option
- acrichto: Could solve slicing by clamping to be in bounds
- niko: Seems a bit weird
- aturon: We should think about going this direction much more generally if we do. There's lots of places where you can do something somewhat reasonable when the arguments are invalid.
- niko: I agree with you about the current situation being very unpredictable. I could probably live with any predictable situation.
- aturon: What's the argument for failing?
- brson: It's mostly just the verbosity of not failing and having to check every single operation on vectors.
- niko: My rule of "don't write ensure() in sub-expressions" says we probably shouldn't have ensure() here.
- aturon: Seems better with bang?
- brson: much better with bang. In no other language I use I have to deal with lots of error handling with strings. Having to do the post operation error checking for these common data types seems bad.
- niko: even with bang it seems a little bad
- aturon: Ok, that's interesting. It seems that panickig on a contract violation is the closest similarity with not catching an exception.
- niko: I'm not a fan of off-by-one with manipulating indices or `if a < length`, but it is relatively unusual. I have some sympathy for if we apply our policy consitently I could see going for the case of fail here.
- aturon: Ok, so niko you've been pushing for strongly preferring result unless there's a good reason not too. Basically what we're saying here is that what we do now aligns with the guidelines in the RFC. You choose what's part of "your contract" and what's not. One factor is that your API should be consistent. We're not gonna push you too far in one direction or not.
- niko: What unites ref cell and this to me is that the state inspection is fairly straightforward. These are also building block things that aren't big operations (part of other operations). If I have to write .ensure() over a sub-vector it's painful.
- acrichto: I like APIs that match over options, e.g. looping over .pop()
- niko: hmm, that's true, that's the example I was thinking of, that actually influences my opinion, because I so much prefer matching over the result of pop than checking with `if`. I'm going to kind of reverse and say that if we have slice sugar we can try `Result` everywhere.

... discussion about vec returning result everywhere

- aturon: how about we try an experiment where it's all returning Result? I think we should push through the change to the name unwrap before we do this. That is a very painful change and if we have lots more unwraps I'd rather have the pain first.
- brson: agreed. It'll give us a lot better feel for the end results anyway
- niko: Result is technically a more "type safe" way, there's nothing that connects the "if" to a "call" for a case like matching on pop()
- aturon: Ok so if we go through this experiment, we probably shouldn't talk about the other apis yet and see what happens. If it works for Vec it will work everywhere.
- nmatsakis: should we just try for refcell?
- acrichto: NO! data is SO overwhelming
- nmatsakis: there are some esoteric use cases for refcell checking
- acrichto: I like idea of a single API being all result or all fail, but I still there are lots of times we won't want to fail. I really don't want to unwrap path operations or println!

...

# Option vs Result

- aturon: right now pop() on a vector returns an option, but as alex pointed out the analagous methods on strings have multiple failures modes and would perhaps return a result. What's the guideline here?
- niko: here's my rule of thumb:

```
vec.pop();
```

- niko: If you have the bare expression, should you get a warning? If yes, Result, if no, Option. As in, Option == "normally not present" and result == "abnormally not present"
- aturon: If we're revamping these apis to do the checking for you you're no longer asserting that the check will succeed.
- niko: if you want to pop but don't care if it happens to be empty, you can write:

```
let _ = vec.pop();
```

- huon: another reason for using result is better failure messages. Result can have a custom type which prints a custom error message
- niko: it's plausible for strings to force you to check the bounds afterwards.
- aturon: That's an interesting guideline: "Should you get a warning if you ignore this".
- niko: I think we should err towards result.
- acrichto: Why not HashMap::find?
- niko: I feel differently about find.
- aturon: This distinction may not matter so much as with error propagation and multidispatch you could convert between the two. We still have to make this decision for each api.

... missed discussion about Result/Option and what to use where
