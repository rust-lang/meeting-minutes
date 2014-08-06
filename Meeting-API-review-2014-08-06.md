# 2014-08-06

# core::slice

http://doc.rust-lang.org/core/slice
https://github.com/rust-lang/rust/blob/master/src/libcore/slice.rs

* DST will change many of the signatures, but in slight ways. Probably not a lot of 'stable' here.
* All traits are named incorrectly ('Vector')
* General naming pattern of 'Immutable[something]Vector' and 'Mutable[something]Vector'
* All traits in the prelude
* 2 free functions and 2 tiny submodules
* Iterators have names, which we may not want, but may have to live with
* The way methods take self may change, but it shouldn't affect callers
* Indexes could change types if we switch the default integer types
  - but that's a 'boiling-the-oceans' change anyway
* Ok to mix unsafe methods and safe methods in same traits?
* ImmutableVector and MutableVector will end up combined

## Recommendations

* module name stable
* Rename all traits to `Slice`.

Resolutions: do above

## ImmutableVector

* split functions exclude the pivot element. Is that normal?
* Are splitn, rsplitn useful?
* Why do iterator methods take by-val self? Post-DST should be &self?
  - Can we change these now? Perhaps it's this way for perf?
* head/tail/init/last - are these the conventions?
  - they are haskellisms, not exacly clear, particularly 'init'
* tailn/initn - useful enough? easy to just use slice
* unsafe_ref - is this the right name?
  - compare to safe 'get'
* bsearch - rename binary_search?
  - should return a custom enum that tells where to insert on failure
* get but no set?
* shift_ref, pop_ref
  - what these are doing looks kind of scary
  - they mutate the slice, in a trait called *Immutable*Vector
  - can't be supported on Vec because the vec would leak the popped elt
  - limited use case, used in ringbuf iters, presumably for perf
  - shift_ref has the wrong name. should be 'pop_front_ref'
* split functions are totally different from those on Str
  - check for consistency with string slice

### Recommendations

* Rename ImmutableSlice - unstable
* split_at - unstable
* get - unstable (index type may change?)
* head/tail/init/last - experimental? (maybe rename 'init'?)
* tailn/initn - experimintal?
* unsafe_ref - rename 'unsafe_get', unstable
* as_ptr - unstable
* shift_ref, pop_ref - deprecate or delete
* slice, slice_from, slice_to - unstable
* iter - unstable (iterator name may change, &self)
* split - unstable (iterator name may change, &self) 
* splitn, rsplitn - unstable
* windows, chunks - unstable (iterator ")
* bsearch - rename 'binary_search', unstable (") return location on failure (custom enum)
* impl for slices unstable (slice type will change with DST)
* deprecate tailn/initn (same as slice_from/to)

Resolutions: do above

## MutableVector

* 'mut' placement is all over. should be last everywhere
 - get_mut, as_slice_mut, slice_mut, slice_from_mut, etc.
* defines as_mut_slice, but as_slice gets its own trait (Vector)
* *most* functions take by-val self, whereas some of ImmutableVectors took &self. Why?
  - can we fix the signatures now and suffer the indirection?
  - mut_shift_ref/mut_pop_ref take &mut self
* mut_shift_ref/mut_pop_ref have same problems as ImmutableVector
* unsafe_mut_ref -> unsafe_get_mut?
* unsafe_get_mut vs unsafe_set. strange incongruence. unsafe_get is on a diff trait
* init_elem - strange name compare to unsafe_set.
  - compare to mem::uninitialized. mem::forget
* copy_memory - potentially very misleading name.
  - implemented as copy_nonoverlapping_memory
  - is it possible to pass overlapping memory to this function?
  - if not, we should not mention it in the docs
* missing mutable equivs for
  - splitn, rsplitn, windows
  - head/tail/init/initn/tailn. we've got 'mut_last'!
  - as_ptr_mut
  - binary_search_mut

### Recommendations

* all self-passing may change to by-ref
* Rename MutableVector to MutableSlice. unstable (region param may go away)
* Fix mut placement
* get_mut - unstable
* as_mut_slice - rename, experimental (may move to a diff trait)
* mut_slice, mut_slice_from, mut_slice_to - rename. unstable
* mut_last - rename. unstable
* swap - unstable
* mut_split_at - unstable
* reverse - unstable
* unsafe_mut_ref -> unsafe_get_mut. unstable
* unsafe_set -> unstable
* init_elem -> rename unsafe_set_uninitialized. unstable
* as_mut_ptr -> rename. unstable
* mut_iter, mut_chunks - rename. unstable
* copy_memory - fix docs or rename to copy_nonoverlapping_memory. unstable
* mut_pop_ref/mut_shift_ref - deprecate or delete
* impl for slices unstable (slice type will change with DST)
* add mutable head/tail/init/initn/tailn fns. unstable

Resolutions

## Vector

* Only contains `as_slice`. Why not on ImmutableVector?
  - that something *can be converted to a slice* doesn't mean something *behaves like a slice*
* *Option implements Vector*
* str::Str also declares `as_slice` with &str retval
  - String implements Str::as_slice
* would be better to have distinct names for byte slices and string slices
* MutableVector defines `as_mut_slice` along with lots of other stuff!

## ImmutableOrdVector

* bsearch_elem - similar fate to bsearch
  - _elem convention still good?

## ImmutableEqVector

* Requires PartialEq - is that the right bound?
- ImmutablePartialEqVector? yes
* position_elem, rposition_elem - unstable
* contains - unstable
* starts_with, ends_with - unstable

## MutableCloneableVector

* diff nameing. ImmutableEqVector vs MutableCloneableVector
* Rename MutableCloneSlice
* copy_from
 - compare to copy_memory, others?

Resolutions:

* MutableCloneSlice
* clone_from_slice experimental

## `raw` module and `bytes` module

* not big enough to deserve modules. can we reorg?

## Other

* impl Default for &[T] has an odd implementation
  - it's returning a pointer to a new empty slice
  - i'm surprised that even works
* ref_slice and mut_ref_slice
  - do we like this trick?
  - do we like them as free functions?
  - in this module?
  
Resolutions:

* call ref_slice, etc. unstable fix name
