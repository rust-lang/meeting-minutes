# Rust macro discussion

# Agenda

- jclements: Need to produce document that suggests future macro plans.
- jclements: Interested in getting others' opinions about macro priorities.
- jclements: I was also hoping disnet can comment on the general model from sweetjs, and how much work it might be to translate that to Rust (which he doesn't know much about)

# brson's 'vision'

- brson vision: priority: un-feature-gating macro-rules. doesn't matter what changes we need to make, as long as they're sensible and can easily be modeled.
- brson: I'm willing to have a macro system that's not pure and beautiful -- as long as the limitations are clear.
- jclements: so clarity, ease of modeling.
- brson: procedural macros I see as lower priority; others see these as the right way.
- brson: my feeling is that procedural macros are too far off, while macro rules are within reach
- jclements: I remember being taken aback by dherman/nmatsakis's attraction to jumping to procedural macros right away.
- jclements: I think proc. macros will nicely supplement what we have and so it's not a bad idea to keep working on macro rules.
- jclements: What about the syntax of the language as it is?
- brson: I believe the parser is in pretty bad shape. I'd prefer it to be more cleanly organized: a separation of the token tree step and the parsing step.
- brson: That seems important, but it's also "just" an implementation issue.
- jclements: What about the places where macro can appear? Do you care?
- brson: I don't care that much; we should avoid future creep.
- brson: One general sentiment: macros are *already* crazy powerful. We don't need to add more power, it's already waay more than systems programmers are used to.

# acrichto's concerns

- acrichto: I have two big concerns.
- acrichto: First, macro import/export. Right now, it's just a global namespace, which goes against our module system.
- acrichto: Therefore, you can't selectively import/export or filter.
- acrichto: The other is hygiene for paths in general. Some specific cases are well-supported now, but it'd be great to have the general story.
- brson: So, right now everyone who consumes a macro has to create a fake namespace to make macros resolve correctly. That's terrible, but the pattern is at least understandable. It wouldn't be the end of the world to keep this style.
- brson: But I do agree with these concerns.
- acrichto: A macro can use things that aren't in the crate itself. It's nice sometimes, but seems wrong sometimes.
- acrichto: In my ideal world, I can only use what's defined at macro-definition time.
- cmr: I think these are easy to solve.
- brson: that's just hygiene, right?
- jclements: Partly. But partly, we need to make sure the definitions are available. If I write a macro in one place and use it somewhere else, hygiene will make sure the name can't be confused with other. But we also have to make sure that the name is available.
- jclements: So suppose I have a macro with some free variables, say imported in that crate's environment.
- jclements: Currently, we have no hygiene: you're just hoping that the call site of the macro has the same imports in place.
- jclements: Hygiene could help ensure that the variables are, at a minimum, unbound. But ideally you'd have a story for (implicit) imports.
- jclements: It may be that for Rust, the imports would have to be explicit.
- brson: Linking to other crates implicitly would be against Rust philosophy, but implicitly bringing names into scope seems OK.
- acrichto: Cross-crate macros are the main thing I care about. If you're within you're own crate, you can just deal.
[ some tutorial discussion about modules vs crates for disnet ]
- jclements: So cross-crate import/export is definitely important, and it sounds like there are some questions about the semantics, i.e. implied imports. I don't know what the right answer is.

# zwarich's concerns

- zwarich: I work on servo mostly, and one problem I've seen in large projects (>1m loc) is that "cute" features like macros, you start banning or restricting as the project gets large.
- zwarich: Are macros going to be robust enough for a servo at that scale?
- zwarich: Import/modularity seems crucial to scaling this up.
- zwarich: Also, macros inhibit other tooling for the language. If you have procedural macros that are basically staging your code, then you can't write a static analysis that's integrated into the compiler. That restricts things you might want to do on large programs.
- zwarich: Another concern: js has some lexical ambiguities that are resolved by parser state (e.g. regex). Rust has similar things; for better or worse, we copied C++-style generics notation, which causes ambiguities. If we get ints in types, for example, things get tricky.
- zwarich: Is there a good solution for macros and dealing with this kind of ambiguity?
- disnet: The problem in js is that you can't lex it without the parser, because / is ambiguous.
- disnet: sweet.js works by holding enough state in the lexer to disambiguate.
- jclements: Right now, the parser knows about the lexer, and it can reverse some of these decisions by re-running the lexer/relexing tokens. Not a great approach.
- cmr: Isn't it simple to resolve it by removing shift-left/shift-right tokens?
- jclements: You can still have spaces in between, which causes people to choke.
- zwarich: This is also a problem for defining a formal grammar for Rust. If you want a formal spec, that's a problem
- zwarich: If it's defined by implementation, maybe that's ok.
- jclements: It sounds like it is important to allow for multiple implementations.
- brson: Agreed.
- jclements: You talked about the difficulty for larger codebases. I think it's important to distinguish different macro uses. One is things like println! and try! and fail!.  If you're adding a widely-used feature that is technically a macro, but basically part of the language, by and large those don't decrease readability. There's a second family of macros, though, where you're abstracting over something you've written several times (DRY). That second kind of macro can present serious difficulties. It can impede readibility.
- jclements: I'm definitely guilty of writing macros in this latter family.
- jclements: You might want tool support that allows you to avoid cut-and-paste, but see the expanded thing.
- zwarich: There are many cases in Rust history where we could've changed the language, but would need good IDE support. But we've always sided on very simple editors being viable for writing Rust. Maybe this support exists for Racket, but it's not there for Rust.
- zwarich: The nice thing is we do have ! to designate macros. That may be a saving grace. But I know there's some talk of losing the ! and that, I think, would be a big problem.
- zwarich: If macros look just like functions, it'll get very painful to read code.
- zwarich: I'm not sure what the final answer should be, but I definitely appreciate the ! right now.
- jclements: You don't have to get into fancy tooling support if you had a pretty printer that supports hygiene -- part of the build process is to generate a final, readable source file.
- disnet: In sweet.js we lint after expansion. You could even use a code map
- jclements: In fact we have nested spans to track exactly that.
- jclements: A quasi-quoting mechanism can address a lot of problems with full syntax extensions -- those can take care of the spans for you.
[ some technical discussion about span tracking/quasi-quoting ]
- brson: The quasi-quoter isn't in great shape; should maybe be dropped.


- aaron: i've only been a lightweight user, but I would echo what brian and alex say: import and export cross-crate seem terribly broken, can't recommend that you export them. smaller thing: as I understand it, the current macro_rules don't let you match at as fine a grain as you might like.  You couldn't distinguish between something being an attribute vs. something else. perhaps the basic pattern-matching is incomplete
- achrictcho: failure of backtracking
- jclements: I think there is a way to define your macros to live mostly in the world of token trees, or in the world of ASTs. On the one side, you might think of a macro as re-arranging token trees. On the other side, you might think about a macro as knowing something about ASTs and manipulating those. Both have advantages. The big plus for token trees is uniformity: the syntax is very simple, and macros can do anything. On the other hand, putting it in the AST world allow syou to be more confident in where your macros are playing out (e.g. only in expr position). For exampe, asking to parse a type next. Disnet, how do you do this in sweet.js?
- disnet: For matching, you pretty much only care about expressions. At the moment, the only thing that's surfaced is: match expr, match id, match literal.
- jclements: We also have types and patterns. There are different parsers for each. I'm not sure about the prospects for unifying those. Do you know what Honu does here?
- disnet: Honu lets you build up whatever you want -- it matches every production of the grammar. Expressions, regex, variable decls etc. Sweet.js could do that as well. You're building up term trees, and those are particular types of the grammar productions, which you can then match on.
- jclements: Is the set of terms 1-1 with the set of AST nodes?
- disnet: Yes. It starts building up the terms, which turn into AST nodes. That's just adding extra source information. An AST is fully expanded, but terms can be partially expanded.
- jclements: So inforestation is dealing with the diffuclty of ASTs but not making the entire problem go away. You still have to handle every kind of node.
- aturon: Do you need all the you-want-it-when machinery for Rust?
- jclements: I think that most of the complexity of you-want-it-when stems from procedural macros, especially those being defined in the same module they're used in. For the moment, our procedural macro system enforced a phase (plugin) distinction. I think when you say phase plugin, you've provided an answer to the you-want-it-when stuff, in that youve explicitly separated the place where the macro is written and where it's used.
- jclements: In Rust, at least, you don't have Racket's notion of "evaluating" a module. You're just providing a main.

- acrichto: One other thing. The experience around a procedural macro is *terrible* today. The whole step of phase plugin, feature phase, compiling for the host arch specifically as a dylib -- all these requirements pile up. It's possible, Cargo has support for it, but it's a real pain. I would like for procedural macros to be much easier to define and compile, with rustc taking care of cross-compilation etc automatically. But it's probably lower priority.
- jclements: My answer is: you-want-it-when! It does add a substantial amount of complexity. But I haven't written any procedural macros. I think you can do it, there's a design, but it comes as a complexity cost.
- jclements: I also am very worried about our current macro export story. It's textually based. I had to do some support recently to keep this behavior, and I really didn't want to sign off on it.
- jclements: The fact that it's a slice of source text means that you'll miss all kinds of things, substitutions, etc.
- jclements: I would rather limit it further than leave what's in there now. E.g., in order to export macros, you have to write it in a separate file that's just dropped in.

# disnet speaks

- disnet: sweet.js's design goals are focused on the "extend the language" usage of macros. We want the extensions to be completely transparent -- no delimitation of macros like Rust's ! and also allow infix macros, operator replacement, etc. We wanted maximal expressivity.
- disnet: If Rust's not interested in that path, then some of sweet.js's techniques won't apply.
- disnet: I feel like the separation of the reader and inforestation works really well (that's not a sweet.js contribution, though).
- disnet: It sounds like you have weird interactions with the compiler, which is hard to spec anyway. Doing the reader/inforestation spit can make it much clearer how things work.
- jclements: Do you know of grammars for js?
- cmr: The spec has one.
- zwarich: I've implemented a js parser. There's vagueness about semicolon insertion. But it's gotten better over time. It's still an English rule over a grammar, but it works.
- zwarich: One other thing I was wondering: the Honu approach of having a bunch of syntactic forms that you can hook into (binary operators, or more generally precedented operators, etc). Is that a sane way to build on top of a C-like syntax? Is it possible to do that for Rust, or are there corners in Rust's syntax that makes that difficult?
- disnet: I don't spend a lot of time with systems languages. But I think you want maximal expressiveness, and then let the team pare it down.
- jclements: I expected that all of Rust's challenges would appear for js as well. But it sounds like you don't have to distinguish e.g. types vs expressions.
- disnet: You still have to care about expressions vs statements. So it happens.
- jclements: Also the < > delimiters. This causes problems for token trees, because those won't be matchable as a sublist. If I see a < in the source, it might be a use of generics or a comparison.
- disnet: Same as / in js. You have to know regex versus division. You know the difference contextually, right?
- cmr: Ye.
- disnet: So sweet.js just looks back a few times and disambiguates. I imagine you could do the same thing.
- jclements: It adds complexity to the reader.
- disnet: That's right -- it has to hold on to some state. But you don't have to look "all the way back".
- jclements: So, zwarich, I think it depends a lot on how much work the Rust team has available to devote to this. Some fairly small steps could dramatically simplify the structure of the parser, making it much easier to maintain. There's some low-hanging fruit there.
- jclements: Even just parsing an "if", I could say: The keyword "if" followed by an expression followed by "a thing delimited by curlies".
- jclements: Whether it makes sense to go all the way to enforestation -- it seems like the macro flexibility isn't a high priority, but it might still offer some benefits.
- zwarich: If you want to define new operators, either you restrict yourself to binary operators and precedence parsers, or you have to do dist-fix (??) parsing. That's what's in the Honu paper. It's pretty awesome that you can define pre/post/infix operators with precedence and have it all wel-defined.
- zwarich: But I'm not sure that you really want custom operators in large projects any way. It can quickly get out of hand.
- cmr: Do you think it's realistic to continue using < > as we are now?
- jclements: Yes!
- cmr: If we want ints in the type system?
- jclements: I don't know anything about that. But I don't think there's much chance of changing this syntax at this stage.
- jclements: But I'm heartened that we could write a token tree parser that can cope with the current syntax.
- brson: Is there a change to generics that mitigates this.
- jclements: It's just nice to know it will be possible.
- aturon: To clarify: we can get token-tree parsing without changing the grammar?
- jclements: That's the conjecture.
- brson: But we could improve things by changing the grammar?
- jclements: Perhaps, but I think people are attached to < >
- brson: If we had an argument that macros can't work with < >, that would be one thing.
- jclements: What I'm saying is, it seems like the parser would be cleaner if it could match < > as a single delimiter. But we know that it's possible to do without token trees.
- brson: The backtracking trick is speccable, though?
- disnet: Yes. It's hard to verify, but yes.
- zwarich: It's hard to specify without providing code that does it. There's no formalism for it.
- disnet: I have a formalization for js that does this, and a proof that shows the disambiguation is correct.
- zwarich: Do you define a language with a grammar and then define a separate disambiguation is correct?
- disnet: Yes.
- zwarich: So, that's still an algorithm. I worry about writing this as pseudocode; e.g. Haskell had a problem with significant whitespace where no implementation ever followed the spec exactly.

# Path forward

- brson: So, can we make a plan at this point?
- jclements: Yes, and that'd probably be a useful thing to do at this point. I will write it.
- jclements: First, I want to summarize the goals.
- brson: Can we do the summarization today?
- jclements: Yes.
