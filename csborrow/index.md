# A comparison of Rust's borrow checker to the one in C#

## Wait, C# has a borrow checker?

Behold: the classic example of rust's zero-cost memory safety...

```rs
// error[E0597]: `shortlived` does not live long enough

let longlived = 12;
let mut plonglived = &longlived;
{
	let shortlived = 13;
	plonglived = &shortlived;
}

*plonglived;
```

...ported to C#:

```cs
// error CS8374: Cannot ref-assign 'shortlived' to 'plonglived' because
// 'shortlived' has a narrower escape scope than 'plonglived'

var longlived = 12;
ref var plonglived = ref longlived;
{
	var shortlived = 13;
	plonglived = ref shortlived;
}

_ = plonglived;
```

OK, so C# doesn't share the Rust concept of "borrowing," so it wouldn't _technically_ be correct to call
this "borrow checking," but in _practice_ when people talk about "Rust's borrow checker" they're talking
about all of the static analysis Rust does to ensure memory safety, for which I think this qualifies.

When I first saw this feature in C# (and also `Span`s, `ref struct`s, and `stackalloc`), I was blown away:
where are all the angle brackets and apostrophes?  How is it possible that I can write efficient and
provably-safe code in C# without a degree in type theory?  In this document I hope to briefly summarize
my understanding of memory safety in C#, make direct comparisons between C# constructs and the corresponding
Rust ones, and maybe shed some light on what trade-offs C# made exactly to get this so user-friendly.

## A brief history of C# ref safety

[Since the beginning](https://ecma-international.org/wp-content/uploads/ECMA-334_1st_edition_december_2001.pdf)
 (2000-ish), C# has had the `ref` keyword for parameters passed into a function by
reference, but that was about all you could do with it.  If you wanted to do efficient things with
stack-allocated memory and indirection, you would generally use the "unsafe" portions of the language, or
call out to C++.  It wasn't until 2017 with the release of
[C# version 7](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-version-history) that
[we started to see](https://devblogs.microsoft.com/dotnet/whats-new-in-csharp-7-0/#ref-returns-and-locals)
this feature generalized into something more useful.  From there, C# added:

- `ref` local variables
- `ref` returns
- safe `stackalloc` initializers
- `readonly struct` and `ref struct`
- `in` parameters (and later `ref readonly` parameters)
- conditional `ref` expressions
- extensions to `stackalloc`
- `ref` fields

In the process of adding the above features, C# needed to define rules around `ref` usage that would
continue to ensure memory safety.  The language specification calls these rules "ref safe contexts" (see
[here](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/variables#972-ref-safe-contexts)
and
[here](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/structs#16412-safe-context-constraint)).
A "ref safe context" is likely better-known to Rust programmers as a
[lifetime](https://doc.rust-lang.org/rust-by-example/scope/lifetime.html), the
region of source text in which it is valid to access/use a reference.

## A comparison of ref safe contexts and lifetimes

Like in Rust, it is not possible in C# to explicitly declare the lifetime of a value.  _Unlike_ in rust,
it is also not possible in C# to assign a name to a lifetime using generic type parameters.
In both languages, the correct usage of a function must be knowable given only its declaration, and
not require analysis of its body.  In Rust, this means that lifetimes must appear in the function
declaration:

```rs
//                  V name the lifetime using generic type parameter
//                          V --------- V reference named lifetime in parameter and return types
fn return_reference<'a>(r: &'a i32) -> &'a i32 {
	r // <-- compiler makes sure we return what we claim to in the signature
}
```

C# has no syntax for this.  Nevertheless, the equivalent code still compiles:

```cs
// No lifetimes!
ref int ReturnReference(ref int r){
	return ref r;
}
```

The C# compiler simply assumes that the lifetime of the return is the same as the
lifetime of the parameter.  Rust can do the same...

```rs
// No lifetimes!
fn return_reference(r: &i32) -> &i32 {
	r
}
```

...and calls this feature [lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html).
In Rust, lifetime elision is optional, and the programmer can always explicitly state the lifetimes
of all references.  In C#, by contrast, the compiler must decide the lifetimes for all function
declarations.  For example, the following Rust function returns only one of its two arguments:

```rs
//                  V---V Two lifetimes for our two parameters
//                              V------------------------V Return lifetime is the same as that of
//                                                         the first parameter
//                                           V Second parameter lifetime is unused
fn return_reference<'a, 'b>(r: &'a i32, r2: &'b i32) -> &'a i32{
	r // <-- compiler would not allow us to return r2 here
}
```

Lifetime elision is not implemented for functions of this form, since there doesn't seem to be
a reasonable default to pick.  Nevertheless, C# must pick one:

```cs
// No lifetimes!  Wait.  What are they, though?
ref int ReturnReference(ref int r, ref int r2){
	return ref r;
}
```

Since it wouldn't make sense to "pick" either `r` or `r2` for the lifetime of the return, C# conservatively
assumes that the return could be _either_ of them.  Thus, both the arguments and the return are assumed to have
the _same_ lifetime, which the specification calls "caller-context."  The equivalent Rust function would
look like this:

```rs
//                  V only one lifetime, called "caller-context"
fn return_reference<'cc>(r: &'cc i32, r2: &'cc i32) -> &'cc i32{
	r // <-- compiler allows us to return either r or r2
}
```

This is less useful than the original Rust function.  For example, the following code will compile
successfully with the first declaration, but not with the second:

```rs
fn wrapper(r: &i32) -> &i32{
	let i = 12;
	return_reference(r, &i) // error[E0515]: cannot return value referencing local variable `i`
}
```

and in C#:

```cs
ref int Wrapper(ref int r){
	var i = 12;
	// Cannot use a result of 'Program.ReturnReference(ref int, ref int)'
	// in this context because it may expose variables referenced by
	// parameter 'r2' outside of their declaration scope
	return ref ReturnReference(ref r, ref i);
}
```

Here we see C#'s first trade-off: lifetimes are less explicit, but also less powerful.  The defaults
can also be unintuitive: say we wanted to write a method on a struct which returns a reference
to one of the struct's members.  In rust, this is simple:

```rs
struct Foo {
	member: i32
}

impl Foo {
	fn get_member<'a, 'b>(&'a self, unused: &'b i32) -> &'a i32 {
		&self.member // <-- compiler would not allow us to return `unused`
	}
}
```

In fact, this is so common that Rust doesn't require you to write the lifetimes explicitly, again thanks
to "lifetime elision:"

```rs
struct Foo {
	member: i32
}

impl Foo {
	fn get_member(&self, unused: &i32) -> &i32 {
		&self.member // <-- would still not be allowed to return `unused`
	}
}
```

The equivalent C# code doesn't compile, though:

```cs
struct Foo {
	int member;

	ref int GetMember(ref int unused){
		// error CS8170: Struct members cannot return 'this' or
		// other instance members by reference
		return ref this.member;
	}
}
```

This is because C#'s default is the _opposite_ of Rust's: a struct method that returns by reference is
allowed to return any reference _other_ than the implicit `this` reference.  The below compiles:

```cs
struct Foo {
	int member;

	ref int GetMember(ref int unused){
		return ref unused;
	}
}
```

C#'s history is rooted in OOP/Java-style programming, and letting methods return `this` references would
prevent you from writing code like the following:

```cs
ref int DoAThing(ref int p){
	// This reference is safe to return because it
	// could only be referencing p
	return ref new Foo().DoWhatever(ref p);
}
```

The lack of explicit lifetime annotations means C# has to choose which patterns are and
aren't allowed.

## The escape hatch: garbage collection

Let's say we want to write a function which returns a reference to an integer in a buffer if it finds it:

```cs
ref int Find(Span<int> haystack, int needle){
	for(var i = 0; i < haystack.Length; i++)
		if(haystack[i] == needle)
			return ref haystack[i];

	throw new Exception("Not Found");
}
```

Instead of throwing an exception, we've decided that this function should always return _something_, even
if it's not in the haystack.  We don't have anything else to return, though!  The following code won't compile:

```cs
ref int Find(Span<int> haystack, int needle){
	for(var i = 0; i < haystack.Length; i++)
		if(haystack[i] == needle)
			return ref haystack[i];
	var def = 0;
	return ref def; // Cannot return local 'def' by reference because it is not a ref local
}
```

Naturally, anything declared in `Find` will fall out of scope when `Find` returns, and so can't be
returned by reference.  C# has a superpower, though.  We can write the following:

```cs
ref int Find(Span<int> haystack, int needle){
	for(var i = 0; i < haystack.Length; i++)
		if(haystack[i] == needle)
			return ref haystack[i];
	var def = new int[1];
	return ref def[0];
}
```

The array referred to by `def` and the function's return value will live as long as there exist references
into it.  Rust has no equivalent to this.  We can return a reference to a constant...

```rs
fn find(haystack: &[i32], needle: i32) -> &i32 {
	for item in haystack {
		if *item == needle {
			return item;
		}
	}
	&0
}
```

...which disallows mutation, or we can return an enum (discriminated union)...

```rs
enum RefOrBox<'a, T> {
	Box(Box<T>),
	Ref(&'a mut T)
}

fn find(haystack: &mut [i32], needle: i32) -> RefOrBox<i32> {
	for item in haystack {
		if *item == needle {
			return RefOrBox::Ref(item);
		}
	}
	RefOrBox::Box(Box::new(0))
}
```

...which is not transparent to the caller of the function.  If we were willing to leak memory, then
we could write this:

```rs
fn find(haystack: &mut [i32], needle: i32) -> &mut i32 {
	for item in haystack {
		if *item == needle {
			return item;
		}
	}
	Box::leak(Box::new(0))
}
```

`Box::leak` returns a reference that is converted to `&'static i32`, where `'static` represents the program
lifespan (i.e. "forever").  The `'static` lifetime is the easiest to deal with because
[it can be converted to any other lifetime](https://doc.rust-lang.org/reference/subtyping.html).
In C#, the garbage collector exists
to [make references last forever](https://devblogs.microsoft.com/oldnewthing/20100809-00/?p=13203),
and thus every heap reference in C# can be considered equivalent to Rust's `'static`.

Ignoring the performance implications, this would seem to be an unambiguously good thing: `'static` can
go anywhere, therefore having all heap references be `'static` guarantees maximum flexibility.  Sadly, no:

```cs
Action CreateCounter(ref int i){
	return () => {
		// Cannot use ref, out, or in parameter 'i' inside
		// an anonymous method, lambda expression, query
		// expression, or local function
		i += 1; 
	};
}
```    

Because heap references can live forever, it is illegal to put `ref`s on the heap.  This means `ref`
can't be used in lambda captures or class/struct member variables.  Instead, the language provides
`ref struct`, a kind of struct that can contain `ref`s but is also required to never go on the heap.

So: garbage collection lets C# do things safely that are impossible to do in Rust, but splits
the language into the "garbage collected" and "stack allocated" worlds.  Rust has a stack/heap
distinction, but doesn't need the concept of a "stack-only" or "heap-only" type.

## Sharing XOR mutation

In rust, every reference is either:
- Shared: multiple references may exist and be read from, but none may be written to
- Exclusive: The reference may be read from or written to, but only one unborrowed reference is allowed to exist

This restriction is central to Rust's safety guarantees, but C# doesn't need it.  The reason is that Rust
has to account for the possibility that a reference may be invalidated _at any time_.  For example:

```rs
let mut v = vec![1, 2, 3];
let r = &v[0];
v.push(4);
// r could be invalid now
```

By contrast, in C# heap references are never invalid, while `ref`s can only be
invalidated by exiting from a block:

```cs
var v = new List<int>{1, 2, 3};
var sp = CollectionsMarshal.AsSpan(v);
ref var r = ref sp[0];
v.Add(4);
// r is definitely still valid (kinda)
```

To guarantee correctness, Rust's borrow checker therefore has to disallow any operation which
could invalidate a reference as long as that reference is in use.  All C# has to do is ensure
that a `ref` to stack-allocated data never leaves the scope it was created in.

## Why does nobody seem to be talking about this?

Maybe I'm bad at searching for these things, but these changes to C# seem to have gone completely under
the radar in places where you read about memory safety and performance.  Maybe it's just
because the language additions have happened super slowly, or maybe the C# and Rust communities
have so little overlap that there aren't enough people who program in both languages to notice the similarities.
Maybe there's something that makes C#'s `ref` subset so unusable that people just ignore it (I'll admit
to only having played around with it a bit, so far).

Here's my theory: C# already had an equivalent to all of these things in its "unsafe" subset,
so when introduced, `ref`-safety changes were typically framed as "bringing the performance of safe code
closer to that of unsafe code," which is arguably the opposite
perspective of Rust's "bringing the safety of high-performance code closer to that of high-level languages."
Perhaps that framing makes people miss that although the two languages are pushing in opposite directions,
they might actually be getting closer together.

## Postscript: C# 11

In the time since I've posted this article, the C# language designers have looked at my first two "C# can't
do this" examples, implemented a language-level fix for both of them, then travelled back in time to
November of last year to release the fix in C# 11.  They were careful
to [barely mention it](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11#ref-fields-and-ref-scoped-variables)
in the release notes to maximize the chance that I'd carelessly read past it while researching.
Thank you to those who have pointed out this devious prank to me.

The first multi-reference example where I explain why this...

```cs
ref int Wrapper(ref int r){
	var i = 12;
	// Cannot use a result of 'Program.ReturnReference(ref int, ref int)'
	// in this context because it may expose variables referenced by
	// parameter 'r2' outside of their declaration scope
	return ref ReturnReference(ref r, ref i);
}
```

...can't possibly work can be made to work by changing the definition of `ReturnReference` to this:

```cs
ref int ReturnReference(ref int r, scoped ref int r2){
	return ref r;
}
```

`scoped ref` is a new reference type which promises to never return the reference or assign it
to an output parameter.  In Rust terms, each C# function really has _two_ lifetimes associated with it,
"caller-context" and "function-member", with the latter used for `scoped ref` and the implicit `ref this`:

```rs
fn return_reference<'cc, 'fm>(r: &'cc i32, r2: &'fm i32) -> &'cc i32 {
	r
}
```

Just like we can "scope" a `ref` parameter, we can "unscope" the implicit `ref this`, which fixes
the other "C# can't do this" example from above:

```cs
struct Foo {
	int member;

	[UnscopedRef]
	ref int GetMember(ref int unused){
		return ref this.member;
	}
}
```

We can still find things that C# can't do, but we start to leave the realm of
simple examples, which is quite impressive:

```rs
struct TwoRefs<'a, 'b> {
	a: &'a i32,
	b: &'b i32
}

fn return_two_refs<'a, 'b>(a: &'a i32, b: &'b i32)
	-> TwoRefs<'a, 'b>
{
	TwoRefs {
		a: a,
		b: b
	}
}

// This works:
fn wrapper(r: &i32) -> &i32{
	let i = 0;
	&return_two_refs(r, &i).a
}
```


```cs
ref struct TwoRefs {
	public ref int a;
	public ref int b;
}

// There's no way to specify that the two refs we 
// return need different lifetimes, and we can't
// use the `scoped` keyword for either parameter.
TwoRefs ReturnTwoRefs(ref int a, ref int b){
	return new TwoRefs {
		a = ref a,
		b = ref b
	};
}

ref int Wrapper(ref int r){
	var i = 0;
	// Cannot use a member of result of 'A.ReturnTwoRefs(ref int, ref int)'
	// in this context because it may expose variables referenced by parameter 'b'
	// outside of their declaration scope
	return ref ReturnTwoRefs(ref r, ref i).a;
}
```

## Post-Postscript

Thank you to commenters who pointed out that the rust `find` example could just return a reference
to a constant, rather than needing to use rust's `Cow` type.  I've updated the example and rephrased
the limitation as a trade-off between mutability and transparency to the caller.


