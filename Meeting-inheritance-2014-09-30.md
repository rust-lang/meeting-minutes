## Goals

- nrc: niko wants to postpone some or all discussion on this until december or later
- nrc: last time we summarized existing rfcs except two, so we can summarize those two as we did last week. then decide how to move forward
- aturon: have question about design criteria. is it required to be able to have a virtual struct hierarchy where you have a method that is dynamically dispatched higher in the hierarchy, but lower in the hierarchy becomes statically dispatched
- nmatsakis: i think so

# 250 https://github.com/rust-lang/rfcs/pull/250

- aturon: takes general approach of extending existing features in incremental ways, with some extensions we might desire anyway, with the intent it can be used to encode the dom example. RFC is structured as a sequence of RFC's.
- aturon: all dynamic dispatch, polymorphism goes through traits. no new additions to struct/enum. most extensions are to traits
- aturon: first feature allows upcasting of trait objects by changing how current vtables are laid out, lets upcasts be no-op in common case, but overhead with multiple trait impls
- aturon: adds assoc fields, ties arbitrary lvalues to trait impl. common case is trait impled by struct, trait ties the abstract field to concrete field. lots of knobs. can insist on specific layout to force optimal code gen. some open questions like how to make it compatible with the borrowck. addresses low-cost field access, seems like it works, but lots of knobs
- aturon: introduces internal vtables, as opposed to today's fat pointers. main proposal of RFC. quite complicated, introduces intrinsics and types to interact directly with the vtables. you explicitly ask for and work with the vtable. not a strong reason to expose this, can be done with attributes or other syntax to request internal vtables. design is questionable imo, not fully fleshed-out. orthogonal to other parts of proposal though
- nrc: could take closed traits proposal from 245 to do this
- aturon: yes, modularity could allow parts to be swapped out
- aturon: remaining features re inheritance. things get harder to work with here imo. split between data inheritance and behavior inheritance: when defining a struct you can 'splice' in another, inlining the other struct. independent of the trait hierarchy, means duplicate hierarchies for data + inheritance
- aturon: implementation inheritance: proposal extends trait default methods such that when defining subtraits you can override methods from supertraits. you provide a new set of default implementations for those methods and [? brson spacing]
- aturon: lot's of complicated considerations here
- aturon: this is how they provide successively more specific implementations as you go down the hierarchy
- aturon: it's not clear that this addresses the issue of changing from dynamic to static dispatch as you get lower in the hierarchy. at some point in the hierarchy you override the default of a parent, you never override again lower in the hierarchy, but the compiler doesn't know that so you don't get static dispatch. so if you want to close the hierarchy you need a different mechanism
- pnkfelix: e.g. Java's final
- aturon: yes, can't seal off further overriding. if you add a mechanism, gets closer to nick's proposal.
- pcwalton: disagree with goal of discouraging OO
- brson: was that an explicit goal?
- pcwalton: "People coming from other mainstream languages are usually familiar with type inheritance, which will make them overuse this feature"
- nrc: Agreed that we consider discouraging OO a non-goal
- nmatsakis: If this was proposed as another feature we'd close it as postponed.
- pcwalton: I'd like to make it clear that discouraging OO is a non-goal
- pcwalton: I don't like the vtable proposal here because it seems like it's intentionally difficult to use, which makes it hard to use OO when you need it!
- nrc: last week we closed a bunch saying that the good bits are already in other RFC's, and there's nothing left in those we need to keep in mind. In this case there are some good ideas we'd want to keep track of
- aturon: We have an issue for virtual structs in RFC repo?
- nrc: non
- aturon: I propose we postpone *all* RFC's, open issue with summary, make clear the timeline for resuming
- nrc: sounds good. I'd like to talk about backwards compatiblity hazards though
- nmatsakis: which backwards compatibily hazards? do you have on i nmind
- nrc: no, but worried that those we discovered for (245?) were subtle. worried that there may be others we're not seeing, should consider them now

# 254 https://github.com/rust-lang/rfcs/pull/254

- nmatsakis: 4 major parts
- nmatsakis: 1) allow structs to inherit with colon notation, no semantics other than shorthand for installing the fields in the right spots. since struct layout is undefined i guess this also adds some guarantees there
- nmatsakis: 2) traits inherit from structs. if a trait inherits from a struct then all things that implement that trait must also inherit from the struct. creates a 'shadow' trait hierarchy, mirroring data
- nmatsakis: 3) upcasting and downcasting, less detailed than eddyb/kimundi rfc. downcasting overloads match with `as` patterns. i'm dubious about parts: if in `match` you do `as Trait` that matches anything that implements that trait, believe it's a dynamic check, missing detail
- nmatsakis: 4) finally it includes the fat-object proposal from last time to cover the internal vtables
- nmatsakis: I think there are a number of subtleties uncovered here, e.g. mixing static + virtual dispatch: can implement methods on the struct type that are statically dispatched, but then you lose your vtables
- nmatsakis: I generalyl think traits inheriting from structs is inferior to associated fields.
- nmatsakis: upcasting and downcasting is similar to other proposals, as were vtable parts, so not a lot of unique features
- nrc: So the best parts are already covered better in other RFC's?
- nmatsakis: imo

# Planning

- nrc: stop now, postpone remaining RFC's as postponed, create issue summarizing current progress, put off further discussion until december.
- nrc: are there other things worth discussing now?
- nrc: jdm are you ok postponing until dec?
- jdm: yes
- nrc: worry about backward compat: since these are post 1.0 we want any changes to be backwards compatible. any parts we might decide in december not compatible would need to be implemented quickly
- nrc: One thing we've done already is to make enum variants members of type namespace
- nrc: another: if you allow enum variants as types you can affect inference, causing subtle errors, but an edge case perhaps
- nrc: can address this by making enum variants usable as types now
- nmatsakis: not willing to put a lot on the 1.0 schedule. we can work around best we can
- acrichto: this is such a major feature, that opting in to different semantics isn't end of world
- brson: could be gated all the way to 2.0
- nrc: whichever proposal we go for there are a few keywords we might want: override, abstract, overriding. seems like we might want these. how about reserving them?
- pnkfelix: I'd want `final` too.
- nmatsakis: what is `overriding`?
- nrc: can't recall offhand. may want a keyword that says 'this method *may* be overriden' and one that says 'this method overrides a super-type method'
- nmatsakis: usually `virtual`/`override`
- acrichto: it's inevitable that some feature will require new keywords and whatever solution we need there should work here. reserving keywords can lead to bikeshedding
- aturon: what is the timeline for servo?
- jdm: we have some gecko experts that don't want to work on servo without language features to allow this. to me it seems like we want to resolve this if we want to start putting more resources into it
- zwarich: might also be required for perf at some point
- aturon: if we need to get this out quickly we may not want to commit to implementing the infrastructure for adding new keywords, might want to just add the new keywords to accelerate our ability to move on this
- nrc: seems we've flip-flopped on the priority a bit, we might want to work on this immediately after 1.0. this may be a special case
- nmatsakis: i'm ok with 'abstract'/'final'/'override'
- nrc: for eddyb/kimundy proposal do we need more keywords?
- aturon: not as proposed, though they intentionally did not introduce new keywords (except override)
- acrichto: i think they added 'default'
- aturon: unclear
- acrichto: sounded like it had to be one to me
- nmatsakis: there are so many keywords one could use
- nrc: can i make an rfc to reserve those kw?
- nmatsakis: yes
- nrc: other backwards compatibilty hazards?
- aturon: not that I'm aware of but I can take another look and think about it
- nrc: summary: we're going to have a big discussion on this in dec; aturon will look for backcompat issues in kimundi/eddyb; nmatsakis will close his; we'll close remaining as postponed; i'll create issue summarizing
- nrc: what about your proposal?
- nmatsakis: i'll write something before dec. easier to hammer out in person
- acrichto: should we back out current virtual struct support?
- nmatsakis: it's feature gated and fundamentally buggy. fine to back it out
- aturon: open an issue about it
- aturon: nrc you've created discuss threads, let's make sure there's a clear message acknowledging the effort people have put into this
- nmatsakis: the new issue can call out the most prominent aspects of RFC's




