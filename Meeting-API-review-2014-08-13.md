# 2014-08-13

# Attending

brson, aturon, nmatsakis, spernsteiner, huon, pcwalton

# Agenda

- core::option
- core::result

# core::option

## Types

Recommendations:
  Option -> mark as stable
  Item   -> mark as stable

General note: public iterator types are a wart we will have to live
with, even though later on, if we do "impl Trait", they may no longer
be needed.

- aturon: let's mark Item as unstable because questions about module org with assoc. items
- nmatsakis: would like Item to be called Items

## Inherent methods

Broken down by recomendation:

### Stable:

```rust
impl<T> Option<T> {
  fn is_some(&self) -> bool
  fn is_none(&self) -> bool\
  fn take(&mut self) -> Option<T>
  fn as_ref(&'r self) -> Option<&'r T>
  fn as_slice(&'a self) -> &'a [T]
  fn and<U>(self, optb: Option<U>) -> Option<U>
  fn or(self, optb: Option<T>) -> Option<T>
}
```

### Unstable due unboxed closures:

```rust
  fn map<U>(self, f: |T| -> U) -> Option<U>
  fn map_or<U>(self, def: U, f: |T| -> U) -> U
  fn and_then<U>(self, f: |T| -> Option<U>) -> Option<U>
  fn or_else(self, f: || -> Option<T>) -> Option<T>
```

### Unstable due to pending conventions:

```rust
impl<T> Option<T> {
  fn as_mut(&'r mut self) -> Option<&'r mut T>
  fn as_mut_slice(&'r mut self) -> &'r mut [T]
  fn expect(self, msg: &str) -> T
  fn unwrap(self) -> T
  fn unwrap_or(self, def: T) -> T
  fn unwrap_or_else(self, f: || -> T) -> T
  fn iter(&'r self) -> Item<&'r T>
  fn mut_iter(&'r mut self) -> Item<&'r mut T>
  fn move_iter(self) -> Item<T>
}

impl<T: Default> Option<T> {
  fn unwrap_or_default(self) -> T
}
```

Note also: thread about limiting to move_iter, given as_ref/as_mut:
  http://discuss.rust-lang.org/t/a-single-iteration-method-for-option-iter/251

Personally, I think we should keep all three for consistency with
other containers, and for later Iterable trait

### Deprecated due to lack of use:

```rust
impl<T> Option<T> {
  fn filtered(self, f: |&T| -> bool) -> Option<T>
  fn while_some(self, f: |T| -> Option<T>)
  fn mutate(&mut self, f: |T| -> T) -> bool
  fn mutate_or_set(&mut self, def: T, f: |T| -> T) -> bool
}
```

### Deprecated due to redundancy:

```rust
impl<T> Option<T> {
  // use .take().unwrap() instead
  fn take_unwrap(&mut self) -> T

  // use .as_ref().unwrap() instead
  fn get_ref(&'a self) -> &'a T

  // use .as_mut().unwrap() instead
  fn get_mut_ref(&'a mut self) -> &'a mut T
}
```

### Missing items:

Some of the methods that have `or` variants lack `or_else` variants.

Recommended: add the following at unstable level:

```rust
impl<T> Option<T> {
  fn map_or_else<U>(self, def:|| -> U, f: |T| -> U) -> U
}
```

## The collect function

This short-circuits the collecting process when None is encountered:

```rust
pub fn collect<T, Iter: Iterator<Option<T>>, V: FromIterator<T>>(iter: Iter) -> Option<V>
```

Recommendation: move this to an impl for FromIterator, mark as unstable pending FromIterator stabilization:

```rust
impl<T, V: FromIterator<T>> FromIterator<Option<T>> for Option<V> {
  fn from_iter<I: Iterator<Option<T>>>(iterator: I) -> Option<V> {
    ...
  }
}
```

NOTE: ping erickt about why this is a free function

## The module itself

Recommend: mark as stable; placement and name will not change

================================================================

# core::result

## The Result type

Recommend: stable

## Inherent methods

Broken down by recomendation:

### Stable:

```rust
impl<T, E> Result<T, E> {
  fn is_ok(&self) -> bool
  fn is_err(&self) -> bool
  fn ok(self) -> Option<T>
  fn err(self) -> Option<E>
  fn as_ref(&'r self) -> Result<&'r T, &'r E>
  fn and<U>(self, res: Result<U, E>) -> Result<U, E>
  fn or(self, res: Result<T, E>) -> Result<T, E>
}
```

### Unstable for unboxed closures

```rust
  fn map<U>(self, op: |T| -> U) -> Result<U, E>
  fn map_err<F>(self, op: |E| -> F) -> Result<T, F>
  fn and_then<U>(self, op: |T| -> Result<U, E>) -> Result<U, E>
  fn or_else<F>(self, op: |E| -> Result<T, F>) -> Result<T, F>
```

### Unstable due to pending conventions:

```rust
impl<T, E> Result<T, E> {
  fn as_mut(&'r mut self) -> Result<&'r mut T, &'r mut E>
  fn unwrap_or(self, optb: T) -> T
  fn unwrap_or_else(self, op: |E| -> T) -> T
}

impl<T, E: Show> Result<T, E> {
  fn unwrap(self) -> T
}

impl<T: Show, E> Result<T, E> {
  fn unwrap_err(self) -> E
}
```

### Missing items:

Some of these can be gotten at by using:

```rust
  .as_ref().ok()  // these are painful...
  .as_mut().ok()
  .ok()           // for move_iter
```

```rust
impl<T, E> Result<T, E> {
  fn as_slice(&'a self) -> &'a [T]
  fn as_mut_slice(&'r mut self) -> &'r mut [T]

  fn iter(&'r self) -> Item<&'r T>
  fn mut_iter(&'r mut self) -> Item<&'r mut T>
  fn move_iter(self) -> Item<T>
}
```

Recommend: unstable due to conventions issues

## Bare fns: collect, fold, fold_

Recommend: do same for collect as with Option

The fold functions:

```rust
pub fn fold<T, V, E, Iter: Iterator<Result<T, E>>>(iterator: Iter, init: V, f: |V, T| -> V) -> Result<V, E>
pub fn fold_<T, E, Iter: Iterator<Result<T, E>>>(iterator: Iter) -> Result<(), E>
```

The fold_ is very rarely used; recommend deprecate

The fold function should perhaps be a method on the Iterator trait,
which will be possible with where clauses.

Recommend: experimental for now, with note that likely to move into method later.

(We will probably want to add other such methods)

## The module itself

Recommend: stable
