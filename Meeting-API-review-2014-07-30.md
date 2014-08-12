# 2014-07-30

# Agenda

* Outstanding conventions issues
* Atomics stabilization
* Advance discussion:
  * Numeric traits
  * Collection traits
  * FromStr

# Attending

acrichto, aturon, brson, spernsteiner, huon

# Conventions issues

http://discuss.rust-lang.org/t/settling-some-key-naming-conventions/269/33

Surprisingly good consensus here. Main sticking point is unwrap -> assert.

Also some question about "move" variants: should we have a common
marker here? Suggestion: "_owned" (as opposed to borrowed). Use when
standard variant works with references.

> See: move_iter, Vec::push_all_move, move_from, move_val_init (deprecated)

iter_owned, push_all_owned, from_owned, val_init_owned
iter_val, push_all_val, from_val, val_init_val

# std::sync::atomics

http://static.rust-lang.org/doc/master/std/sync/atomics/index.html

## Ordering enum

Recommend: stable

General point: some of the operations only support a subset of these
orderings. In some cases we fail (fence fails on Relaxed), but in most
cases we interpret nonsensical orderings as SeqCst (the strongest ordering).

Recommend: fail everywhere on nonsensical orderings. Don't think it's
worth trying to slice up Ordering into different enums to catch this statically.

## Primitive structs: AtomicBool, AtomicInt, AtomicUint, AtomicPtr

e.g. http://static.rust-lang.org/doc/master/std/sync/atomics/struct.AtomicBool.html

Discuss:

- the Atomic prefix here. In general, wanting to move away from
`module::ModuleFoo` names, but for these primitive types the prefix is
probably desirable. Recommend: keep

- the names `store` and `load`. Basically the only usage of these
  names in libstd. Recommend: change to `set` and `get`

Overall recommendation: stable

No changes: mark as stable.

## AtomicOption

http://static.rust-lang.org/doc/master/std/sync/atomics/struct.AtomicOption.html

Seems fine; somewhat less "primitive" than the other types, not widely
used.

Recommendation: probably stable, unless people feel we might want to
remove.

DECISION: deprecate

Allows us to eventually move this into core

## Statics

Provides `INIT_ATOMIC_FOO` for static initializers. Since
representation is hidden, clients can't do this directly.

Recommendation: stable.

DECISION: makr unstable, modulo static conventions

## fence function

Provides memory fences.

Recommendation: stable.

DECISION: rename to atomic, mark module stable

# Numeric traits

http://static.rust-lang.org/doc/master/std/num/index.html

Stands out as one of the most abstract/fine-grained uses of traits in
libstd.

Try to simplify.

# Collection traits

Want to get alignment with the team here on the right way to go forward.

# FromStr

Used only in libnum's rational implementation.

huon has other use cases in mind, will send separately.
