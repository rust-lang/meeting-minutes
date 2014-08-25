# 2014-08-19 Markers/variance

- nmatsakis: I don't like what we do today in the case where a type param or region param is unused/unconstrained.
- nmatsakis: I had a proposal to change this behavior to something that seemed more sensible to me. An alternative would be to just make it an error.
- nmatsakis: The error would say: "this type param is unused, you must use a marker"
- wycats: Could you allow lifetimes on raw pointers?
- nmatsakis: It can still happen that a param is unused, but it'd be much less frequent.
- zwarich: What about variance annotations on the params?
- nmatsakis: What I was considering proposing -- but I want more data -- is that if the inference generally goes one way (say: type params covariant and lifetimes contra) then we make those the defaults and use mut to mark when you don't want that variance.
- nmatsakis: The data seems not 100% one way or the other.
- nmatsakis: There are some tricky aspects. This would only allow you to mark things as invariant, which seems simple/understandable (without requiring people to really grok co/contravariance)
- nmatsakis: But for traits that represent functions, lacking co/contravariance is a problem, which makes me unsure about the proposal.
- nmatsakis: You could instead imagine syntax for both co/contravariance anyway.
- zwarich: You'd probably want variance annotations for HKT anyway. Scala does that.
- nmatsakis: The argument in favor of inference is that people don't like to think about this issue. And Rust, if you avoid cell/refcell, gives you covariance.
- zwarich: The weird thing for Rust is that & is contravariant, which is confusing.
- zwarich: in/out is still better than +/-, despite that fact.
- zwarich: An unused lifetime param makes no sense. Not like phantom types.
- nmatsakis: Bivariance is almost always wrong.
- nmatsakis: The argument against inference is the usual one: if you change the type to include a cell, you've now broken your external interface.
- wycats: I think we want tools to detect breakage anyway.
- zwarich: How does this deal with backwards compatibility?
- nmatsakis: For example, I think we want to warn when you opt out of Send
- zwarich: Will Cargo have a general tool to detect semver violations?
- wycats: That's the hope.
- nmatsakis: Do people prefer inference or explicit annotations?
- nrc: Want as much inference as possible.
- zwarich: I think we want inference but also an annotation rather than a marker.
- nmatsakis: What's the proposed syntax? in out mut? But I don't want out to be a keyword, if we can avoid it.
- pcwalton: Is this backwards-incompatible?
- nmatsakis: Not necessarily. Changing what we do when we have no constraints it backwards-incompatible. But we can make it an error for now, and transition from markers to annotation syntax later.
- nmatsakis: Right now, type params are invariant. We're not using any inference to decide this.
- zwarich: Will we allow bivariant type params?
- nmatsakis: No, always an error. Or at minimum, you have to opt in.
- nrc: Assuming the annotations are rare, just use "covariant" and "contravariant" as keywords
- wycats: It comes up a lot with FFIs
- zwarich: "Longer" and "shorter" for lifetime annotations could be useful, and then "in" and "out" for type params.
- nrc: "in" and "out" don't tell you what's actually happening.
- nmatsakis: It's how I explain it to people.
- nrc: You can probably contrive cases where the variance doesn't match this intuition.
- zwarich: I think in most cases this would lead to invariant constraint.
- nrc: The easy cases, we infer. You're left with this only for the hard cases, where you actually have to think about what it means.
- dherman: There's a big difference between type theory mechanisms and programmer mental models. You want programmers to think about what it means for their interface, not the type theory.
- dherman: I think in/out is problematic for a different reason, for lifetimes it's backwards.
- dherman: But longer/shorter really gets at what you're trying to communicate on an interface. It's OK for the words to be long, since it's a rare case.
- wycats: I don't think it's so rare/advanced. You hit it with FFI
- nmatsakis: Unsafe pointers make it not uncommon.

```
Some statistics:
- in standard library: 85% of region parameters are contravariant, 15% are invariant
- for types, harder to analyze quickly, but: ~50% invariant, ~25% co/contra
```

- wycats: When you're writing FFI code, you know you're escaping normal Rust patterns, but you have to tell the compiler how to think about the lifetimes, so you have to do some manual work.
- zwarich: For covariant lifetimes, you only get it for callbacks, where you're not using refcounting or GCs; that's pretty rare. But maybe it's just because people have shied away from closures.
- nrc: My worry about this is: it's kind of what Java tried to do with wildcards, and that's totally failed.
- zwarich: Wildcards are a lot more complex, though.
- dherman: I think one of the reasons it failed is just that co/contravariance are just hard. But we have to do the best design that we can. We want the most intuitive syntax we can manage.
- nmatsakis: My original proposal was: unconstrained lifetimes are contravariant.
- nmatsakis: Bivariance is useful when a trait has a lifetime, but your impl doesn't need it. But typically you can just use 'static
- zwarich: Can we just drop covariant lifetimes?
- nmatsakis: Yes, but it doesn't solve the whole problem.
- zwarich: For 1.0 we only have variance for lifetime params, right?
- nmatsakis: Yes, the other change is backwards-compatible, so it's just a nice-to-have.
- pnkfelix: For future-proofing syntax, can we make use of the ? with DST for annotations here?
- pnkfelix: The idea was Sized? was just one example, where ? could be used for other things as well.
- nmatsakis: Just to avoid a keyword? Like a magic trait bound? Seem weird.

Lunch!
