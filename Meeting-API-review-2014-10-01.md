# 2014-10-01

# std::char

- http://doc.rust-lang.org/std/char/
- https://github.com/rust-lang/rust/blob/master/src/libcore/char.rs
- http://docs.oracle.com/javase/6/docs/api/java/lang/Character.html

Lots of free functions, some duplicate trait functions.

Non-unicode stuff comes from core::char, unicode from unicode::u_char

recommend: module name stable

- MAX
  - recommend: stable

# Char trait

recommend: trait is experimental because trait composition and organization may change

- fn is_digit_radix(&self, radix: uint) -> bool;
  - fails if radix is oob. returns none if num is oob
  - documentation_refers to `is_digit`, which is on the CharUnicode trait and handles
    unicode. this seems kind of surprising
  - recommend: rename is_digit, unstable pending error conventions
- fn to_digit(&self, radix: uint) -> Option<uint>;
  - fails if radix is oob. returns none if num is oob
  - handles 0-36.
  - recommend: unstable pending error conventions
- fn from_digit(num: uint, radix: uint) -> Option<Self>;
  - fails if radix is oob. returns none if num is oob
  - recommend: unstable pending error conventions
- fn escape_unicode(&self, f: |char|);
- fn escape_default(&self, f: |char|);
  - both of these have *strange* signatures, calling back a function with
    a series of `char`s. If we really must not allocate I would expect
    a stack buffer cast to &str at least. in-tree uses push onto a string.
    Could also do like encode_utf8 and push into a user-provided buffer.
  - recommend: unstable pending trait organization, change to return iterators
- fn len_utf8_bytes(&self) -> uint;
  - naming is a bit awkward
  - missing len_utf16_bytes
  - recommend: unstable ", rename len_utf8, add len_utf16
- fn encode_utf8(&self, dst: &mut [u8]) -> Option<uint>;
  - recommend: unstable pending iterator conventions, rename utf8, return iterator
- fn encode_utf16(&self, dst: &mut [u16]) -> Option<uint>;
  - recommend: unstable pending iterator conventions, rename utf16, return iterator

observations:
* All methods but encode_utf8/encode_utf16 are also free functions.
* Methods take `&self` while functions take `char`. Should methods take `self`?
* maybe it's not worth splitting Char/CharUnicode. not much here
* is_digit_*radix* vs to_digit, from_digit. all take radix, but one has it in
  the name.
* is_digit is in CharUnicode
* missing encode_utf32
* should the constructors be free functions or not?

missing method: fn from_u32(i: u32) -> Option<char>
  - recommend: add to Char, stable, deprecate this one

recommend: deprecate free functions (except from_*?)

# UnicodeChar

trait name: experim
more duplicated free functions

- fn is_alphabetic(&self) -> bool;
  - recommend: stable
- fn is_XID_start(&self) -> bool;
  - Java Character docs don't contain the word XID
  - recommend: experimental
- fn is_XID_continue(&self) -> bool;
  - Java Character docs don't contain the word XID
  - recommend: experimental
- fn is_lowercase(&self) -> bool;
  - recommend: stable
- fn is_uppercase(&self) -> bool;
  - recommend: stable
- fn is_whitespace(&self) -> bool;
  - recommend: stable
- fn is_alphanumeric(&self) -> bool;
  - recommend: stable
- fn is_control(&self) -> bool;
  - recommend: stable
- fn is_digit(&self) -> bool;
  - recommend: stable, rename is_numeric
- fn to_lowercase(&self) -> char;
  - recommend: stable
- fn to_uppercase(&self) -> char;
  - recommend: stable
- fn width(&self, is_cjk: bool) -> Option<uint>;
  - wierd unicode case
  - recommend: experimental. ask somebody

recommend: deprecate free functions

talk to simon about these names

# Free functions

- pub fn canonical_combining_class(c: char) -> u8
  - undocumented
  - recommend: experimental

- pub fn decompose_canonical(c: char, i: |char|)
- pub fn decompose_compatible(c: char, i: |char|)
  - more wierd function sigs
  - recommend: experimental

- pub fn compose(a: char, b: char) -> Option<char>
  - undocumented
  - recommend: experimental

observations:
* seems like maybe libicu should be doing this stuff




```
trait Foo { fn from_bar(b: &Bar) -> Self; }

// today
let b: SomeType = Foo::from_bar(&b);

// tomorrow
let b = SomeType::from_bar(&b);
```
