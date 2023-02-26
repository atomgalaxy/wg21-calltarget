---
title: "calltarget(unevaluated-call-expression)"
document: D2825R0
date: today
audience:
  - EWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
toc-depth: 2
---

# Introduction

This paper introduces a new compile-time expression into the language, for the
moment with the syntax `__builtin_calltarget(@_postfix-expression_@)`.

The expression is a compile-time constant with the value of the
pointer-to-function (PF) or pointer-to-member-function (PMF) that *would have
been called* if the `@_postfix-expression_@` had been evaluated.

In that, it's basically a compile-time resolver.

# Motivation and Prior Art

The language already has a number of sort-of overload resolution facilities:

- `static_cast`
- assignment to a variable of a given function pointer type
- function calls (implicit) - the only one that actually works

All of these are woefully unsuitable for type-erasure that library authors
(such as [@P2300R6]) would actually *like* to work with. Sure, we can always
indirect through a lambda:

```cpp
template <typename R, typename Args...>
struct my_erased_wrapper {
  using fptr = R(*)(Args_...);
  fptr erased = +[](my_erased_wrapper* self, auto&&... args_) -> R {
    return self->fptr(std::forward<decltype>(args_)...);
  };
};
```

This has several drawbacks:

- it introduces a whole new lambda scope
  - annoying for optimizers because extra inlining
  - annoying for debugging because of an extra stack frame
- a requirement to divine forward noexceptness by the user,
- a requirement that arguments are *convertible* (but not necessarily the same) as the ones the user specified in their infinite wisdom,
- a requirement that the return type is convertible (but not necessarily the same) as the one the user specified in their infinite wisdom,
- the forward-through-a-reference inhibits copy-elision of arguments
- inability to flatten subsetting type-erased wrappers (we can't divine an
  exact match in the presence of overloads) (think "subsetting vtables")

Oh, if only we had a facility to ask the compiler what function we'd be calling and then *just have the pointer to it*.

This is what this paper is trying to provide.

## Related Work

Of course, reflection would give us this. However, reflection
([@P2320R0],[@P1240R1],[@P2237R0],[@P2087R0],[@N4856]) is both nowhere close to
shipping, and is far wider in scope as another `decltype`-ish proposal that's
easily implementable today, and `std::execution` could use immediately.

It's also not clear how difficult it would be to do with reflection.

# Proposal

We propose a new (technically) non-overloadable operator (because `sizeof` is
one, and this behaves similarly):

(strawman syntax)

```cpp
auto fptr = __builtin_calltarget(@_expression_@);
```

Where the program is ill-formed if `@_expression_@` does not call a
user-defined function as its top-level AST-node (good luck to me wording this).

Examples:

```cpp
void g(long x) { return x+1; }
void f() {}                                                // #1
void f(int) {}                                             // #2
struct S {
  friend auto operator+(S, S) noexcept -> S { return {}; } // #3
  auto operator-(S) -> S { return {}; }                    // #4
  auto operator-(S, S) -> S { return {}; }                 // #5
  void f() {}                                              // #6
  void f(int) {}                                           // #7
  S() noexcept {}                                          // #8
  ~S() noexcept {}                                         // #9
  auto operator->(this auto&& self) const -> S*;           // #10
  auto operator[](this auto&& self, int i) -> int;         // #11
  static auto f(S) -> int;                                 // #12
  using fptr = void(*)(long);
  auto operator void(*)() const { return &g; }             // #13
};
S f(int, long) { return S{}; }                             // #14
struct U : S {}

void h() {
  S s;
  U u;
  __builtin_calltarget(f());                     // ok, &#1             (A)
  __builtin_calltarget(f(1));                    // ok, &#2             (B)
  __builtin_calltarget(f(std::declval<int>()));  // ok, &#2             (C)
  __builtin_calltarget(f(1s));                   // ok, &#2 (!)         (D)
  __builtin_calltarget(s + s);                   // ok, &#3             (E)
  __builtin_calltarget(-s);                      // ok, &#4             (F)
  __builtin_calltarget(-u);                      // ok, &#4 (!)         (G)
  __builtin_calltarget(s - s);                   // ok, &#5             (H)
  __builtin_calltarget(s.f());                   // ok, &#6             (I)
  __builtin_calltarget(u.f());                   // ok, &#6 (!)         (J)
  __builtin_calltarget(s.f(2));                  // ok, &#7             (K)
  __builtin_calltarget(s);                       // error, constructor  (L)
  __builtin_calltarget(s.S::~S());               // error, destructor   (M)
  __builtin_calltarget(s->f());                  // ok, &#6 (not &#10)  (N)
  __builtin_calltarget(s.S::operator->());       // ok, &#10            (O)
  __builtin_calltarget(s[1]);                    // ok, &#11            (P)
  __builtin_calltarget(S::f(S{}));               // ok, &#12            (Q)
  __builtin_calltarget(s.f(S{}));                // ok, &#12            (R)
  __builtin_calltarget(s(1l));                   // ok, &#13            (S)
  __builtin_calltarget(f(1, 2));                 // ok, &#14            (T)
  __builtin_calltarget(new (nullptr) S());       // error, not function (U)
  __builtin_calltarget(delete &s);               // error, not function (V)
  __builtin_calltarget(1 + 1);                   // error, built-in     (W)
  __builtin_calltarget([]{
       return __builtin_calltarget(f());
    }()());                                      // ok, &2              (X)
}
```

## Interesting cases in the above example

- resolving different members of a free-function overload set (A, B, C, D, T)
  - the (D) case is important - the `short` argument still resolves to the `int` overload!
- constructors and destructors (L, M, U, V) - see the **possible extensions** chapter.
- resolving different member of a member-function overload set (I, J, K, N, Q, R)
  - the (J) example is important - the call on `u` still resolves to a member function of `S`.
- resolving built-in non-functions (W): we could make this work in a future
  extension (see that chapter).
- resolving `operator->` (N and O). [expr.post.general]{.pnum} specifies that
  `@_postfix-expression_@`s group left-to-right, which means the top-most
  postfix-expression is the call to `f()`, and not the `->`. To get to
  `S::operator->`, we have to ask for it explicitly.
- surrogate function call (S) - again, the top-most call-expression is the
  function call to `g`, so that is what is returned.
- nested calls: (X) the top-level call is a call to a function-pointer to #2,
  so that is what is returned.


## Alternatives to syntax

We could wait for reflection in which case the syntax would be

`(^f()).TODO_ASK_SUTTON_OR_VANDEVOORDE()`.

However, I expect it would have rather more intricacies.

## Naming

### Grabbing a pattern

A suggestion of an esteemed former EWG chair is that we, as a committee, grab
the keyword-space prefix `__std_meta_*` and have all the functions with that
prefix have unevaluated arguments.

In that case, this proposal becomes

```cpp
__std_meta_calltarget(@_unevaluated-expression_@);
```

This is done so as to stop wringing our hands about function-like APIs that
have unevaluated operands, and going for less appropriate solutions for want of a function.
The naming itself signals that it's not a normal function. Its address also
can't be taken, it behaves as-if `consteval`.

It's ugly on purpose. That's by design. It's not meant to be pretty.

### Possible names

For all intents and purposes, this facility grammatically behaves in the same
way as `sizeof`, except that we should require the parentheses around the
operand.

We could call it something really long and unlikely to conflict, like
`expression_targetof`, or `calltargetof` or `decltargetof` or `targetexpr` or
`resolvetarget`.

# Possible Extensions

We could make compilers invent functions for the cases that currently aren't legal.

## Inventing contructor free-functions

For instance, constructor calls could "invent" a free function that is
expression-equivalent to calling placement new on the return object:

```cpp
// immovable aggregate
struct my_immovable_type {
  my_other_immovable_type x;
  some_immovable_type y = {};
};
my_immovable_type x(my_other_immovable_type{});
// note: prvalue parameters in type, since in-place construction through copy-elision is possible
std::same_as<my_immovable_type(*)(my_other_immovable_type, some_immovable_type)> 
  auto constructor_pointer = __builtin_calltarget(my_immovable_type(my_other_immovable_type{}));
auto y = constructor_pointer(my_other_immovable_type{});
// x and y have no difference in construction.
```

## Inventing destructor free-forms

Or, for destructors (this would make smart pointers slightly faster and easier to do):

```cpp
std::same_as<void(*)(S&)> auto
  dtor = [](S* x){return __builtin_calltarget(x->~S());}(nullptr);
S x;
dtor(x); // expression-equivalent to x.~S()
```

## Inventing pointers to built-in functions

We could invent pointers to functions that are otherwise built-in, like built-in operators:

```cpp
__builtin_calltarget(1+1); // &::operator+(int, int)
```

# Usecases

Broadly, anywhere where we want to type-erase a call-expression. Broad uses in
any type-erasure library, smart pointers, ABI-stable interfaces, compilation
barriers, task-queues, runtime lifts for double-dispatch, and the list goes on
and on and on and ...

## What does this give us that we don't have yet

Two things, mainly:

- no need for indirection through a lambda when resolving overlaod sets
- ... which allows us to use copy-elision to construct function arguments in
  type-erased interfaces, which is currently impossible without accurate type
  information from the user.

## That's not good enough to do all that work. What else?

Together with [@P2826R0], the two papers constitute the ability to implement
_expression-equivalent_.

[@P2826R0] proposes a way for a function signature to participate in overload
resolution and, if it wins, be replaced by some other function.

This facility is the key to *finding* that other function. The ability to
preserve prvalue-ness is crucial to implementing quite a lot of the standard
library customization points as mandated by the standard, without compiler
help.


---
references:
  - id: TaxonomyRevzin
    citatation-label: TaxonomyRevzin
    title: "What is unified function call syntax anyway?"
    author:
      - family: Revzin
        given: Barry
    URL: https://brevzin.github.io/c++/2019/04/13/ufcs-history/
  - id: D2823R0
    citation-label: D2823R0
    title: "Declaring hidden non-friend functions to be found by argument-dependent-lookup"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2823R0
  - id: D2822R0
    citation-label: D2822R0
    title: "Providing user control of associated entities of class types"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2822R0
  - id: P2826R0
    citation-label: P2826R0
    title: "Replacement functions"
    author:
      - family: Ažman
        given: Gašper
    URL: https://wg21.link/P2826R0
---
