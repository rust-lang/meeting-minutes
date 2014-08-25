# 2014-08-20 Lifetime syntax (the Tick!)

- nmatsakis: pcwalton had a proposal to drop the ' from lifetime syntax
- nmatsakis: Many people find it ugly. It breaks simple editors. And if we move to a more uniform, HKT world, it's kind of odd to have different syntactic categories for things that are all really part of the type system. Sort of like having sigils to distinguish arrays and scalars -- it's odd.
- nmatsakis: On the other hand, it makes lifetimes marked, and it would be hard to take it out. Also, lifetime elision makes the problem less acute. We need to make a decision.
- pcwalton: Tricky politics here.
- nmatsakis: Are there other syntax changes beyond dropping the tick? Seems relevant
- pcwalton: `a &T` versus `&a T`

.. discussion at whiteboard ..

(other ideas: `(a) &T` or `[a] &T` ... lots of others)

- zwarich: I think putting the & first makes sense -- it's the most important part of the type.
- pcwalton: I argued that removing the ' would not hurt people's ability to read code, because in practice you use case to distinguish types and lifetimes anyway.
- pcwalton: Some people thought that was not a valid argument, because the compiler doesn't look at case. But in practice, we also use case to distinguish nullary enum variants vs variables, and that's worked really well -- but then people complained about that practice.
- pcwalton: I prefer not having the ' because of the visual improvement

```
fn foo<'a,T>(x: &Foo, y: &'a Bar) -> &'a int { ... }
    
fn foo<a, T>(x: &Foo, y: a& Bar) -> a& Bar { ... }

fn foo<a, T>(x: &Foo, y: a&mut Bar) -> a& Bar { ... }

fn foo<a, T>(x: &Foo, y: &(a Bar)) -> &(a Bar) { ... }

fn foo<a, T>(x: &Foo, y: a &Bar) -> a &Bar { ... }

fn foo<a, T>(x: &Foo, y: a &[Bar]) -> a &[Bar] { ... }

fn foo<a, T>(x: &Foo, y: a &str) -> a &str { ... }

fn foo<pool, T>(x: &Foo, y: pool &Bar) -> pool &Bar { ... }

fn foo<a, T>(x: &Foo, y: a&Bar) -> a&Bar { ... }

fn foo<'a,T>(x: &Foo, y: 'a &Bar) -> 'a &Bar { ... }

fn foo<a, T>(x: &Foo, y: &a Bar) -> &a Bar { ... }

fn foo<a, T>(x: &Foo, y: &(a) Bar) -> &(a) Bar { ... }

fn foo<a, T>(x: &Foo, y: &.a Bar) -> &.a Bar { ... }

fn foo<a, T>(x: &Foo, y: &_a Bar) -> &_a Bar { ... }

fn foo<a, T>(x: &Foo, y: &:a Bar) -> &:a Bar { ... }

fn foo<a, T>(x: &Foo, y: &a: Bar) -> &a: Bar { ... }

fn foo<a, T>(x: &Foo, y: (a) &Bar) -> (a) &Bar { ... }

fn foo<a, T>(x: &Foo, y: &(a) mut Bar) -> &(a) int { ... }

fn foo<a, T>(x: &Foo, y: Ref<a, Bar>) -> Ref<a, Bar> { ... }

fn foo<a, T>(x: &Foo, y: MutRef<a, Bar>) -> Ref<a, int> { ... }

fn foo<a, T>(x: &Foo, y: &<a> Bar) -> &<a> Bar { ... }

fn foo<a, T>(x: &Foo, y: &<a> mut Bar) -> &<a> int { ... }

fn foo<a, T>(x: &Foo, y: &/a Bar) -> &/a Bar { ... }

fn foo<a, T>(x: &Foo, y: a/&Bar) -> a/&Bar { ... }

fn foo<a, T>(x: &Foo, y: a/&mut Bar) -> a/&Bar { ... }
```

- nmatsakis: I wish we had a subscript. But parens are suggestive.
- zwarich: It's weird for the parens only be for the lifetime variable -- really, & is a type constructor with two arguments.
- pcwalton: I've done a thought experiment: if the C++ committee decided to add lifetimes, what would they use? I don't know the answer, but somehow that argued for parens for me... but I'm not sure.
- dherman: An idea implicit in the parens is, instead of having a sigil on the name of a lifetime, think of it as providing more information on the &, so you really want to mark it somehow. Imagine it was &:lifetime T. So the lifetime itself is still a bare identifier.
- pcwalton: I'm nervous about having two punctuations next to each other.
- nmatsakis: We may want these in an expression context.
- dherman: I think the whitespace convention is really important here for readability: `a &T`
- erickt: If we want to add HKT, would these syntaxes support it?
- zwarich: With HKT you have to be able to mark lifetime kind
- nmatsakis: This approach is using inference to determine the kinds. But we may want an explicit way to mark the kind, and we may need that as part of the associated items proposal anyway.

```
struct Foo<type T> { ... }
struct Foo<lifetime a> { ... }
```

- nmatsakis: How much work is this?
- pcwalton: We could have a lint to help automate lifetime elision, and then we'd only have to deal with the remaining few cases.
- dherman: even though it seems minor, dropping a sigil can go a long way aesthetically.
- pcwalton: Totally agreed, but we need community buy-in.
- acrichto: I'm not arguing against it, but I'm not sure I see the argument in favor other than "it looks better" -- a big change for such a small justification
- zwarich: I'm not bothered by the current syntax, and I like it better than the proposed syntaxes.
- pcwalton: The reddit consensus is that the current syntax is bad, but no one knows a better one.
- dherman: Despite a lot of familiarity with ML in this room, that's not a very widespread constituency. Outside of the ML universe, ' has a very strong association with string/char literals. So it's very jarring to use it -- especially in this unbalanced way -- for something unrelated to text. It's a mixing of incompatible traditions.
- dherman: Moreover, though I can't prove it, I think in syntax design for allowing people to read "fluidly" without getting tripped up, you have to strike a balance between (1) having enough visual interruptions that it's not just a sea of text (e.g. coffeescript or applescript, where everything is words) and (2) having too much differentiation ("line noise")
- dherman: So you want to be able to break up the words, but not get too noisy and lose the content. We have & serving as the sigil giving you the "macro" structure -- a pointer -- and the words give you the rest.
- dherman: The important point is: "this is a borrowed pointer", and the & gives you that. The ' is not really adding something that's worth additional visual distinction.
- dherman: Also, there are some practical, tooling issues.
- dherman: Finally, every time we remove places where we look visually distinct from C/C++/Java traditions, the more approachable we've become.
- steveklabnik: I've heard GO people use this kind of thing as a justification for not having generics
- zwarich: I think sticking with < > is way more visually disturbing
- zwarich: The line noise argument is just so subjective.
- dherman: It's easy to write off any syntax discussion on this basis. It is hard to find slam-dunk arguments. But I think you can think about it rigorously -- you just have to recognize that the data is noisy and that there's subjectivity involved.
- dherman: I think there is something to the "striking a balance" point; there are clear outliers like Perl (and also Haskell) that are very noisy.
- dherman: It's an information density point. We want to find the sweet spot in the entropy range.
- zwarich: I agree with the basic principle, I just think that Rust has bigger problems on this front. For example, you rarely nest ticks like &'a &'b T. Whereas generics get nested a lot. Or things like &**.
- nrc: We only have so much political capital for wide-ranging syntactic changes. Is this where we want to pay that? It's going to be hard work.
- dherman: I think we can make unpopular changes and still be OK. People will move on, and we can improve our ability to handle it. You have to worry about the future community, too. The important thing is to make sure the community can give feedback and be heard.
- nmatsakis: I think a worthwhile pre-step is to do the patch that removes elidable lifetimes
- pcwalton: Mostly what's left is struct definitions
- acrichto: And impl headers
- zwarich: Editors are not going to be able to separate lifetime variables
- nmatsakis: Not as well as today, but they can still do it
- pcwalton: I thought `a &T` would be great for editors, but bitwise & is a thing...
- pcwalton: IN theory we could change this post-1.0, by making ' valid in identifiers
- nmatsakis: yes, but better if we decide now.
- nrc: Especially with the line noise complaint
- nmatsakis: I'm wondering what data could help. Certainly applying elision everywhere would help isolate examples... but probably we just have to make a call.
- pnkfelix: What's the problem with parens?
- dherman: Meaningful parens are werid.
- nrc: Also confusing with tuples.
- nmatsakis: Not ready to make a decision.
