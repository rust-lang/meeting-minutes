# CI

... some missed ...
- acrichto: Our default suggestion for the travis yaml is you build against beta and also against nightly, but nightly can fail. So we'd release a new beta -- that would trigger a build for everything in the world, and then we'd look at all the results
- acrichto: Because every package is already testing against nightly, we could check up on the statuses in advance (even periodically), and basically ensure that we only ship beta when nightly is green.
- pnkfelix: What about bugs in people's code?
- acrichto: We could filter those out
- nmatsakis: Seems pretty hard
- acrichto: We could just do cargo build, no running tests. Doesn't give us full coverage (since run semantics could change)
- acrichto: The idea is just to have signal at all times about breakage. We need access to everyone's Travis, but should be doable
- acrichto: The sooner we get these channels out, the better.
- steveklabnik: Note that rust-ci is a community project

# QA

- brson: What about a QA process? Right now we have CI and regression testing, but no QA
- acrichto: We definitely need it for installers -- smoke tests
- brson: We usually do that much already.
- nmatsakis: I thought QA was out of fashion?
- acrichto: Seems like regression tests should be enough for a compiler; what else are you going to check?
- dherman: Maybe we'll want more over time in the regression testing (including perf regression)
- brson: Here's an example. Right now platform coverage is limited to what we're paying bots for. When we release, we might check more platforms.
- dherman: Can probably do it by need
- brson: Android is a good example here, we test one configuration but there are millions
- nmatsakis: It does seem like there are tiers of tests. There's bors, there are other platform configs, there is the larger cargo community
- brson: Right now, every release is essentially untested
- brson: We could also test various tools, including running tools manually
- nmatsakis: Maybe we can write a script equivalent to someone downloading and starting their first project, exercise all the basics
- nmatsakis: Should probably just do that all the time...
