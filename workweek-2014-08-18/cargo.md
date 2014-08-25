# 2014-08-19 Cargo

# Agenda

- issues from Jack
- is Cargo a 1.0 deliverable?
- Registry braindump

# Jack

- jack: cross-compiling for Android is one issue I've been worried about
- acrichto: it's there, but there may be snags
... some discussion of details ...
- acrichto: Are you planning on making the whole build system for Servo just "cargo build"?
- jack: platform tests may be an issue
- acrichto: If you set up make to set up variables and call cargo build, that might do it
- wycats: We should make "cargo build" handle whatever Android issues there are
- wycats: What do people do for Android in general?
- jack: Usually using Java, where this isn't a problem
- acrichto: This is a problem for cross-compiling in general.
- pcwalton: Native Android apps are always hacks. There's no good workflow. The platform isn't designed for it.
- dherman: I've heard of people using C++ for iOS/Android portability
- wycats: Seems like state of the art is --android
- pcwalton: The NDK has its own build system, makefiles, etc. Works only for C++ and Google style
- acrichto: Is there a fixed location where the NDK is located?
- jack: We can write it to a static file
- acrichto: We can have a cargo.cfg file that points to the NDK
- wycats: Can we make it more generic: "here is the cross-compiling toolkit for this triple?"
- acrichto: Yes.
- wycats: My worry with --android is that it feels like a common case, but not a special case.
- acrichto: If cargo configure generates a cargo.cfg file, you should just be able to do "cargo build" and be all set
- acrichto: You could build for multiple targets at once, and you don't want to use the same GCC for all of them.
- pcwalton: The best user experience for an andorid dev is: download the NDK, place it somewhere, set a cfg option, and build
[agreement]
- jack: If we can make this work for spidermonkey it'll work for everything. Spidermonkey is special: it builds a build program and a host program (a code installer on the build host) and then builds all the android stuff. So it needs both the system and android cc
- pcwalton: The output of an android is a dylib, never an exe
- jack: Not totally true.
- pcwalton: You can't put exe in the app store unless it's java
- jack: libservo is a dylib, but everything is statically linked
- wycats: The normal cargo workflow is to statically link everything, and then at the endpoint you can decide between static or dylib
- pcwalton: You want to minimize the number of dylibs
- wycats: Since we have no ABI compatibility, there's not much reason to split into multiple dylibs
- pcwalton: And having more dylibs increases startup time.
- pcwalton: The only people who care about it are linux package managers.
- acrichto: With spidermonkey, do you pass in the android root path?
- jack: Yes; we special-case spidermonkey and pass in these arguments.
- wycats: at some point, we should make the ./configure step not be necessary.
- jack: I haven't done spidermonkey in cargo yet.
- wycats: it sounds like you said you need ./configure AND cargo build?
- jack: We have several libs that are just wrappers around C libs, and sometimes servo needs to use newer versions so we package them. In Gecko, they're in tree. In Servo, they're submodules. For Cargo, I'm making dummy packages with a blank .rs file and a C library. The lib fontconfigsys.dylib that cargo generates is empty and unused. But we use it to link to the actual lib.
- jack: You don't actually need a separate configure step because the build command in the sys file runs configure and make for you.
- jack: So I think we don't need a ./configure step except that Servo will probably need it for other things. Gecko uses a system called "mock" that does orchestration like make, and we'll probably use it for Servo as well. It'll probably just run cargo build. 
- jack: The general developer workflow should be "cargo build" and "cargo test". Only for web platform tests would you have to do something more interesting.

- jack: Otherwise, acrichto already fixed most of the issues I had.
- jack: One other question is whether the hack I described above is a good idea.
- acrichto: You're just using a package to build a C library, you're not actually using the Rust library
- wycats: We want you to be able to use cargo build to get .a files and use those for subsequent scripts.
- jack: The problem is that you have to generate a rust artifact.
- wycats: Maybe we just want a flag, or some other way to say that you're doing this. We have to trade off between intentional usage like you're doing, and accidentally forgetting to put in a library.
- jack: The dependency tree doesn't involve the Rust side; you don't immediately need or want Rust bindings
- wycats: I think we need to support this pattern. 
- jack: We have both cases in Servo, sometimes the bindings and C library are separate packages, sometimes the same package.
- wycats: We just need a way to turn off the empty library check.

- pnkfelix: How does the rustc invocation learn about the -L flag needed for the library? How do I tell rustc to use a custom library?
- wycats: That's different from jack's case. Your use-case is what acrichto is working on.
...
- wycats: We don't support placing a dependency in a completely arbitrary place. Basically, we want people to use jack's approach.
- pnkfelix: So I can have it redirect to my custom package?
- jack: You'd just package it up.
- wycats: That way as a user you don't have to figure out where to place random packages.
- acrichto: This came up on IRC the other day. It can be useful for just testing things out.
- wycats: You can make a hacked package with PATH=
- acrichto: But you still have to put it in a cargo package
- wycats: Our path for just testing things out does involve cargo packages; that's where we want things to end up anyway
- wycats: The TL;DR is: there is a way to say "I have a cargo package located in this directory", so you can use jack's hack and point it at your custom package. But when you deploy, you have to upload the package etc.
- wycats: If we have Jack's hack as a first-class thing, we may want to cache things created that way globally. Right now we can't cache dependencies globally. I don't remember what the problems were.
- jack: You could probably cheat by using ccache for this case, and punt on the whole thing.
- wycats: I think you can cache leaf nodes.
- acrichto: You can cache everything, but it gets big quickly.
- wycats: The .a situation is a special case.

- jack: Some of the system libraries we replace are just to make sure everyone has a reliable build of a given library.
- jack: But we might want to use the system library if it's available.
- wycats: That's what link config is supposed to do. For example, a lot of people have postgres.app on their Mac. So we can have a program that checks for it, and hook things up appropriately.
- wycats: Basically, try to use package config and if it gives me what I want, great. Otherwise, fallback to custom lib.
- wycats: But we need to resolve all the dependencies for deterministic builds.
- acrichto: This will be shaky if we lose syntax extensions for 1.0
- jack: We have a bunch of platform-specific deps for Servo.
- wycats: My suggestion is that such packages should wrap themselves in a cfg for the platforms they support, and ALL platforms include them.
- acrichto: It's pretty inconvenient.
- wycats: our strategy is to make a separate top-level module and then just glob import from it.
- acrichto: But globs are feature-gated.
- wycats: The example package is linux-rust, under Carl's github account. It's not currently doing the hack.

- jack: Another issue from the Rust upgrade: Servo embedding requires rpath. Will there be a way to do this?
- acrichto: We don't want to expose the rustc command line. But some flags, like rpath, can be whitelisted.
- wycats: I don't want to support them as command-line flags. But in the cargo.toml file. That way there's no special incantation to build.
- wycats: You may need profiles to make this work well.
- wycats: I think, with linkargs, we'll expose only the two things you care about: what library and where is it located. We'll let you specify those things and then convert into linkargs.
- wycats: We probably don't want to expose LLVM args.
- wycats: The point is, we want to analyze all the things you can do with these flags and figure out the right usage patterns to expose.

- jack: When you first check out servo, it'll have a cargo override that will set things up to use submodules, etc.

- jack: When you run cargo test, is it smart enough not to run tests for other targets?
- acrichto: Not yet.
- wycats: A reasonable starting point would be to match the current target exactly.
- zwarich: You care about compiling and testing 32 bit on a 64 bit machine, though.

- jack: rust-http generates code to compile. If you opt out of cargo's building, then you have to invoke rustc yourself (if the code generator is written in Rust).
- jack: It'd be nice if cargo supported this, and could compile/run the generator for you.
- wycats: We've talked about this and probably want to support it.
- acrichto: We need to know the outputs, and the inputs that are relied on, for dependency tracking. That's the tough part -- knowing when to re-run. 
- jack: Cargo needs some way to know the mapping from the generator to the generated files.
- acrichto: Ideally, you write a syntax extension. But they're unstable and painful to write.
...
- wycats: You could have the code generated into a separate crate and then use extern crate to bring it in.
- jack: What about cyclic deps?
- wycats: In that case, you probably want it in your src directory.
- wycats: The advantage of doing it in a separate crate is that rustc already knows how to deal with it. I'd like to understand the need for cyclic deps better.
- erickt: You can run into this with lex or yacc, for example.
- nmatsakis: This is all just a temporary hack until syntax extensions are working.
- acrichto: A syntax extension has include directives for everything it needs. But if you invoke some other process, we lose any tracking on deps.
- nmatsakis: This doesn't solve the dep management problem, but it does take care of "where do we put the output files" -- there just wouldn't be output files in this scheme.
- jack: Syntax extensions work now, or not? I understand they'll be feature-gated.
- nmatsakis: They're not stable now for sure. Not sure if you could do this today. I just don't want to design big parts of the system to work around something that will change in the future.
- jack: This piece of code accounts for 40% of the build time for Servo.
- zwarich: Could we rewrite the binding generation in Rust?
- jack: The Gecko team keeps adding new features, and we pull those in. We'd have to write all that from scratch.
- nmatsakis: Maybe we want codegen as a first-class feature.
- wycats: It seems like it could be a library on top of a syntax extension.
- dherman: In my mind, one of the biggest wins from macros is not having to have additional build config. But maybe it's different when you have a "blessed" tool like cargo.
- wycats: Even there, you have to worry about dependencies, and generation causes problems for that.
- nmatsakis: You need some way to tell cargo what the deps are.
- dherman: It's not the end of the world if we need to build some support for this now, and then later on when syntax extensions are ready it becomes deprecated.
- acrichto: I think we could get all of this today. We'd write a new syntax extension that concats an env variable and a path, and uses that. Just extend the include macro.
- acrichto: Cargo isn't very smart about re-invoking build commands, but we could improve it.

- wycats: Takeaway: we need to work on code gen; we have some ideas about it, but we need to deal with dependency tracking.

# Registry

- wycats: The basic idea for the registry is the same as what every other language means by it.
- wycats: There's a place where you can publish artifacts with the essentials for building a package.
- wycats: Publication gives you a way to say: "here's what I actually need to build this version"
- wycats: You also have a general metadata server that lets you compute dependency information without cloning a bunch of git repos.
- wycats: The general workflow is: you start off on github, and then at some point you publish into the registry.
- wycats: The registry is append-only; you can't mutate packages already pushed. But in practice, people will accidentally upload data that cannot legally be allowed. So then you need a "yank" feature.
- wycats: But you can think of yanks as being append-only. You don't let people replace yanked versions.
- dherman: So it's as if you marked a given version as "illegal".
- wycats: And sometimes people realize that they just made a terrible mistake, and you need some mechanism for blacklisting those publications.
- wycats: We'll probably host this on s3.
- dherman: Social aspects. I had to experience this directly to "get it". What happens socially with a central registry is you create a positive feedback loop and momentum in the community -- you create a community.
- dherman: It's like an app store where people can search for packages in your ecosystem. There are other ways to achieve it, of course, but centralizing it with a simple workflow greases the wheels.
- dherman: Even today, it's pretty hard to find out what's going on in the Rust ecosystem. There are various websites you can go to, but having one blessed central place will help.
- dherman: Gamification can help too -- where the front page shows the counter of packages, and people get excited as that number climbs and approaches other ecosystems.
- dherman: The final piece is, the central repo gives you a place to grab names, which also generates excitement.
- wycats: A side-effect, too, is you get proper nouns for packages.
- dherman: It forces people to give good descriptive names for their projects, which then creates a common vocab for people to use for products within the ecosystem.
- wycats: I think that cutesy names are a feature, not a bug.
- dherman: It forces people to come up with a marketing name.
- dherman: These are all contributing factors to the positive feedback cycle that comes from having a central registry. I think that something really big will happen, along with 1.0, when we have this registry for people to come to. I think it'll accelerate adoption.
- wycats: Systems languages don't have package managers, generally; this is an opportunity to bring modern ecosystem ideas to systems languages.
- dherman: I think Rust will introduce a lot of people to systems programming for the first time. I think our biggest market might turn out to be not today's system programmers, but rather people who thought systems programming was beyond their reach but are being drawn in by safety and modern tooling.
- wycats: That was my experience, personally.
- dherman: I think our community may look radically different after 1.0 from how it looks today.
- dherman: The overall point is that, the combination of 1.0 and a registry will be huge, and will drastically change the makeup of our community. I think we'll open the door to a lot of new systems programmers.

- jack: We don't have a style guide for package naming.
- wycats: Related point. An open question is: should we reserve a prefix for core team-owned packages? Like std-.
- wycats: The jquery plugin repo had a concept of a suite with a prefix, where you could grab an entire prefix and then control those names.
- dherman: Another way to think about this is a hierarchy of names.
- wycats: I don't want a colon prefix or anything -- that gets messy. But I think registering prefixes could be useful.
- jack: Today's Servo probably doesn't need that, but we may break into more packages in the future, so reserving the prefix could make sense.
- dherman: Would we want to allow thing like the owners of a suite allowing others to introduce new packages into the suite?
- wycats: We're using github orgs for the permissions, so just go through that.
- dherman: What about renaming/migration? Do you copy the contents and start at 1.0, or keep the version?
- wycats: You keep the version, for sure. But you may want a way to say that multiple packages support the same interface.
- wycats: You may want a way to redirect packages.
- dherman: You certainly don't want to delete the old package name. Too  much breakage.
- wycats: I want rustc to tell me, in unmangled symbols, the globally-scoped names that a crate introduces.
- dherman: That's nice for us, since globals are quite rare. It'll almost always be a bug if there are overlapping globals.
- dherman: So the two option questions: do we want suites, and what kind of naming conventions do we want?
- jack: I think we should ban the use of rust-. And we should have package-rs be bindings. I also want a -sys suffix for wrappers of system libraries.
- erickt: You need to disambiguate in you github repos, because you're working in multiple languages.
- acrichto: If we have -rs in the crate names, causes problems for extern crate.
- jack: We should just collapse to underscores.
- jack: Currently, we have github repo glfw-rs, with a cargo package name glfw (since that's in the context of rust)
- nmatsakis: I'd rather have all the names be the same.
- dherman: Are you saying that in the registry, you need two separate packages? One that just bundles the native package and the other is the Rust bindings?
- jack: That's needed for Servo, in a few cases (freetype, glfw, fontconfig) -- those are libaries that may be broken upstream.
- brson: Cargo does allow the package and crate names to differ?
- wycats: No, they're the same.
- jack: To be clear, worried about conventions/banning only in the registry. For github, do whatever you want.
- jack: I think we should ban "rust-" and I think we should suggest "-rs" and "-sys" suffixes.
- dherman: The more important thing is to write out guidelines; people want to follow them.
- jack: In Cargo, you want the name to just match the C name -- you already know it's in rust. But you should name your repo -rs or -sys
- wycats: We do want a -sys suffix that shows up in the registry. Different from -rs that's only in the repo name.
- jack: The only downside of the binding naming is that it's a pain if you're using rustc directly.
- wycats: Cargo encourages duplication anyway, so this is a pain regardless.
- erickt: What about the case where I want to fork?
- wycats: Prefix it with your name. If you're doing it on github, just change the toml file.
- wycats: But if you want to publish to the registry, you'll have to pick a new name and people will have to point at it directly.
- wycats: The issue is, you might want to change a transitive dependency. We discussed some solutions to that earlier, but it adds a lot of complexity. We should hold off until it's clearly needed.
- wycats: There'd be a mechanism to say: if you ever encounter a specific dep, redirect to another one. And cargo would check that this is sane.
- erickt: Sometimes you need to have a private fork while you're waiting for a patch to land upstream.
- wycats: At the top-level, it's really simple.
- wycats: The typical workflow is just forking on github and then pointing the toml at it. I think changing transitive deps is pretty rare.
- jack: We create the git fork, make it a submodule, and then use cargo overrides.
... some discussion of naming conventions: _ versus - ...

... discussion of which of rustc and cargo are responsible for name "translation" (e.g. changing underscores to hyphens)

- wycats: One worry is that in the future we want cargo to support more sophisticated lookup/replacement rules
- dherman: We're just talking about a simple desugaring prepass.
- zwarich: There are system libs in OS X that have underscores in their name.
- acrichto: But we won't ever link directly against those.
- pnkfelix: So the translation is applied only to the identifer, not to any custom crate string?
[yes]
- nmatsakis: Either we use _ in package names, or we translate _ to -. I would prefer the latter.
- wycats: Cargo should lint on underscores for registered package names. Probably on cargo new as well.
- dherman: So the resolution is: we desugar _ in extern crate name to - and Cargo lints on _ in package names? Convention over configuration.
- wycats: Just need people to internalize the desuaring.

Back to registry:

- wycats: The plan of record is to keep metadata locally, using git repo, so that updating metadata is a git pull.
- wycats: We plan to have some checksum so that we can confirm that the package we get from s3 matches.
- wycats: For security purposes, having the checksum and packages live in different places helps guard against compromises.
- wycats: Probably, every year or so, we'll want to collapse the metadata logs to keep git running well.
- stuart: You can just do a shallow clone.
- wycats: Then we don't have to snapshot ever, we can always shallow clone.
- wycats: It seems like git will do all the incremental updating infrastructure that you'd need. I did a test with 100,000 packages and it seems plausible.
- wycats: Probably Bundler's biggest pain point is getting the metadata API to be efficient. Having all this stuff locally seems better.

# Cargo 1.0 deliverable/semver

- wycats: What does semver mean for cargo? Do we care about observable behavior of cargo? Output locations? Flags?
- wycats: For example, we've repeatedly changed the exact output location, and I want to keep this flexibility, but you might want to write tools on top of Cargo that need to know that location. Having some ability to query location is probably OK.
- wycats: Point is, we need to define which of these aspects is considered stable.
- wycats: I would suggest follow the same route as the discussion yesterday; we just need to be clear about what's reliable.
- jack: Are we going to ship Cargo with rustc nightlies?
- wycats: I think we should do it now.
- acrichto: The automation behind that is tricky, because you have to unify two separate system/builds.
- wycats: Already part of rustup. But not good enough for Windows.

