# RFC triage 2014-06-26

# Attending

brson, luqman, cmr, spernsteiner, nrc, pcwalton, zwarich, huon, pnkfelix

# Action Items

- acrichto - close https://github.com/rust-lang/rfcs/pull/98
- brson - close https://github.com/rust-lang/rfcs/pull/45
- brson - close https://github.com/rust-lang/rfcs/pull/119
- huon - close https://github.com/rust-lang/rfcs/pull/120
- acrichto - close https://github.com/rust-lang/rfcs/pull/122

# RFC's

# https://github.com/rust-lang/rfcs/pull/98 - Add 'maybe initialised' pointers (pretty much out params, aiui) - gereeter

*    A new form of reference, `&uninit`, is added that is write-only and points to possibly uninitialized data.
*    Mixed reactions - overall positive, but agreement on low priority.
*    Recommend close as postponed - was this agreed last week?

# https://github.com/rust-lang/rfcs/pull/45 - Avoiding integer overflow - bmyers

*    Proposes range types and other complicated stuff, but discusses other options.
*     Lots of discussion on integer overflow in general, no agreement. Also  discussed to death on the mailing list several times, including  currently.
*     Recommend close - we might conceivably do something, but we won't do  this and we won't lose anything by closing this RFC (there's also a new  RFC with a different proposal derived from the most recent mailing list  thread).
     
- nrc: there's a newer version of this and i don't see us using the technique described here

# https://github.com/rust-lang/rfcs/pull/119 - add support to serialize::json for incrementally reading multiple JSON objects - XMPPwocky
    Apparently this is included in RFC 22, which has an implementation.
    Recommend close in deference to RFC 22.

- nrc: erickt working on serialization library that includes this behavior

# https://github.com/rust-lang/rfcs/pull/120 - Reintroduce 'do' keyword as sugar for nested match statements - bvssvni

*    Syntax for flattening match statements by 'chaining' arms of a match statement.
*     Feedback is mostly negative. Some positive feelings for including the  macro rather than first class syntax. Others want to wait for HKT and  have a function.
*    Recommend close as postponed. We probably want something like this, but not pre-1.0.
    
(agreement)

# https://github.com/rust-lang/rfcs/pull/122 - Syntax sugar for prefix-style type parameter lists - ben0x539

*     Sugary syntax for putting a group of type parameters and their bounds  before a group of functions. Motivation is our often unwieldly lists of  type parameters.
*    Not much feedback, but mostly positive. Generally for the motivation, rather than the solution.
*    Recommend close in deference to RFC 135 (where clauses) which solve the motivating problem here, along with other issues.

(agreement)
