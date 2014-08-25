# 2014-08-20 using `@`

pcwalton: The tricky case is `@` identifier open paren. You parse it as a token tree first. If the first thing afterwards is an item, then it's an attribute, otherwise it's a macro.

@foo = bar
@foo(<token tree>) *keyword for item* --> attribute for item

```
fn main() {
    @println("Hello, {}", "World!")
}
```

pcwalton: Does ruby get trolled for the @ thing?
dherman: Julia doesn't I think
steve: In ruby you don't use @ b/c you normally use attr_accessor


```
@macro_rules foo( () => () )

@foo()
fn main() {}
```

acrichto: Isn't this ambiguous?
pcwalton: Should item macros be terminated with a semicolon? I'm worried about the C++ wart of class.
pnkfelix: We could require that item macros use curly braces.

...

pcwalton: I don't like `@foo = "bar"`.

...

niko: You could require parens for `key = value`. Perhaps just a matching delimiter. `@[foo = bar]`

```
@[foo, bar(baz)]
@foo
@[foo = "bar", baz = 3, foo1(foo2, bar = "34")]
```

Examples:

```
@inline
@[inline]
#[foo = bar]
@foo = bar
@[foo = bar]

@foo("bar") vs @[foo = "bar"]

@deriving(Eq)

@deriving(Decoder(T=SomeType))

@deriving(Decoder<T: SomeType>, Eq)
```


Today:

```
attribute : '#' '!' ? '[' meta_item ']' ;
meta_item : ident [ '=' literal
                  | '(' meta_seq ')' ] ? ;
meta_seq : meta_item [ ',' meta_seq ] ? ;
```

Tomorrow:

```
attribute  : '@' '!' ? meta_call
meta_call  : ident ( '(' meta_seq ')' ) ?
meta_seq   : meta_arg [ ',' meta_arg ] +
meta_arg   : meta_call
           | ident '=' string_literal
```

acrichto doesn't like: @foo("bar")
acrichto doesn't like: @foo(bar = baz(bar))

aturon: what about @deprecated("use bar instead") <- want this
  have to do: @deprecated(text = "use bar instead") ?
  (Today: #[deprecated = "use bar instead"])

@foo() vs @foo

@deriving(Decodable(T="Eq<foo>"))



```
@foo
mod foo {
    @!bar
}
```





