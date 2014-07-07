# 2014-07-07

# Attending

brson, aturon, pcwalton, acrichto, huon, luqman, spernsteiner, nrc


# The `owned` module (defines `Box`)

http://static.rust-lang.org/doc/master/std/owned/

## Module

Module name “owned” reinforces misconceptions; primary about placement, not ownership.

Suggest: 
- module name should match the type name (box for Box)
- mark unstable

## The `Box` type

`Box` is somewhat evocative for heap placement.

OTOH, `box (GC) x` yields `Gc<T>`

and `box` generally means “place here”. So maybe `box (HEAP) x` yields `Heap<T>`?

Suggest:
- rename if we can reach consensus
- mark unstable modulo possible renaming

AFTER DISCUSSION:

- rename: owned -> boxed
- keep Box name, mark unstable pending addition of allocators later on
- mark HEAP as experimental; pending possible renaming, custom allocators

## Traits

I  would like these to live in `std::any`, but the `any` module is defined  in libcore while `Box` is defined in downstream liballoc.

### `AnyOwnExt` 
http://static.rust-lang.org/doc/master/std/owned/trait.AnyOwnExt.html

Suggest: 
- rename to BoxAny (modulo changes to name `Box`)
- rename method `move` to something more informative, e.g.:
    - `downcast`
    - `try_cast`
    - `into`, following the `as_ref` method defined on `&Any`
    - bad choice, but: `try_unwrap`

### `AnySendOwnExt`
http://static.rust-lang.org/doc/master/std/owned/trait.AnySendOwnExt.html

recently added by pcwalton in https://github.com/rust-lang/rust/commit/05e3248a7974f55b64f75a2483b37ff8c001a4ff

Appears unused. pcwalton, can you explain?

AFTER DISCUSSION:
 - try removing AnySendOwnExt, see if it works. This is working around a variance issue with mutable internals.
 - AnyOwnExt should be impl on DST - change signature of trait to take Box instead of impl on Box
 - Rename AnyOwnExt to BoxAny, mark unstable
 - Rename move -> downcast or possibly downcast_move, mark unstable

# The cell module

http://static.rust-lang.org/doc/master/std/cell/

Defines `Cell`, `RefCell`, `Ref` and `RefMut`

Overall, loose analogy between Cell and AtomicPtr, stronger analogy between RefCell and RWLock

Suggest: mark module as unstable

## Cell

Suggest: 
- mark methods as *stable*
- mark trait impls as *stable*
- mark type as unstable, likely renaming

## RefCell, Ref, RefMut

Suggest: 
- mark Ref and RefMut as *stable*
- mark RefCell::new as stable
- mark RefCell::borrow as stable
- mark RefCell::{borrow_mut, unwrap} as unstable, pending conventions finalization
- mark RefCell::{try_borrow, try_borrow_mut} as experimental; consider replacing with inspection methods

## clone_ref

This is split out as an external fn to avoid shadowing via Deref.

Suggest: 
- mark as unstable; after UFCS lands, move into extension trait

FROM DISCUSSION
- Mut for RefCell 
- Cell (PodCell, CopyCell) vs RWCell
- Slot rather than Cell

- Stabilize Cell inherent methods but not name
- Cell trait impls unstable
- Lint should warn when you use an unstable trait even with a stable impl
- Mark Ref RefMut unstable throughout -- until traits become stable
- Mark RefCell::new as stable, everything else unstable (commenting on the relevant conventions issues)

# Discussion

- aturon: not happy with 'owned' as name of module containing 'Box'. First of all, 'owned' is not the name of the type. Second, box is not saying anything special about ownership, it's about putting something on the heap.
- aturon: name of module should be same as type, also want to consider name 'Box'. it evokes heap allocation.
- aturon: you can use 'box' to create things that aren't boxes
- brson: disagree. Gc and Rc are types of boxes. Putting something in heap is 'boxing'.
- pcwalton: nervous about changing. it's working
- brson: agree
- acrichto: could rename 'HEAP' to 'BOX'
- huon: some confusion about 'box' operator being generic.
- brson: won't be confusing once we start using 'box' for 'Rc'.
- acrichto: i think once it works for other types it will be less confusing
- nrc: why not 'new'?
- acrichto: because `HashMap::new()`.
- nrc: this module seems to be about allocations. why not in liballoc?
- acrichto: it is
- aturon: agree that 'box' should be name of module
- brson: 'box' is a keyword
- aturon: 'owned' is bad
- pcwalton: 'boxed'
- acrichto: 'heap'

(agreement 'owned' is wrong)

- brson: liballoc already has a mod `heap`
- acrichto: pref for 'boxed'






