# 2014-08-18 versioning

- wycats: this is a proposal to do something like rapid release as the rust release process
- wycats: high-level is that people observe that continuous deployment is a good idea but it has a problem with semantic versioning. When shipping browser or language you also want to give some stability guarantees. Most common way to express that these days is semver.
- wycats: my view is that reconciliing CD -- shipping versions as rapidly as possible -- with semver has some challenge.
- wycats: high-level summary of semver: if you do not inc major version, no break changes.
- wycats: additional parts of this: inc minor version means you can add features, tiny version means no features. not so imp't.
- wycats: some people impl semver by just incrementing the major version all the time
- wycats: but regular human beings actually care about stability
- wycats: reason that incrementing major version all the time feels painful is because it is in fact painful for people to track breaking changes
- wycats: so #1 you do not want to break very often and #2 you want a nice way to signal when it happens
- wycats: some things that you would like to have that might be in tension:
  - continuous deployment: as soon as its ready, make it avail
  - a slow release process encourages rushed features
  - would like to be able to experiment, without experimental features getting coupled to other features
  - want feedback for unstable features (from those who can tolerate instability)
  - at the same time, allow for people to use only fully stable versions
  - want large ecosystem that depends on a stable base (so that it does not force others into instability)
  - want to avoid defacto dependencies on "unstable" things
- wycats: The solution to these problems is "Release channels", like Chrome/Firefox
- wycats: Basic idea: things land in Master, then get promoted to Aurora, then Beta, then Release.
- wycats: Crucially, those three happen in parallel, as a pipeline
- wycats: In practice, channels have different names but work the same:
    - nightly -- the build that last past the tests every day
    - aurora/dev -- master branch, but with a bit of QA over the tests
        (probably with Rust, nightly is enough, since there's no GUI)
    - beta 
    - release/stable 
- wycats: the rules:
    - every feature lands in master behind a feature gate
    - the last passing commit of the day becomes the new alpha
    - every week, core team decides "Go" or "No go" for every active feature gate
    - every six weeks, master becomes beta. Go'ed features are available by default. Other features are not available
    - also every six weeks, the current beta becomes the release
- wycats: An important point is DROPPING features that are not "Go", otherwise you de facto have stabilized them
- dherman: I'm worried that a lot of features will be important for libraries, and we want as big of an ecosystem as possible
- wycats: We should have micro-ecosystems, where a library on Cargo can say "I need this feature" and that will then only work for clients that are on the correct release channel.
- wycats: As feature gates are merged in, you won't have to update the manifest
- nmatsakis: Part of this proposal is that you have to install the alpha channel, or whatever -- a heavy opt-in
- wycats: If you start making exceptions, your ability to say "if you want to use unstable features you have to use nightly" gets eroded
- nrc: When is the feature gate turned off?
- wycats: The next time there's a beta branch.
- wycats: We won't have any feature gates in beta/release (except for legacy cases)
- dherman: So everything new in nightly has an off-by-default flag. But as soon as you get the "Go", there's no more feature flag -- it's on, and promoted
- wycats: You want to be able to revert, so you need a feature flag.
- wycats: You want the same thing in your code to be a feature flag for nightly, and an ifdef for beta/release
- wycats: There's an out-of-band way to set the Go/No go status
- dherman: This out-of-band data is the only thing that changes at a Go/No Go meeting
- nrc: Are beta and nightly separate branches?
- wycats: Yes
- nrc: Bugfixes just sit on nightly and get lifted out?
- wycats: Yes. In Ember, we have some automation with tagged commited "BugfixBeta" "BugfixRelease" and the CI will merge automatically. (These get extra scrutiny)
- nrc: Is there a technical reason to having a weekly Go/No go meeting (rather than six week)?
- wycats: Two problems. Huge meeting. And there's a big delay in getting feedback to PR authors.
- pcwalton: Can we fold this into triage?
...
- wycats: Usually, the meeting is the first time some people on the core team is seeing the feature. That's because merging a feature is "cheap", since there's a feature flag. The meeting is where things get overall scrutiny.
- nrc: It seems like most features should have gone through the RFC process, so this seems like overkill
- wycats: Remember, once a feature is approved, it follows semver rules.
- nmatsakis: The purpose of Go/NoGo is to make a commitment. Need to make sure that the state of the code matches the RFC, adequate testing, etc. Not sure what the right meeting is, but we're going to need this kind of review.
- wycats: The smallest turnaround is 7 weeks.
- pcwalton: I think you need actual testing of features, not just checking against RFC. We need people to use nightlies (although not too many)
- nmatsakis: Ideally, you'd have some people using all three channels
- wycats: After 3 weeks into a cycle, you should be nervous about new features coming in.
- pcw: you can still kill it in beta
- wycats: Yes, but generally people dev in beta and ship on release
- pcwalton: So killing in beta is a failure mode
- wycats: I think all of this applies to libraries as well as language features
- dherman: At some point I'd like to discuss the beta user role -- what is that community? Which kinds of needs does each channel satisfy?
- wycats: You should run CI against beta. Nightly is pretty unstable, but beta should be semver compatible.
- dherman: Why not use stable?
- nmatsakis: You want to catch problems early. You should develop against beta and release against release
- wycats: For a few features, we can hit release with a kind of "opt-in" or feature gated status -- but then we have the burden of checking changes we want to make against the corpus of code out there. And that's just for 1.0 legacy.
- nmatsakis: Is this just macros?
- pcwalton: Struct variants?
- nmatsakis: I'd like to drop those
- wycats: Globs
- wycats: I think we should try to stabilize these before 1.0. What Ember did before 1.0 is just wrote a blog post about things that are likely to change, but feature gates seem better
- pcwalton: This does conflict with some horse trading with RFCs, where the RFC specifies a feature gate
- wycats: That forks the community
- pcwalton: Yes, that has to stop. Feature gates won't be usable in this way.
- wycats: One thing with Ember is, we do phased deprecation for bugs (misfeatures). It's reliant on people upgrading regularly. Should be done rarely, but it's ok.
- wycats: Important point that takes a while to internalize: do not rush to merge before a branch point. The next train is coming!
- pcwalton: The end-state for stability will be something like C++ where there's a little more tolerance for breakage due to the number of compilers and number of changing libraries. We're not going to have a huge amount of leeway, but more than the Java enterprise community
- wycats: If we need more stability, can always do a long-term release
- steveklabnik: Because releases are backwards-compatible, people will upgrade more often.
- dherman: What's the relationship between release cycles and bumping the major version?
- dherman: It's not clear how much flexibility for making breaking changes. In Java, they almost never make breaking changes. What's your vision for when we'd do major version changes, and what level of incompatibility is appropriate?
- dherman: What really matters here is de-facto incompatibility.
- wycats: It doesn't work with libraries, though.
- wycats: I prefer for breaking change releases and "marketing" releases to happen at the same time
- dherman: So we can wait to make a major release until we have sufficient marketing reasons or whatever, but it's not tied to the train schedule.
- nmatsakis: It seems too early to plan Rust 2.0; we're going to have to play it by ear.
- zwarich: Are there ever going to be Rust features behind a feature gate because of Servo, but where we're not ready to release more generally? That would force Servo to use nightly.
- nmatsakis: You could imagine allowing opt-in at Beta
- wycats: In Ember, you can actually customize the Go/No go yourself -- it's intentionally painful, but it allows you to opt-in
- dherman: Earlier, you were saying that there are very few messaging opportunities and you don't want to dilute the message. I want to understand this better.
- wycats: Any time someone builds from source, they're outside the messaging story. Maybe Servo can do that.
- wycats: The messaing hammers are: if something is in nightly, it may not be stable and may stop working. If it's in stable, it'll remain backwards-compatibly until the next major release.
- wycats: Beta is what you test your release-based stuff against to make sure you won't run into a problem. Also important to help make sure we don't break the world.
- dherman: If Servo wants to hand pick a small number of features it cares about, then there can just be a Servo fork of Rust with whatever features configured as opt-in, but otherwise uses the stable Rust
- dherman: I would hope that only experimental libraries would be used for Servo, eventually -- and then it could be done through Cargo. But that's still a ways off.
- steveklabnik: As a library author, being able to do testing against master/beta is really important. It lets upstream know about breakage quickly.
- steveklabnik: I know Servo is the most important Rust project right now, but having a privileged user can be really damaging to the community. (See Rails).
- dherman: It can also lead to bad design. It's easy to fall into scenario-solving.
- nmatsakis: I don't see a big problem. Sometimes Servo devs will use nightly, and if for some reason that doesn't work, Servo can build from source with separate feature flags.
- wycats: As Rust has gotten better about moving things into libraries, you're more able to experiment.

# Libraries

- wycats: In my view, libraries are not so different from language features. The goal should be that people can download a new version of Rust and have things keep working.
- wycats: If you take a library that was part of Rust and move it outside, you can evolve it separately from the six-week schedule. Likewise, separate semantic versioning for libraries than from Rust; people can lock their code into old library versions together.
- wycats: Assuming we do our job right with Cargo, there's not that big a difference between having it in Core and having it be installed through Cargo. You can still "bless" a library by putting it in the rust-lang organization.
- nmatsakis: It seems you need a finer-grained distinction. Many libraries will be partly stable.
- acrichto: We have a lot of libraries (like rustc) that we will not be able to stabilize, but they have to be part of the distro.
- wycats: I'd like there to be a strict requirement that code on rust 1.1 compiles for rust 1.2, so it seems very dangerous to expose things like rustc
- nmatsakis: There's no way we can stabilize libsyntax or librustc.
- brson: So if we said that literally no one can link to librustc, they won't be able to use regex, for example
- nmatsakis: You'd have to use a nightly build.
- dherman: So these libraries would show up in the Cargo repo with 0.X semver
- nmatsakis: Can we build a functional rustc that doesn't offer librustc?
- acrichto: Yes.
- brson: This is going to force some really hard decisions.
- wycats: It's either that, or we have really confusing messaging and can't keep semver promises
- acrichto: To be clear, it's just compile-time regex (syntax extensions) that's a problem.
- nmatsakis: If we ship regex with rust, we just need to ensure that it doesn't break.
- pcwalton: It's OK with me for compile-time regex to be nightly only.
- wycats: That will put pressure on stabilizing these features.
- pcwalton: Similar story with rust-postgres
- nmatsakis: So we can say that Rust 1.0 ships without compiler plugins.
- acrichto: If I depend on someone that depends on nightly, I have to use nightly?
- wycats: Yes -- strictly speaking, you depend on a feature flag, which will for some time tie you to nightly.
- erickt: For syntax extensions, is there a chance of getting quasi-quoting in a releasable state?
- acrichto: No; you'd have to change the return value to not be an AST object. We can talk about those specifics separately.
- acrichto: I'm also worried about the facade issues here -- I don't want people to link to liballoc.
- brson: Part of the point of the facade was to allow changes underneath.
- acrichto: So allow(experimental) is a feature gate?
- dherman: Is the problem here that we have libraries depended on by libstd (so they have to be distributed), but people need to be able to use them?
- wycats: We don't want them to be usable.f
- erickt: What if I want to use something unstable, but not opt-in to nightly?
- wycats: It's too subtle. I don't know how to message it.
- nmatsakis: It seems like we want to use stability attributes to deal with these questions -- as acrichto said, allow(experimental) is a feature gate and therefore only permitted in nightly.
- dherman: If we're serious about syntax extensions, we have to be honest about them -- all people using them are opting into nightly
- pcwalton: The biggest risk here is nightly becoming de facto Rust. I think we have to accept this risk.
- dherman: That seems more likely to be true in the beginning, but over time as we stabilize it will shift.
- nmatsakis: It seems like those who are trying to put things into production will try to avoid unstable features, and we need to provide a Rust that they can use.
- steveklabnik: There's a fundamental problem with libraries that is not tied to this versioning strategy.
- wycats: The versioning strategy forces us to be honest.
- dherman: I do think that messaging a distinction between stable and unstable is important; I've felt that for a long time and was impressed with Node doing this. I've only in the last year been seeing that they're not trying to identify a stable/useful core and build that out, and that's a problem.
- erickt: Another issues. Do we have any considerations for older releases? There may be people who don't upgrade for some period of time.
- wycats: We should promise to do security releases for some period. And we should see if there's demand for a long-term support release.
- erickt: What about bug fixes?
- wycats: Depends on their importance. Make that call on a case-by-case. But not for security issues -- those should have an explicit policy.
- dherman: Have to be careful about changes that are technically not backwards-compatible but are de facto OK. Probably best to be honest about this.

# Summarizing

- nmatsakis: It seems like the most basic principle here is that you can upgrade to 1.0 to 1.1 without any breakage. That means that syntax extensions can't be used in release builds. Not clear exactly what we want to call the stability attributes and how they interact with release process, but only "stable" items can be used in release.

# Deprecation

- huon: What about deprecation?
- acrichto: You can mark something deprecated and then remove in the next major release.
- wycats: Yes, unless the item is clearly a "bug"; then you can remove it after another cycle.
- nmatsakis: Or perhaps if it won't effect many people.
- wycats: But it's dangerous.
- pcwalton: We reserve the right to add warnings on point releases, which includes deprecation. So people can opt-in to that.
- wycats: The idea with deprecation is that you're learning about idiom changes.

# Major releases

- erickt: How often will we make major releases?
- brson: Once a decade :)
- wycats: No more often than once per year.
- nmatsakis: Clearly, we just have no idea yet.
- dherman: We're not setting any specific schedule for major releases here.
- nmatsakis: I hope that Rust 2.0 is a long way off.
- pcwalton: C++ has never has a 2.0
- wycats: You may end up with deprecated holdovers, and you may want a 2.0 to just drop this stuff away.
- wycats: Also issues like adding enum variants, which are technically breaking changes.

# automation

- wycats: we should test everything with feature flags on and everything with feature flags off

# phasing

- acrichto: how do you phase this in?
- wycats: at the same time as 1.0
- brson: shouldn't we have this running before 1.0?
- wycats: automation yes
- klabnick: we can do this whole process without having a stable branch before 1.0?
- wycats: I'm suggesting that the time to call it stable is 1.0
- nmatsakis: can't we just say that, starting from 0.12, we will be screening out things that are illegal in nightly builds etc?
- ...
- nmatsakis: It seems like allowing 0.12.0 to be incompatible with 1.0, that's allowed by semver
- wycats: My point is about messaging. You have to explain clearly what the plan is. You can't just drop it in for 0.12.
- nmatsakis: We're planning to do that.
- wycats: We just need to be clear about what we're doing, and set expectations. We probably don't need more than one or two test-run releases.
- acrichto: We need for people to actually use the test releases.
- acrichto: We can just change rustup to download the beta release rather than nightly
- wycats: We could call it 1.0 beta
- acrichto: We'd need confidence that we're six weeks out.
- nmatsakis: Do we need more than 6 weeks to get feedback about the subset of features they're using?
- nmatsakis: I still feel like we should just dry-run this process as soon as it's ready.
- wycats: The problem is "Release" implies compatibility. The win of being on that branch is compatibility
- nmatsakis: I feel like making a release 0.14 without fanfare wouldn't imply any compatibility
- dherman: This is about removing features, right? You can't just suddenly remove half the features in Rust.
- acrichto: We can't just expect that to come out OK. This could just push people into using nightly.
- wycats: That would mean that 1.0 is premature.
- brson: That's exactly the point. It'd be nice to have some advance notice.
- wycats: We have to meet the schedule. Then we'll have a process to evolve.
- steveklabnik: We'll get a huge influx of people wanting to use Stable.
- nmatsakis: If they end up using Nightly, so what? We'll release early and often.
- aturon: it seems like a compromise would be a 1.0 beta/RC, to get some feedback and still be able to message it clearly.
- wycats: That gives us a 12 week cycle, basically.
- nmatsakis: Ok. We should work on getting this infrastructure ready.

My presentation: http://cl.ly/3L3C0i3U3l2y

# Conclusions

- A program that compiles with Rust 1.0 stable also compiles with Rust 1.1 stable
- We don't want access to unstable/experimental libraries or feature gates in release builds
- Some low-level details about how to prevent access etc (for liballoc and friends)
- We need to consider candidates for libraries that are too experimental to include in libstd
- We will work on automation for the nitty-gritty details
