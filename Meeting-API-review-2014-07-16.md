# 2014-07-16

# Agenda

* What does unstable mean?
* core::cmp
* core::default
* std::task
* alloc::rc

# Attending

brson, cmr, spernsteiner, aturon, acrichto

## "unstable"

- brson: I'd like to agree about what unstable means. We've had a few different interpretations of it. I believe Aaron is more willing to break unstable functions than I am.
- aturon: That's right. I sent an email to Brian and Alex where I wrote some definitions...

--- aturon's email

# Stability levels

I think of the terms primarily as levels of promises:

Experimental
- Item may radically change or disappear altogether.

Unstable
- Item may be refactored, but its functionality will always be available in some form.

Stable
- Item locked in place. Only non-breaking changes permitted.
- Example non-breaking changes:
     * Making an argument type more general, e.g. going from &[T] to Iterable<T>.
     * With `impl Trait`, you could also free yourself to change the concrete return type.
     * Adding new, default type parameters.


# Stabilization process

Here's a proposal for a two-pass stabilization process. I think it's pretty close to what we've been doing, but clarifies a bit.

## Pass 1: initial API scrutiny:

Refactor/deprecate as needed, mark iffy things as experimental, and otherwise default to `unstable`.

Crucially, we should be giving a text note to *every* stability  attribute other than `stable`, as a breadcrumb for the second pass. This  should say what is preventing us from marking `stable`. If there's no  specific reason, we can say:

    #[unstable = "will become stable during next pass"]

In a few cases, we may feel confident enough about an API to mark it  `stable` during the first pass. I think we should push for this when we  can, because (1) time is short and (2) making commitments in the easy  cases will help lay the groundwork for committing to the harder ones.  The key danger here is that the API may have some relationship to other  APIs we haven't visited yet (what happened with the mem/ptr case.) So we  should do this only when we're confident about those relationships.

For APIs in need of substantial work, do that work prior to  stabilization meetings. I'm working on a roadmap/plan for this. (So far I  think we've all been picking the low-hanging fruit for stabilization  meetings.)

## Pass 2: pulling the trigger:

At some point, take a second pass over modules we've already visited. This pass should be much faster: it's about moving unstable to stable.

Prior to pass 2 on a module, the meeting leader should check over all of the blockers listed as text on `unstable` attributes.

We should start performing pass 2 on the "most important" modules even  while other modules are still awaiting pass 1. I think that's preferable  to a rushed second pass prior to 1.0. And again, there's a snowball  effect: having existing promises makes it easier to make new ones.


# The relationship to 1.0

To many Rust users, breaking API changes feel not much different from  breaking language changes. If we're going to shed our reputation for  breakage, we need to do it across the board.

With APIs it's much easier to deprecate and leave in place, of course,  but I think it's critical that we mark as `stable` a large portion of  the most commonly-used APIs (which brson's data helped identify) prior  to 1.0.

I'm trying to get out ahead of the stabilization process and identify a  roadmap of work to do across the board on APIs. Stay tuned :-)

-----

- brson: I have two points: The whole system is derived from Node, which has definitions of the stability levels. Their unstable level is more strict than that. That's just an observation, we don't have to use the same thing.
- acrichto: Node's definition of unstable is also up to interpretation, "We can break back-compat if reasonable"
- brson: Absolutely. The thing that worries me is that the functionality is what we care about, and not the API. If we had a feature that provided utf16 decoding, and we decided we implemented it wrong and made a different API, I feel like that's not a transition [...] but stretching it a bit.
- brson: I like the idea of a two-pass process. Maybe we will be looser about what this means exactly for a while, and I'm fine with that.
- aturon: The focus shouldn't be functionality, I think that's right. I think it should mean light refactoring, with a straightforward fix. For example making a bare function to a method. I think that is in scope for stable.
- brson: I agree.
- aturon: And when we've made changes to unstable, it's generally been for that sort of reason. I also strongly encourage use to use the notes feature, to keep breadcrumbs on our reasoning. If we're not confident about something, we can say way. This gives us more leeway to give warnings.
- brson: This all sounds good to me. I think we're pretty much in agreement with what unstable means.
- cmr: Is there space for another level between stable and unstable?
- brson: I don't want there to be!
- cmr: Maybe the difference between stable and frozen is what we're looking for.
- brson: Node also has the weird concept of "locked", super stable, not possibly changing it.


# core::cmp

http://doc.rust-lang.org/core/cmp/

https://github.com/rust-lang/rust/blob/master/src/libcore/cmp.rs

Done but waiting on trait reform for something.

https://github.com/rust-lang/rust/issues/12517

Eq/Ord/PartialEq/PartialOrd are pretty well settled.

* module stable
* PartialEq/PartialOrd/Eq/Ord - unstable because of trait reform issue?
* lexical_ordering - not used anywhere. oneliner. what is this for? (deprecate)
* min, max - stable
* Equiv - experimental
* Not sure about impls because of trait reform changes

- brson: Mostly traits, few small things. Iterated on these for years, and there's essentially no right solution. My understanding is that most involved think the current formulation is "good enough", but that there is some language changes that [...]. You should never have to implement PartialEq yourself, but today we do.
- acrichto: In a world where we derive Eq and never PartialEq, they will have some duplication.
- brson: So there's no inheritance here?
- acrichto: Nope, there will be a default implementation of each for the other that you can override.
- brson: Sounds like there's some changes to happen here.
- acrichto: I was hoping to have the coherence changes by now.
- acrichto: prefer lexical_ordering to be a method
- aturon: expect it to be Ord<Pairs of things>
- spernsteiner: are places where you would use lexical ordering where you can't just tuple the two items
- acrichto: agree. let's deprecate
- aturon: problem with equiv: hashmaps have this problem but current approach involves api duplication
- aturon: cmr, y u no like trait?
- cmr: it's used for questionable purposes
- brson: wart that it exists at all?
- acrichto: true trait is simple, but ...

# core::default

http://doc.rust-lang.org/core/default/
https://github.com/rust-lang/rust/blob/master/src/libcore/default.rs

Recommendation: all stable

# std::task


http://doc.rust-lang.org/std/task
https://github.com/rust-lang/rust/blob/master/src/libstd/task.rs


Observations:

* Committing to calling threads 'tasks' - probably fine
* One of the few uses of builder pattern
* TaskBuilder uses a default typaram, the `Spawner` impl
* Overloadable stdout/err is pretty 'special'; no other custom environment configuration
* fn `named` takes IntoMaybeOwned. MaybeOwned is not loved
* TaskBuilder<SiblingSpawner>::new is odd since it's on a specialized impl
  - compatibility hazards if we ever want ctors for other TaskBuilder specializations
  - doesn't look possible to even construct a TaskBuilder with a non-default Spawner
* TaskBuilder::spawner - why replace a spawner?
* `try_future` only use of `Future`, which is a problematic type. Could return a Receiver instead
* deschedule is unfortunate, but supportable (might rename)
* `failing` is problematic because it depends on task state, whereas unwinding does not
  - making decisions during dtor based on whether one is failing is a bad smell

Recommend:

* module stable
* TaskBuilder, Spawner, SiblingSpawner, stable
* impl Spawner for SiblingSpawner stable
* TaskBuilder::new stable
* TaskBuilder::named: unstable (modulo MaybeOwned changes)
* TaskBuilder::try_future: experimental (futures are experimental) (add try_chan - mark unstable)
* TaskBuilder::{stdout/stderr} experimental
* Other methods stable
* with_task_name: deprecated (add `name`)
* deschedule: unstable (rename)
* failing: unstable (may move)

# alloc::rc

http://doc.rust-lang.org/alloc/rc
https://github.com/rust-lang/rust/blob/master/src/liballoc/rc.rs

Observations:

* Does anybody use weak Rc pointers successfully?
* Servo calls `downgrade` once
* Extra word is very wasteful for Rc-heavy code

Recommend:

* module name stable
* Rc stable
* new stable
* downgrade experimental
* impl Deref unstable (Deref is unstable)
* impl Drop stable
* impl Default stable
* impl Eq, etc. unstable
* impl Show unstable
* everything related to Weak experimental



