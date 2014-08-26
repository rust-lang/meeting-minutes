# 2014-08-20 Associated Items

# Associated Items

- aturon: The RFC is pretty big, let's give the highlights and focus on the contentious points.
- aturon: The RFC is about two things, associated items (for associated types) and multidispatch traits.
- aturon: We already have one form of associated items with static functions, and the proposal is to allow other kinds of items to be included in traits. Each impl gets to specify its choice for a type (or function), and you can reference the item through the trait.
- aturon: There are various reasons for associated types. For example generic graphs have several types involved, the graph, nodes, edges, etc. The whole graph comes as one kind of package which is encoded as one trait with a few associated types.
- aturon: pcwalton has already implemented some associated types in some aspects, (needs fixes).
- aturon: Remember that associated types want where clauses because bounds generally refer to the Self type so you need a where clause for the associated types to have bounds.
- aturon: There are other items in the proposal like associated statics as well, although they are fairly straightforward considering associated types

# Multidispatch

- aturon: Trait dispatch today is completely determined by the Self type, and nothing else. This is not sufficient to handle everything that we'd like.
- aturon: The canonical example is that you can't impl the Add trait for all combinations of types you'd like.
- aturon: There are a few ways to encode this sort of ability, but the RFC ties this with associated types.
- aturon: There are two ways to connect non-self types to a trait. One way is that a trait can have generic type parameters (like today), and the other is that there are associated types. The division will be tied to the dispatch logic. Associated types are "outputs" which are determined by the input types (type parameters). If you want to do binary operators, for example, the trait Add would take a type parameter for RHS (dispatching on Self and RHS), and the output would be an associated type (uniquely determined by (Self, RHS)).
- aturon: It's simple to understand the dispatch story with this sort of division as today you can't do something like implement the Iterator trait for two different types. With the RFC the logic will actually work because each instantiation of a trait is a distinct trait. Fairly intuitive.

# Issues with this Proposal

- aturon: Two issues. Can associated items have defaults (like default methods). Tricky because if you can give a default associated type, can the default methods assume that the associated type has that default value. The use case for default associated types wants the methods to be able to rely on the associated types having their default values.
- aturon: The proposal is that you can have defaults, but if you override any associated type then you have to also override all default methods and functions. This is one point that's a little contentious.
- pnkfelix: The key point is that you have to override a default associated type, not just any type.
- nrc: Can we add this backwards compatibly?
- aturon: Yes. The motivator is Equiv/HashMap that we'd like to stabilize for 1.0
- niko: I think there is a lot of consensus that the defaults being specified in traits is suboptimal. We may make this the convenient form but provide a more flexible system in the future (post-1.0). I think it's pretty unlikely that this will cause too many problems.
- nrc: It seems that not having default types is less painful than default methods. Whereas with methods there are lots of code, so there is a lower cost to not having default types than default methods.
- niko: That can be argued both ways, however (isn't necessary or doesn't do much harm)
- aturon: One of the other finer points is connected with trait objects. Trait objects are a bit tricky because today the self type and other type parameters are treated quite differently in that only the Self type is erased. Question is how do associated types play into this? The proposed design is roughly equivalent to what we have today where only Self is erased. This means that trait objects must have all their type parameters and associated types specified (so they're statically known).
- aturon: In general to have trait objects and associated types work we have to add sugar (can't use where clauses in an obvious way). The proposal is to add `Box<Trait<T1, T2, K1 = T3, K2 = T4>>` where all type parameters are provided and all associated types are named with their value (ordered by name, not by position).
- pnkfelix: would a default type parameter for K_i allow one to elide the `K_i = T_j` in the Box<Trait<...>> ?

... (discussion about omitting types in trait objects which acrichto missed)

- niko: It seems to be useful to omit types where the default applies most of the time. There are some related issues and this comes up in some other contexts.
- pnkfelix: Type expressions with Self can't work?
- niko: In this particular example, if we have that rule where you could elide, you could not elide Self (?).
- niko: If there is a case where the associated types are not needed, you can make a supertrait that doesn't include the associated types that has a subset of methods and then make an object without the associated types. You can make trait objects which don't have associated types.
- aturon: Almost all traits that are generic today will use associated types, not generics. And those are things you very much want to use as trait objects, so we have to be able to use trait objects with associated types.
- aturon: I want to mention some downsides. There is some duplication of mechanism where there are where clauses and the = syntax. We have to allow that for trait objects, but it's optional when you're not using a trait object. Today you can constrain type variables with trait bounds, but this allows you to write arbitrary equalities between types.
- niko: I don't think this is that bad. I have thought about this and I think that it's ok. I think you can model this with traits if you want to. You can write a trait that forces the types to be unified.
- aturon: The final point is that this really doubles down on treating Self specially. The Self type is an input parameter, but the other input parameters are not on equal footing with Self (today?).

- nrc: I was going to go back to trait objects. I'm not a fan of having bindings for associated types. The alternative is that you can use path-dependent types for any associated types in a trait object.

```
trait Iterator {
    type X;
}

fn foo(x: &Iterator, z: &Iterator) {
    let y: &x.X;
    let z: &z.X;
}
```

- nrc: both `y` and `z` have to be unsized. These are like trait objects bounded by the bounds in the trait.
- niko: Scala has this.
- nrc: You have partial knowledge about types.
- pnkfelix: If it's inherently unsized then you'll lose expressiveness.
- nrc: If you're using a trait object you get trait objects out of methods.
- niko: With DST we should know that a trait object implements the trait itself. I don't think you can achieve that here because a partially specified trait object does not specify the trait. We can't monomorphize based on the trait objects given, so trait objects could not implement the trait. Other than that it seems like we could add this later because it allows erasing multiple things and you can get as much access that you want. This is complex and we may be able to do this later.
- nrc: One alternative is you allow trait objects, but you can only call methods that don't use the associated types. If we do that for 1.0 and add this later, is that feasible?
- niko: Then object types don't implement the trait. This seems like it may be a big problem. There may be workarounds
- aturon: Auto-implementations will have to be magic anyway though, isn't this more magical? Is that ok?
- niko: Yes it seems ok, I prefer the simple rule ...

... discussion about trait objects implementing traits

- aturon: There's the question about when do you need to pin down all the associated types to create the associated objects. Also how do you pin down the types. The `Type = Type` syntax means that you can loosen the restriction. This seems orthogonal, we may be able to add this later.
- aturon: Nrc, is your concern that how you constrain types. or that you have to constrain all of them.
- nrc: I am concerned about how you constrain them. If you write a trait with input type parameters then sometimes you have to use just those and other those plus the associated ones.
- niko: You may want to be able to write a where clause for your example, but it only works if x is not mutable. It's common to say that I want an object that iterates over characters and you do want to specify it (not abstractly). To express that in your proposal you need a where clause for unification. Mutability becomes important quickly (with path dependent types), not a problem with the other proposal.
- aturon: I worry that path dependent types bring along their own problems.
- nrc: Yeah they add complexity.
- niko: They feel like a valuable extension to keep. They're a way to generalize erasure.
- niko: I feel pretty strongly that we should keep the distinction between input and output as clear as possible because it's fairly subtle. I object to moving things into the parameter list when it's actually an output (with trait objects?).
- niko: It seems that the current proposal isn't the best with respect to that particular question.
- nrc: A similarly coherent alternative would be going for the proposal awhile back with dispatch on tuple self
- niko: I don't think it changes much?
- nrc: Only Self is input in that case, you can allow other type parameters other than associated types as output types.
- niko: Then you're forcing people to choose between the list and a parameter. It's a weird choice.
- niko: If you had to guess a syntax you wouldn't guess tuples though.
- nrc: It's what I would guess!
- aturon: The tuple syntax means that tuples are now binders which are weird where everywhere else we bind with brackets (except for Self).
- nrc/niko: I like Self being special.
- pcwalton: I'm proud of Self being special. Haskell people will be angry but they don't know what they're missing.

- pcwalton: How much of this is necessary for 1.0?
- aturon: Good question! Associated lifetimes are not so important. Associated statics are nice but we can live. Multidispatch we really want.
- pcwalton: What about associated items in objects?
- aturon: That seems important, otherwise trait objects are unusable.
- acrichto: I don't think we should forbid an iterator trait object.
- niko: I'm not sure why it's so hard. It doesn't change your basic approach. You've been enumerating these into new type parameters and this is just specifying the values of the type parameters.
- pcwalton: This could be annoying.
- aturon: For library stabilization, the trait objects don't matter so much. We could just make objects with associated types limited.
- niko: We may be able to live, but I'd want it shortly after 1.

# Conclusions

- Let's not change the RFC much
- pcwalton has implemented is 1.0, plus maybe multidispatch.
  - associated types but not other kinds of items
  - object types are nice if we can get it
  - multidispatch is imp't
