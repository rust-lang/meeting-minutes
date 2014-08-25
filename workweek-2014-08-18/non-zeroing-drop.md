# 2014-08-20 Non-zeroing drop

https://github.com/pnkfelix/rfcs/blob/fsk-nzdrop-rfc/active/0000-remove-drop-flag-and-zeroing.md

- pnkfelix: The way Rust currently works, if you have a struct that implements Drop, there's a secret boolean flag that's added to it. Thus, the size of structs gets increased by 1 bytes, which with alignment can cause the size to double.
- pnkfelix: The reason we have this flag is the semantics of dropping. You can move a struct to another place (via assignment or a call), and the new owner is in charge of dropping it. But the old location is still around. We use the implicit flag to track whether a drop is still needed (at the end of the scope).
- pnkfelix: The RFC proposes to switch from this dynamic drop semantics to a static drop semantics. Idea is that, instead of saying at the end of each scope we dynamically walk over all variables declared in that scope, we force control-flow merge points to have dropped exactly the same variables.

... example in Vidyo ...

- pnkfelix: Initially, I thought that the compiler would just force you to manually move/drop to ensure that control-flow join points always have the same live values. But it's just not workable in practice -- there are too many cases where it's fine to have the compiler do the drop for you.
- nrc: We also have zeroing. Are we replacing both?
- pnkfelix: Yes.
- nmatsakis: In some cases it's a pointer that gets nulled, not a drop flag, but it's the same idea.

... more on screen with Vidyo ...

- pnkfelix: Right now, I haven't put in the code to generate the drops. I've just implemented a lint that catches when explicit drops are only occurring on one control-flow branch.
- pnkfelix: The reason I've focused on the lint for now is that there are some RAII-like patterns where you really want to know when this is happening. If there are side-effects like releasing a lock, you really don't want that to run earlier than you expected.
- pnkfelix: So part of the RFC is to distinguish between types with side-effectful drops and those where drops are benign. Then we can warn when there would be an implicit side-effectful drop.
- brson: Do you have examples, other than mutxes, that fall into this category?
- pnkfelix: Closing a file or flushing a buffer would be examples. There's certainly room for disagreement about particular cases. I want the library designer to be able to make this decision.
- pnkfelix: Freeing memory is a "pure" drop. I can understand that you might care about freeing memory -- it is observable.
- pnkfelix: There's another piece of the RFC: another lint that can warn even on implicit "pure" drops (but this lint is allow by default)
- brson: Do you have concrete examples of where mutexes go wrong? 
- nmatsakis: For lower-level mutexes, this can be a problem. But the higher-level mutex probably doesn't need to be marked "side-effectful"
- pnkfelix: That's part of the idea: when you build higher-level wrappers/abstractions, you can effectively hide the effectful drop.
- pnkfelix: I started with the conservative viewpoint of side-effectful by default, where purity is opt-in, but it seems like that's not the right default.
- pnkfelix: A large portion of the RFC was to reduce the number of false positives by the lint, which sometimes requires contortions. But if you make drops pure by default, we can probably avoid a lot of these contortions.
- dherman: It feels like RAII usually abstracts away a lot of these details. If you have no interesting join points, you have a silent drop anyway. This proposal seems consistent with that view -- you shouldn't have to think about the cleanup explicitly.
- nmatsakis: The lock is a good example of this, where the existence of the object gates access to the data, so it's irrelevant when it's dropped. That's where RAII is working at its best.
- pnkfelix: So there are language changes -- running drops before the end of a scope -- but the other piece of the puzzle is the implementation strategy. I've implemented a lint, but done no changes in trans to insert the early drops.
- pnkfelix: We could land this lint soon (perhaps allow by default), which would tell people when their code is not future-proof -- we could catch cases where people are relying on drop flag semantics.
- dherman: Seems like we'll get performance benefits, right?
- pnkfelix: Yes, I certainly hope so.
- brson: Do we need to prove that before landing this?
- pcwalton: I don't think so. I think the drop flag is something that makes us less clearly a systems language.
- dherman: I feel like it's justifiable, but it would be nice to get some numbers.
- nmatsakis: We'll do measurements. It should certainly improve compilation times.
- pcwalton: My guess is, runtime will not change much if at all. Code size will improve, 5-10%
- dherman: Why no runtime gains?
- pcwalton: Almost every time we've reduced the number of copies in L1 cache, it's made no difference.
- dherman: Because caches are so good?
- pcwalton: That, and register renaming on x86. Reg->reg moves are basically free.
- zwarich: For things that are in registers, zeroing can improve performance, because the register renamer can then re-use the register.
- zwarich: For things on the stack, you do save storing 0 on the stack.
- zwarich: The space usage improvement of not having drop flags, in the long term, should be big
- pcwalton: And space predictability. struct layout, etc
- dherman: I agree, and I don't think we need performance to justify this. We shouldn't have funky things going on behind your back. It's a fuzzy line, but a lot of it comes down to what's traditional in a systems language.
- pcwalton: Of course, enums are another example of magic, but I think it's fine.
- erickt: I thought LLVM doesn't optimize things in certain cases due to the drop flag?
- pcwalton: I'm not sure; it certainly puts more load on the memcopy optimizer
- pcwalton: Because C++ move semantics are basically a library convention, the language doesn't prevent you from running destructor on things that have been moved, so you end up running it twice. SO the move operation has to take care of bookkeeping to make sure that running the destructor twice is benign. Which generally involves zeroing out.
- pcwalton: So we'll be improving over C++, though it's not a big deal in C++ since people don't use move semantics very much. But it is a feather in our cap.
- nmatsakis: I think this fits in really well with our story that Rust code translates very directly into assembly.
- pcwalton: How much code does this break?
- pnkfelix: Depends on what you mean by breaks
- pcwalton: The match rule that you added
- pnkfelix: Currently in Rust, you can have a match that takes in a struct, and one arm goes by ref, the other by value.
- pnkfelix: It's very difficult for that to make sense in the new paradigm. So the RFC says that if any arm moves, all arms must consume their input by the end of the match.
- pnkfelix: Again, the compiler can do it automatically.
- pnkfelix: I'll have to double-check, but I don't recall seeing many problems with this rule.
- nmatsakis: I think it's a good change.
- pnkfelix: With the current borrow checker, you basically can't bind by ref in one arm and move in the other.
- aturon: Would non-lexical borrows fix that problem?
- pnkfelix: I think so
- nmatsakis: It's a very rare case anyway. I'd rather start with conservative semantics, and then we can loosen over time if needed.
- nmatsakis: In Felix's proposal, there was a QuietEarlyDrop trait. I think this is a good example of the opt-in-built-in-take-2. It fits the pattern exactly.
- nmatsakis: There was some discussion of the defaults here
- nmatsakis: I think brson's example convinces me even more that quiet-by-default is right
- acrichto: We've had a lot of success with unused-must-use etc. We should follow that pattern.
- pnkfelix: The reason to use a trait instead of an attribute is to express the "bubbling up" -- e.g., an Option<T> where T is effectful should be effectful. The attribute mechanics can't do this
- nmatsakis: And sometimes you want the bubbling to be conditional.
- pnkfelix: With opt-in-built-int-2, we would just use it; but for now I"ll just implement it in the same style and we can switch later.

RESOLUTIONS:

- conservative match semantics
- lose reasoning about reordered drops
- use trait for quiet/loud, default to quiet

