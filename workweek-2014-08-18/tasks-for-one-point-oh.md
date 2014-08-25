# Task planning

## Existing PR

## In progress

## Assigned

## Unclear



- Big ticket items:
  - UFCS
    - Trait::fn // works today
    - <T as Trait>::fn // does not work today but is relatively straightforward
    - Type::fn --> G::N // does not work today and is more complicated
  - Associated types
  - Multidispatch
  - Unboxed closures
  - static/const
  - DST
  - Where clauses full implementation:
    - Generalized where clauses
    - Method specialization in traits
  - foo! vs @foo
  - Integer inference
  - markers/variance
  - I/O reform, libgreen
    - write RFC (in progress)
    - refactor current code, removing libgreen
    - add nonblocking and other APIs (not strictly 1.0)
  - slice sugar `[]`
  - globs and resolve
  - bounds checking
  - defaulted type parameters
    - settle on fallback, decide what to do
  - Allocators
    - Adapt RFC
    - Implement trait
  - Error handling
    - RFCs (in progress)
    - Renaming and adapting to conventions
    - Possible sugar
    - Traits and propagation, gated on multidispatch
  - Library stabilization
     - Iterators (needs method where clauses)
     - Collections (needs default type params, associated items with defaults)
     - I/O -- proceeds in parallel with I/O reform
  - make enum variants part of the type namespace
  - non-zeroing drop
    - lint 
    - trans changes
  - box desugaring
  - higher-ranked trait bounds
  - API evolution
    - produce RFC
  - release infrastructure
    - feature-gating feature-gating
    - setup release channels:
    - build rules for beta branch
    - organization of s3 to accommodate new artifacts
    - infrastructure for merging (adaptable from yehuda, nice to have)
  - cargo
    - registry
    - servo bugs
  - windows
  - trait reform, method resolution final details
    - big implications on library design
    - proper deref behavior
  - adapt High-5 to assign incoming PRs to somebody at random
    - write up and announce peer structure
  - blog post schedule
  - 1.0 events
    - Rust camp
    - developing tutorial material
    - training sessions internally within mozilla
  - documentation
    - user-facing: 
    - reference manual audit
  - guidelines
    - coding style guidelines for code that we control
  - opt-in builtin traits 
    - copy opt-in, apply builtin rules
  - privacy rules
  - destructor rules #8861
  - object lifetime stuff
  




























