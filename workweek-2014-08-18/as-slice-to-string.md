# 2014/08/18 as_slice/to_string

RFC: https://github.com/rust-lang/rfcs/pull/198

- aturon: problem: any code that works with Vec/String has lots of 'as_slice'. Annoying because the distinction doesn't matter. Way worse than borrowing with '&'.
- aturon: used to have coercions for this, less magic now
- aturon: solve ergonomic problem
- aturon: RFC adds slice notation, so `.as_slice()` can be written `[]`.

- brson: have you compared the numbers: removing `as_slice` by using `[]` and implementing StrSlice on String? Both reduce as_slice calls.
- aturon: no

- pcwalton: I want to compare to the version that uses Deref. Same amount of boilerplate: `&*` vs `[]`.
- wycats: `&*` is scarier than `[]`.
- nrc: could also have coercions (cross-borrowing) that don't require this.
- aturon: think that's a bigger conversation.
- aturon: not obvious that Deref will work because method resolution rules may change that would break it.
- aturon: slice notation is good regardless.

- nrc: you want to never think about whether you've got &str or String.
- pcwalton: don't agree. also argues for coercing T to &T that we've not done.
- nmatsakis: I have found it hard to track what is being moved where
- wycats: I think the problem is having the two kind of String/str for normal usage in the first place, hard to teach

...

- nrc: difference between giving an intro talk and actually learning rust. not convinced there's a big deal
- pcwalton: C++ tried to have one string type. disaster because: atomic ref counted ropes. perf characteristics are bizarre.
- niko: isn't String vs &str same as owned vs. borrowed.
- acrichto: diff because of literal syntax
- wycats: you want to teach it as being owned vs. borrowed but you can't
- klabnick: we also have the MaybeOwned type
- wycats: hard to teach it consistently due to inconsistencies
- brson: like what?
- wycats: normally you use `&` to borrow, but in string you start with slice
- wycats: you ought to take `&str` and not `&String`
- acrichto: if you have Box<T> you can do everything with that that you can do with T, but String vs &str has distinct sets of things
- acrichto: lots of methods you have to call as_slice() but with String you can push on it etc
- wycats: conceptually two different types, strings and slices
- wycats: sometimes you take &str because you want borrowed not necessarily a slice
- nrc: ... 
- wycats: &String is a type that exists and arises due to collections
- aturon: yes but that's not so great
- aturon: seems like what alex is getting at is that &mut Vec and &mut [] are quite different in capabilities
- wycats: my feeling is that it is subtle, so maybe we can change way that we talk about it and way that syntax works less subtle
- pcwalton: not sure what that argues for concretely
- wycats: in Go, array and slice, they talk a lot about it 
- niko: I agree we need to document this. I don't think that people will have trouble understanding substringing. This is a borrowed strings. `&str` lets you take arbitrary substrings. The fact we're pointing out that these have different semantics. Having a conversion may not be that crazy.
- wycats: I like that a lot. I think it's important to clarify the distinction between "slice" and "string", and explain that we take "slices" (aka views) to add flexibility for the caller.
- nrc: after doing lots of rust coding. It's really painful to type out `.as_slice()` a lot.
- pcwalton: I'm not sure if writing out a `[]` would be that bad, same with a `&`.
- niko: the syntatic distinction should go for things that are more different.
- wycats: saying `T` is the same as `&T` ... (missed)
- pcwalton: I'm nervous about taking the full plunge to autoref. We haven't tried brackets yet.
- wycats: we do a cross borrow in receiver position.
- nrc: the dot is magic. It can do more than if you're just passing something to a method.
- wycats: good petagogical explaination. But why make you go through this effort when passing something to a method.
- pcwalton: can't come up with a good syntax for accessing methods. Did some soul searching. Couldn't find anything other than auto-deref that works. Thinks like it's a necessary evil.
- nrc: not a good hard argument. dealing with ergonomics. In a receiver position, it's easier, in others it's not. Receivers are special.
- wycats: symmetry leads to refactor problems.
- pcwalton: Not sure if it's fixable without doing ADL.
- wycats: when I say a hazard, I may be misspeaking.
- pcwalton: Name lookup part would still be asymmetrical.
- acrichto: there are some contentious points with the slicing syntax.
- nmatsakis: we have .. in pattern matching, which is inclusive, but slicing is exclusive

summarizing:
- [] for as_slice
- [mut] because we have to have it
- 
