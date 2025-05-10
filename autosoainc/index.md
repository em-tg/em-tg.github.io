# Implementing Automatic Struct-of-Array Conversion in C

[An article](https://brevzin.github.io/c++/2025/05/02/soa/) was recently posted detailing how
one might go about using new "reflection" features in C++ to implement an automatic conversion
from something like this:

```c
struct vec_t {
	struct elem { // assume struct elem could be whatever, here
		int various, member, variables;
	} *buf;
	size_t len;
	size_t cap;
};
```

...to something like this:

```c
struct vec_t {
	struct vec_t_arrays {
		int *various, *member, *variables;
	} arrays;
	size_t len;
	size_t cap;
};
```

In words: a transformation from a
dynamic array of structs to a single struct where each member is its own
dynamic array.  Such a conversion is impossible to automatically perform in C without
code generation, because the language lacks the necessary reflection features.

Here's how you might try to do it anyway.

## Fun Preprocessor Metaprogramming Tricks

Most information about C structs evaporates at compile time, which makes structs
simultaneously a powerful low-level tool and a pain in the butt
for metaprogramming.  There is, however, a
workaround.  Instead of writing a struct definition the normal way, we can do this:

```c
#define FIELDS \
	x(int, a) \
	x(int, b) \
	x(void *, c)

struct my_struct {
	#define x(t, nm) t nm;
	FIELDS
	#undef x
};
```

`FIELDS` is what's called an "[X macro](https://en.wikipedia.org/wiki/X_macro),"
a macro that expands to a list of macro invocations, one per struct field.
By "hoisting" the struct fields to the preprocessor like this, information about them
can be made available to other places in the code.

If we want to automatically define one array per field, we can now do that:

```c
struct soa_vec {
	struct soa_vec_inner {
		#define x(t, nm) t *nm;
		FIELDS
		#undef x
	} inner;
	size_t len;
	size_t cap;
};
```

Dynamic array methods can then be defined like so:

```c
int soa_vec_append(struct soa_vec *v, struct my_struct *src){
	if(v->len == SIZE_MAX) return 0;
	if(!soa_vec_ensure_cap(v, v->len + 1)) return 0;

	#define x(t, nm) v->inner.nm[v->len] = src->nm;
	FIELDS
	#undef x
	v->len += 1;
	return 1;
}
```

The problem with this technique is that all of the functions and types now directly reference the one
`FIELDS` macro, when ideally we'd want this data structure to work for _any_ struct defined
using an x macro.  The usual way to implement generic data structures in C relies on wrapping all
of the types and functions in macros, but that won't work here since you can't put `#define` in
a macro definition.

There's an alternative, though:
[template headers](https://www.davidpriver.com/ctemplates.html).  If you squint a bit, the
`#include` directive is also a macro.  It copies tokens from one source file into another, with
preceding `#defines` acting similarly to macro parameters.  Such macros have decent syntax highlighting,
interact well with the debugger, and most importantly for our use case: are allowed to
contain preprocessor directives and further macro definitions.

Here's what the `soa_vec` implementation snippets look like after reworking:

```c
#ifndef FIELDS
#error "Need fields"
#endif

#ifndef id
#error "Need id"
#endif

struct id(soa_vec) {
	struct id(soa_vec_inner) {
		#define x(t, nm) t *nm;
		FIELDS
		#undef x
	} inner;
	size_t len;
	size_t cap;
};

struct id(soa_vec_entry) {
	#define x(t, nm) t nm;
	FIELDS
	#undef x
};

int id(soa_vec_append)(struct id(soa_vec) *v, struct id(soa_vec_entry) *src){
	if(v->len == SIZE_MAX) return 0;
	if(!id(ensure_cap)(v, v->len + 1)) return 0;

	#define x(t, nm) v->inner.nm[v->len] = src->nm;
	FIELDS
	#undef x
	v->len += 1;
	return 1;
}

#undef FIELDS
#undef id
```

It's mostly the same, except that we've wrapped all of the globally-visible
identifiers in an `id` macro so they can be given a different prefix
for each include.

Thus we have a generic, automatic struct-of-array transformation in C using the language's
built-in metaprogramming features.  I've posted a complete
example [here](https://github.com/em-tg/autosoa/blob/master/soa_vec.h).



