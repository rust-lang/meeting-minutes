# RFC triage 2014-06-26

# Attending

pcwalton, brson, aturon, huon, cmr, pnkfelix, acrichto, zwarich

# Action Items

* (pnkfelix) close with expl https://github.com/rust-lang/rfcs/pull/94
* (pnkfelix) close https://github.com/rust-lang/rfcs/pull/102 with explanation
* (brson) close https://github.com/rust-lang/rfcs/pull/98
* (brson) close https://github.com/rust-lang/rfcs/pull/101
* (acrichto) close https://github.com/rust-lang/rfcs/pull/110

# RFC 94

https://github.com/rust-lang/rfcs/pull/94 - Disambiguate enum variant names - MagnetoHydroDynamics
     Allow multiple enums in the same module to have the same variant  names,  and have `EnumType::EnumVariant` to disambiguate between them.
     General support for the principle, but some problems with the  specifics  - introducing allowed name collisions. Some worries that the  RFC is  lacking detail.
    Recommend closing as postponed (we probably want something like this post-1.0, but it is not high priority now

- huon: There's a backwards compatibility issue here: modules and types can have the same name.
- all: file a P-backcompat-lang issue to ensure types/modules are in the same namespace so that this is an error.

# RFC 98

https://github.com/rust-lang/rfcs/pull/98 - Add 'maybe initialised' pointers (pretty much out params, I think) - gereeter
    A new form of reference, `&uninit`, is added that is write-only and points to possibly uninitialized data.
    Mixed reactions - overall positive, but agreement on low priority.
    Recommend close as postponed.

- acrichto: Niko has ideas for how today's static analysis can work w/o new pointer type. If we're adding ptr types don't need to do it right now.
- brson: Agree
- huon: should we reserve a keyword? Or just use language versions?
- brson: I don't want to.
- acrichto: A slippery slope; we'll end up reserving everything.
- zwarich: There are some problems with the examples in the RFC. It's using out-variables in pattern matches; not sure if that's safe for unwinding. You have to deal with the partially-initialized problem there, and we don't currently have infrastructure.
- zwarich: It's probably solvable, but would need to be spelled out.
[Edit (zwarich): elsewhere in the RFC there is discussion of this problem]

# RFC 101

https://github.com/rust-lang/rfcs/pull/101 - More flexible pattern matching for slices - krdln
    No comments. The RFC is a little bit vague, but seems like kind of a win. Backwards incompatible.
    Recommend close (low priority).

- pnkfelix: didn't we make this more restrictive recently?
- acrichto: subslice patterns always broken, jakub has fixes, not landed
- huon: looks backwards compatible to me (except putting the .. at the end of the match)
- huon: Could use a different syntax for the fixed-size case
- brson: This seems pretty niche to me.
- cmr: We can keep today's syntax and add this fixed-size thing later.
- pnkfelix: I just want to make sure we're not committed to the wrong choice now.
- cmr: I like the post-fix ..
- brson: I'm curious whether others think this is valuable. Is this something we actually want to do?
- cmr: If we're going to have vector matching, this seems like a clear win.
- pcwalton: I'd prefer to remove vector patterns.
- brson: I also think vector patterns are pretty niche; there are lots of collections in the world, why treat vectors specially? I'm not in favor of pursuing this.
- brson: Let's close, but leave a door open syntactically to pursue in the future.
- pnkfelix: But not postponed?
- brson: Yes, close the issue. We'll leave the possibility open, but not commit to doing it.

# RFC 102

https://github.com/rust-lang/rfcs/pull/102/files - Autoderef pointers as in C++ - Valoric
    Recommend close (not interested in fundamental changes at this stage)

- acrichto: Having just removed "cross-borrowing", we should live with that for a while before putting something like this in.
- pcwalton: I vote to close. It would be a mistake.
- pnkfelix: Agree.

# RFC 110

https://github.com/rust-lang/rfcs/pull/110 - allow the absolute path `::foo::bar` as an alias for `::bar` inside `foo` - kmc
    Not much in the way of feedback. acrichto objects. Seems small, a bit confusing, and unnecessary to me.
    Recommend close (low priority)

- pcwalton: I'd prefer not to do this.
- acrichto: The motivation has to do with macro hygiene. But the RFC itself is a little odd -- it's a one-off feature with some warts.
