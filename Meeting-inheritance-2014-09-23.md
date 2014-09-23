Candidates
--------------

* Fat objects (https://github.com/rust-lang/rfcs/pull/9 ) - MicahChalmer; champion: pnkfelix
* Extending enums (https://github.com/rust-lang/rfcs/pull/11 ) - bill-myers ; champion: zwarich
* Trait based inheritance (https://github.com/rust-lang/rfcs/pull/223 ) - gereeter ; champion: brson
* Enhanced enums/traits (https://github.com/rust-lang/rfcs/pull/245 ) - nrc
* Associated field inheritance (https://github.com/rust-lang/rfcs/pull/250 ) - kimundi/eddyb; champion: aturon
* Layout inheritance (https://github.com/rust-lang/rfcs/pull/254 ) - rpjohnst; champion: Niko
* [Enhanced enums (https://github.com/rust-lang/rfcs/pull/142 ) - nrc]
* [Virtual Structs (https://github.com/rust-lang/rfcs/pull/5 ) - nrc]


Criteria
---------

* cheap field access from internal methods;
* cheap dynamic dispatch of methods;
* cheap downcasting;
* thin pointers;
* sharing of fields and methods between definitions;
* safe, i.e., doesn't require a bunch of transmutes or other unsafe code to be usable.
* mix of virtual and non-virtual methods at different levels of the hierarchy;
* easy / implicit upcasts;

jdm's DOM example: https://gist.github.com/jdm/9900569

Niko's attempt to dissect these RFCs:

https://docs.google.com/a/mozilla.com/spreadsheets/d/17JfZ-Z9BIgriKqSZ8Am1UPRppcj1qgFR_8_AF_qYzz8/edit#gid=0

Action Items
----------------

pnkfelix: Close RFC PR 9 (good stuff has propagated into other RFCs)
zwarich: close #11 (good stuff has propagated into other RFCs)
brson: Close 223
nrc: close 5 & 142

Minutes
----------

# Goals
 
- pnkfelix: should establish goal of meeting. not clear to me: pick the one we want, pick top contenders.
- nikomatsakis: think neither. I doubt there's a single we would take now, better aproach to figure out what is in-scope, not in-scope, get a general feeling for the direction
- brson: also closing some?
- pnkfelix: figuring out what properties and goals of each are good and bad might help us determine what we want out of final RFC
- aturon: another goal would be to look at each constraint and get a sense for broad approaches. many do have similar themes.
- nmatsakis: trying to transcribe a chart that lists every proposed feature, which RFC's they came from
- pnkfelix: probably we don't want to get into the details of individual RFC's?
- nmatsakis: we do want to summarize?
- pnkfelix: yes, let's not get into the weeds though

# Criteria

http://discuss.rust-lang.org/t/summary-of-efficient-inheritance-rfcs/494/

- aturon: what's meant by 'internal methods' in first bullet?
- nrc: can ignore 'internal'. what's important is the ability to get values of fields and methods without a vcall
- jdm: yes, traits give cheap inheritance but not cheap field access
- nmatsakis: also matter of having virtual methods and non-virtual on different elements of hierarchy?
- nrc: cheap access to 'internal methods' not in list
- aturon: in RFC i'm championing there is a cost to make initial dynamic dispatch, but once you've done that further calls do not. they argue this meets the criteria. do we need to avoid even the initial cost and access the field directly?
- jdm: yes
- nmatsakis: kimundy/eddyb proposal?
- aturon: yes, though they revised to add the feature
- zwarich: two other criteria not on list: 1) implicit upcasting would be desirable, 2) need to implement methods with self being JSRef<T>, not &JSRef<T> or &T.
- aturon: what's meant by 'cheap dynamic dispatch of methods'? is that different from today's trait dispatch?
- pnkfelix: assume our existing trait system is ok here, but we don't want it to be *more* expensive
- nrc: dynamic dispatch should not be slower than C++
- zwarich: in single inheritance case vtable should be laid out same as C++, one level of indirection and a call. related to downcasting because in single inheritance case in a closed world (1 crate) you can do optimizations that make the implementation more efficient than C++ RTTI, matching what most people actually use in practice.
- pnkfelix: zwarich mentioned that upcasts should be easy to write. does the same apply to downcasting? eddyb proposal required adding downcast methods to every type you might want to downcast to.
- nmatsakis: seems like usability is important

Revised:

   * cheap field access from internal methods;
   * cheap dynamic dispatch of methods;
   * cheap downcasting;
   * thin pointers;
   * sharing of fields and methods between definitions;
   * safe, i.e., doesn't require a bunch of transmutes or other unsafe code to be usable.
   * (new) syntactically lightweight or implicit upcasting
   * (new) calling functions through smartpointers, e.g. `fn foo(JSRef<T>, ...)`
   * (new) static dispatch of methods

# Proposal summaries

## Fat objects 

https://github.com/rust-lang/rfcs/pull/9

- pnkfelix: tries to be modular by solving one problem. proposal: currently we have distinction between thin pointers/fat pointers, sized/unsized types

* Sized types T
* Unsized types U
* Traits are one kind of unsized U
* A ptr `&U` is a fat-ptr, consisting of the pointer to instance of actual type, and the metadata word (a vtable)
* Fat Objects: 
* Fat<U,T> : concrete object type T, preceded by the vtable for the impl of U for T.
* Fat<U> : dynamically sized existential type, like U.  read as Exists T. Fat<U,T>
* but its special because &Fat<U> is a thin pointer.

- pnkfelix: proposal keeps using traits as mechanism for inheritance. to keep thin pointers trait introduces 'fat objects': Fat<U, T> (concrete object type + vtable); Fat<U> like unsized type U
- pnkfelix: appealing idea, small and orthogonal. don't know if this is sufficient to meet the goals we have. don't think it handles upcasts well. think the good parts have already made it to other RFC's
- nikomatsakis: some things I don't like: `Fat` is a very special type, might as well be a keyword; &Trait gives a fat pointer &Fat<Trait> gives thin pointer; lots of variations on traits. usability seems to suffer, could be circumvented with better keywords.
- pnkfelix: `&virtual Trait` might read better

* &Trait -- fat pointer
* &Fat<Trait> -- thin pointer
* impl Trait -- fn<T:Trait>(T)

- aturon: this gives more flexibility wrt thin pointers than needed. for any given trait/impl you can choose whether you want a thin pointer or fat pointer
- niko: reason for this proposal is to make vtables more orthogonal, at a price
- aturon: flexibility doesn't help any known use case
- nrc: should we keep open, not make a decision, or close?
- niko: agree with felix this is subsumed
- pnkfelix: I'll close

## Extending enums

https://github.com/rust-lang/rfcs/pull/11

- zwarich: first round of inheritance RFC's. inspired later RFC's but doesn't meet criteria
- zwarich: adds nested enums, structs as leaf nodes of 'enum tree'. doesn't completely unify structs/enums. adds extensible open enums, open question about whether it's open to other crates; standalone structs can inherit from enums; adds enum variants with generic parameters that are independent of the enum (implications are not fleshed out).
- zwarich: there's two variants of impl used to make trait implementations on these. one is 'impl as match' on enum, pattern matches on all enums and combines them into one impl. another is 'impl .. use (...)', which allows for forwarding of a trait to an path from 'self'. details of optimization of vtable aren't explained.
- zwarich: no upcasts are downcasts. doesn't really solve DOM solution
- niko: lots of unanswered questions. whether base types are sized or unsized, more. didn't address thin pointers.
- aturon: have the good ideas been propagated to others? can we discard this?
- zwarich: yes I think so

## Trait based

https://github.com/rust-lang/rfcs/pull/223/files

- brson: This builds on the Fat Objects RFC. It takes that proposal and adds solution for up/down casting and structural inheritance
- brson: Casting. It defines a trait that wraps transmute for casting. Then it defines another trait, Extends, which you can use to upcast pointers. That trait is implemented *automatically* when a struct is annotated with the first field as being a super type. There are several ways to do that. The discussion in the RFC has moved it to a better design.
- brson: There's a way to bundle methods with objects. It's the same as Fat
- brson: Otherwise, there's downcasting, but as the author says it's a bit half-baked
- niko: Similar to what we have with Any
- brson: Some discussion about whether Any is suitable or needs to be adjusted
- brson: In general, this is a bunch of building blocks that can be used to do some OO, but it's not a class system -- it's just a bunch of pieces.
- niko: Didn't have a strong reaction.
- brson: It's like the Fat proposal, basically; lots of orthogonal pieces but not terribly ergonomic
- niko: I'm wary about there being so many moving parts; it's a strain to see how all the parts fit together
- brson: Right, nothing jumps out and says: this is how you actually write code
- nmatsakis: Of course, that's an explicit design goal; pursuing orthogonality can be worthwhile
- pcwalton: A meta-point: there's a certain feeling that ease of use is not a goal because people don't like OO and want to guide you toward other features
- nmatsakis: I'm not very sympathetic to that
- brson: There is an example in the RFC, but it's pretty hard to get your head around
- nrc: Should we close now?
- nmatsakis: Yes.

## Meta discussion about goals: ergonomics

- pcwalton: ease of use of creating OO structures is desirable. would like that to be an explicit goal and reject the idea that this feature *should* be complicated because we *don't want people to use it*.
- aturon: ideally we have ergonomic OO that is a natural extension of the existing language mechanisms. making something unergonomic on purpose is bad design

## Enhanced enums

https://github.com/rust-lang/rfcs/pull/245

- nrc: evolved from 142, which is a less good version of 245
- nrc: structs and enums are more powerful, unified (struct and enum are synonymous), both nested, named fields, unnamed fields, variants. simpler but more powerful data model
- nrc: don't have to have these lexically nested, can use extension (struct Foo: Bar) to make it look like inheritance.
- niko: are struct/enum actually synonymous?
- nrc: with the latest update, yes. was pushback to them being almost the same but different
- nrc: enum variants are useful as types, useful independently. once variants are types you have to be able to implement traits on them. if trait is implemented for the enum, then also for the variant. As extension: if you implement a trait on Option, then also on None, it overrides the impl on Option, embellishing idea to leave some specified and some not, gives you inheritance (brson is lost)
- nrc: new annotations: `closed` on a (?) means you can use efficient vtables and thin pointers because type can't be extended; `unsized` on types or even variants means they can be minimally sized.
- niko: do types have to be marked `closed`? what's that mean?
- nrc: yes, only closed types can implement closed traits. impls for closed types have inline vtables. for enums and structs the type tag will be a vtable
- niko: have feeling that enums and structs are not meant to be unified because e.g. taking size of maximum and this issue with vtables (not sure what i mean)
- pnkfelix: there's a restriction that these extensions only go in the same crate. necessity to know the discriminant size. that's *the* reason for the restriction, but not mentioned in proposal
- nrc: 

...

- pnkfelix: this RFC talked about drop semantics, which I appreciated
- nrc: Can I close 5 and 142?
- niko: I think so, but I don't know which I prefer
- pnkfelix: prefer to close because I found the abundance of choice confusing




