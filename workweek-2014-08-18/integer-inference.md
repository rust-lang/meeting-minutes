# 2014-08-18 Integer inference/default fallback

- nmatsakis: A couple of related topics combined into a single discussion.
- nmatsakis: This is a chance to review the ergonomics of the int fallback removal
- nmatsakis: and to discuss a way of dealing with fallback by using default type parameters
- wycats: The default type param approach would solve the problem for me.
- pcwalton: Nervous about scope creep. Probably backwards-compatible, though
- nmatsakis: It's a small change, though.
- nmatsakis: rough draft of RFC for proposal https://gist.github.com/nikomatsakis/7e071b5b54f1da601e73

- nmatsakis: the basic idea is that when you have a type param with a default:

```
fn range<T:Enumerable=uint>(from: T, to: T) { ... }
```

we can say that, when we're inferring the type to use for T, if that turns out to be unconstrained, we'll pick the default.
- nmatsakis: In the RFC, I spell this out for every kind of type parameter.
- nmatsakis: Concretely, the effect is that you can do:

```
for i in range(0, 10) { ... }
```

and you'd get uint based on the signature above.
- nmatsakis: So, it's not specific to integers, which is nice. There are other cases where this might be applicable.
- nmatsakis: The interaction between default type params and inference in today's Rust is not well-specced. Basically, if you don't override the default, it's forced. With this proposal, more examples will typecheck (but existing examples should work)
- wycats: I want this for traits as well
- pcwalton: fmt is the killer example -- you want the following to work:

```
@format("{}", 1)
```

```
fn foo<S: Str = String>(string: S)
```

- nmatsakis: that's kind of a weird case; you'd use an int fallback but it's kind of random.
- pcwalton: I'm saying it'd be nice if our solution allowed you to do that.
- wycats: I know that if you have a trait-based overload, it's common to have an unconstrained caller
- nmatsakis: So, it seems like there are cases where you'd like fallback but the callee isn't in a good position to specify the desired fallback.
- wycats: Collect is an interesting example.
- nmatsakis: If we said that .collect falls back to Vec, but then you had the above function, under the proposal I put forward the two fallbacks would conflict and thus give you an error.
- aturon: w/ today's default typarams, you can use them to add typrams
- niko: only if they are new
- aturon: w/ this fallback we can't add default traits without breakage
- aturon: more significant case for defaults is adding brand new params, and that's fine
- pcwalton: with int inf. it may be easiest to add the fallback
- niko: in future we may want overloaded literals of other kinds that i don't want to think about now

...

- acrichto: off-by default lint that tells you when your integer literal is unconstrained?
- aturon: losing the fallback has bad ergonomic consequences. all proposals do something to fix it
- aturon: niko is worried about other overloaded literals

...

- nrc: do we have examples besides 'range' where the fallback is painful.
- pcwalton: format!, `let mut i = 0; loop { ... }; println!("{}", i);`
- nrc: what's to stop us from adding it now?
- nmatsakis: mostly that lack of integral fallback is painful right now
- nrc: it seems better to make typed parameter fallback right
- pcwalton: accumulators are very common
- nrc: I feel like you ought to be specifying
- wycats: why?
- nmatsakis: this seems like a place where a lint is appropriate.
- pcwalton: forcing people to specify int size doesn't actually mean they have to think about overflow. it's good for security, but we have to make sure we're not the TSA here.
- steveklabnik: It's also a problem in the guide. I've just been annotating all literals with int.
- wycats: It also forces you to care about the int/uint distinction, and maybe you don't.
- brson: Should we fall back to uint?
- wycats: What about negative literals?
- nmatsakis: We can lint for that.
- dherman: The first time I wrote a loop in Mozilla code, it was descending and totally broken (I used a uint)
- pcwalton: To me, that's an argument against C-style loops. But I worry about people getting burned with accumulators.
- nmatsakis: We have to deal with the -1 problem though.
- pcwalton: I think int is the right way to go.
- nrc: The very fact that we're having this discussion makes me NOT want to have a default
- wycats: Most of the time it doesn't matter.
- pcwalton: The biggest ergonomic problem is asserting that a value is equal to some literal.
- nmatsakis: So a lint you can turn off in tests seems reasonable.
- pcwalton: I've never seen a bug arise from this, and we should reserve our caution for such cases.
- brson: One of the arguments for signed fallback was "-1". But if the idea here is there's no constraints saying what type you need, what does it matter?
- wycats: You don't want the printout to give you MAXINT
- nmatsakis: it seems like the default should be the closest to "normal" numbers
- brson: Have we considered requiring suffixes everywhere?
- pcwalton: I consider that a nonstarter.
- steveklabnik: One side effect: there's a difference between sample code and real code, and the sample code will be disproportionately annotated.
- pcwalton: Not having a fallback is great for production-quality code, but not for samples/tests
- acrichto: And once you care, turn on the lint.
- wycats: We don't do anything for overflow.
- pcwalton: We decided against overflow checking because of runtime cost. This is purely ergonomics, so not entirely comparable.
- pcwalton: When I think about overflow problems in C++, a lot of it has to do with pointer arithmetic, so we're already good there thanks to bounds checking. The second big class is integer overflows where you knew the sizes, but you didn't consider problems with user inputs. And then literal inference doesn't enter into it. I can't think of any cases in the wild where literal inference would actually cause a bug. I have seen bad interactions with coercions, but we don't have that.
- nrc: So does anyone NOT want to add back the fallback?
- steveklabnik: I think strcat objects.
- acrichto: I think the issue is that you can remove code, and then the types of your literals change.
- nmatsakis: I can't think of an example where that actually breaks.
- wycats: The only bugs have to do with being in between MAX_INT and MAX_UINT
- erickt: Did anyone bring up the 16bit processor issue?
- niko: the lint can help a user in the 16bit circumstance by forcing the user to be explicit about the integer types.
- Resolution:
  - Add back integral fallback to int
  - And a lint for cases where it is unconstrained
  - aturon to write RFC
  
- nmatsakis: For default type params, I worry about the poorly-specced interaction with inference currently, and that you should be able to use _ to fill in only some default type params.
- wycats: I want the right interaction with inference.
- nrc: Do we allow named type parameters currently?
- nmatsakis: Not currently. I considered a key=value notation. I was assuming we did not want to take the Pythonic approach that mixes positional and named arguments.
- nrc: Since the associated items proposal already has key/value for generics, maybe two birds with one stone?
- nmatsakis: But then names become significant everywhere. Maybe that's ok.

(brief tangent below:)

- dherman: If we decide that signed is the default, what about out-of-range literals?
- acrichto: There's a lint that warns about that.
- nmatsakis: When we tried to be strict about this, we ran into cases where people were intentionally using overflow. Requiring an explicit suffix in these cases is probably ok.
- brson: We've had lots of bugs with constant evaluation. Isn't that a problem?
- nmatsakis: I'm sure there are bugs, but I'm not sure how it relates to the current discussion.
- dherman: So something that's out of the range for the type we're inferring should be an error, which is fixed with an explicit annotation?
- nmatsakis: One problem is that the range is target-dependent.
- wycats: Then you definitely want a compiler error.
- pcwalton: But then you might build OK on a large machine and fail to compile after uploading to a small machine.
- dherman: So we're silently inferring a target-specific type, which is inherently nonportable, so you should probably get an error. But I guess it's not guaranteeing you much anyway.
- acrichto: There's a separate question about how large uint/int should be. I want literal inference regardless.
- dherman: I'm just arguing about out-of-range literals.
- nmatsakis: We have a lint for this already, but I'm not sure how it interacts with explicit suffixes.
- nmatsakis: In the end, there are too many weird cases to deal with here; we can't make bullet-proof guarantees. So a lint seems like the right, best-effort approach to catch the most common mistakes.

(back to default type params)

- nmatsakis: Not sure how to resolve interaction with fallback. The conservative route would be to give an error on multiple fallbacks.
- nrc: We can implement that and see if it's actually a problem.
- nmatsakis: I'll experiment and report back.

# int/uint sizing

- nmatsakis: pcwalton at some point suggested int/uint always be 32 bit, and have separate pointer-sized types. There are interactions with indexing.
- nmatsakis: Also, an RFC said that on a 16-bit arch you might want data and pointer sizes to differ. It's an additional motivation for the above.
- acrichto: https://github.com/rust-lang/rfcs/pull/161

from summary:
"Either rename the types int and uint to index and uindex to avoid misconceptions and misuses, or specify that they're always at least 32-bits wide to avoid the worst portability problems."

- erickt: I do like index and uindex much better than intptr.
- erickt: On most systems size_t is the same as uint_ptr_t everywhere except for segmented systems.
- acrichto: from the RFC: "The worst failure mode is in libraries written with desktop CPUs in mind and then used in small embedded devices"
- acrichto: I'm not worried about the literal inference as it relates to 16bit CPUs
- nrc: If you're working on a machine like that, you should be using the sized integers. So then the problem is libraries that are using unsized ints but should be using sized.
- pcwalton: The reason that Go changed from 32 to 64 bit is that len is a built-in and they couldn't change the signature without breaking back-compat, so they changed the size of int instead. That's not a problem for us.
- nmatsakis: I just feel like when working with C it's difficult to choose the right types. Our current system just feels so simple.
- acrichto: That's a real danger -- the C types change on  various systems, it's hard to predict
- pcwalton: I'm worried about cases where size differences have large performance impact
- pcwalton: Another concern is memory usage. I wish that the Servo team was here.
- nrc: That's why Firefox is 32 bit on windows. It's faster and quite a bit smaller.
- pcwalton: programmers sprinkle "int" everywhere, and so memory footprint is much smaller on 32 bit
- nrc: If you want to save memory, then you're thinking about the size of your ints and you should be typing i32. But I am more worried about the 16 bit case -- if people are writing libraries that are using "int", that can screw up the 16 bit case.
- acrichto: But probably in embedded settings you're writing all the code yourself, not using general purpose libraries.
- brson: We can punt to the library authors.
- nmatsakis: What would it mean if we did make this change? What would be the impact if we went to index and uindex?
- nmatsakis: Presumably, the type that's expected by the indexing operator would change. Would we want it to be 32 bit? It'd change the way the trait is set up so that you could use either index. Or maybe we're just strict about it.
- acrichto: Right now it requires uint.
- dherman: It feels like the ideal here is that algorithms that use int flexibly are "morally" generic over a trait.
- aturon: You'd need functors to make that work well.
- nmatsakis: Without int/uint it's very difficult to write portable code.

... discussion of 8 bit case

- nrc: the RFC is about libraries using int when they should be using i32
- nmatsakis: So that argues that int is too small, while pcwalton argues it's too big. So i32 is the right middle ground.
- pcwalton: If you're using a 16 bit CPU, you don't care about speed, but you might care about correctness. What I'm not clear on is the exact changes we'd need to do this. I think we're just talking about naming changes, not any deep semantic thing.
- pcwalton: For naming, I prefer "size" to "pointer"
- dherman: One downside of "size" is that usually you're thinking of this as an index
- pcwalton: What's the result of len?
- pcwalton: I think we need to try to do this and collect data on what happens.
- dherman: The thing that worries me is that the programming model we're talking about has a few radical differences. Is the meaning of most programs going to shrink the program to whatever the size of the architecture is? Or should it have a fixed meaning that works the same on different archs. Those are radically different.
- acrichto: This also introduces a lot of extra distinctions, but 16 bit machines are such a rare case.
- pcwalton: Right now a lot of code uses int that could use i32 (we think); you really only want the index sizes to grow/shrink.
- wycats: The thing that worries me is the subtle difference between an int and one that's sized correctly for indexes.
- pcwalton: One of the dangers here is that a lot of functions that currently return int where you can directly use them as indices, but with this change you couldn't use those directly (would need explicit coercions)
- dherman: I think that's a salient issue. But you don't have to think of "sized" as "scales to arch", but rather, "this is what you use to index".
- wycats: We should write something up that explains the coercion hazard.
- pcwalton: People use enumerate to get ordinal values and do math with them, but may never use the result for indexing.
- nrc: One other worry: we might paint ourselves into a corner with say 32bit ints. How long do we expect Rust to be around? At some point, 64 bit will be a better default.
- pcwalton: But C is locked into 32 bit and the CPU manufacturers are never going to make C slow.
- pcwalton: SIMD will always be better with smaller values, though.
- dherman: Multiple languages have gone through this question, and they keep getting different answers. It seems like none of us are really sure about the right principles.
- nmatsakis: I feel like we half want a unit type system
- dherman: No way to make that practical in time.
- derman: The worse-is-better, ergonomic solution is to just have one default integer type that scales to the platform you're on. That's the path of least resistance -- least risk in the sense of complexity/usability issues for the language.
- pcwalton: I agree. It's the most conservative option. The biggest risk is memory footprints and microbenchmarks.
- nmatsakis: I think dherman is right. It just seems like Rust is so much easier than C on this front, and I'm wondering what we're giving up -- probably scaling to other architectures.

... missed ...

- dherman: What we want is for "int" to be right default. We need a philosophy for where you use int, and where you use specific sizes (like serialization)
- pcwalton: The upside is ergonomics going btween indexing and match. But downside of increased memory usage and SIMD.
- wycats: Those downsides apply only if people properly use those other types.
- nrc: My objection isn't about performance, but safety: you have to think about overflow when working with ints. 
- nmatsakis: I'm not sure how much we want to worry about 16 bit embedded systems
- dherman: I'm definitely not ready to write off embedded systems programming in Rust
- nmatsakis: Sure, but just 16 bit cases, which are disappearing.
- acrichto: I'm not sure Rust would even fit on such a machine.
- dherman: In the long run, our implementation might improve so that we can fit. But 16 bit machines may stick around -- sensors, etc. Who knows?
- dherman: I could imagine adding features later e.g. scoped sizes for ints. I think Cargo subecosystems are going to be important anyway. I could see that being used for embedded/16 bit systems
- dherman: If we don't find an airtight solution, we should do the most conservative thing. Something like where we are today seems like the safest route -- there are downsides, but that we know how to cope with, and can mitigate down the road. On the other hand, I worry about introducing a bunch of types.
- nmatsakis: The hazard in Servo, keep in mind, is having *too much* range, which seems like the right size to err on.
- nrc: Seems like simplicity wins the day everywhere except 16 bit, and there you'll have special libraries anyway.
