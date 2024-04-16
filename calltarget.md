---
title: "declcall(unevaluated-postfix-expression)"
document: P2825R1
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
moment with the syntax `declcall(@_expression_@)`.

The `declcall` expression is a constant expression of the type pointer-to-function
(PF) or pointer-to-member-function (PMF). Its value is the pointer to the
function that would have been invoked if the _expression_ were evaluated.
The _expression_ itself is an unevaluated operand.

In effect, `declcall` is a hook into the overload resolution machinery.

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

### Reflection

The reflection proposal does not include anything like this. It knows how to
reflect on constants, but a general-purpose feature like this is beyond its
reach. Source: hallway discussion with Daveed Vandevoorde.

We probably need to do the specification work of this paper to understand the
corner cases of even trying to do this with reflection.

Reflection ([@P2320R0],[@P1240R1],[@P2237R0],[@P2087R0],[@N4856])
might miss C++26, and is far wider in scope as another `decltype`-ish proposal
that's easily implementable today, and `std::execution` could use immediately.

Regardless of how we chose to provide this facility, it is dearly needed, and
should be provided by the standard library or a built-in.

See the [Alternatives to Syntax](#alternatives-to-syntax) chapter for details.


### Library fundamentals TS v3

The [Library Fundamentals TS version 3](https://cplusplus.github.io/fundamentals-ts/v3.html#meta.trans.other)
defines `invocation_type<F(Args...)>` and `raw_invocation_type<F(Args...)>` with
the hope of getting the function pointer type of a given call expression. 

However, this is not good enough to actually be able to resolve that call in
all cases.

Observe:

```cpp
struct S {
  static void f(S) {} // #1
  void f(this S) {}   // #2
};
void h() {
  static_cast<void(*)(S)>(S::f) // error, ambiguous
  S{}.f(S{}); // calls #1
  S{}.f(); // calls #2
  // no ambiguity for declcall
  declcall(S{}.f(S{})); // &#1
  declcall(S{}.f());    // &#2
}
```

A library solution can't give us this, no matter how much we try, unless we can
reflect on unevaluated operands (which Reflection does).

# Proposal

We propose a new (technically) non-overloadable operator (because `sizeof` is
one, and this behaves similarly):

```cpp
declcall(@_expression_@);
```

Example:

```cpp
int f(int);  // 1
int f(long); // 2
constexpr auto fptr_to_1 = declcall(f(2));
constexpr auto fptr_to_2 = declcall(f(2l));
```

The program is ill-formed if the named `@_postfix-expression_@` is not a call
to an addressable function (such as a constructor, destructor, built-in, etc.).

```cpp
struct S {};
declcall(S()); // Error, constructors are not addressable
declcall(__builtin_unreachable()); // Error, not addressable
```

The expression is not a constant expression if the `@_expression_@`
does not resolve for unevaluated operands. Examples of this function pointer
values and surrogate functions.

```cpp
int f(int);
using fptr_t = int (*)(int);
constexpr fptr_t fptr = declcall(f(2));
declcall(fptr(2)); // Error, fptr_to_1 is a pointer value
struct T {
    constexpr operator fptr_t() const { return fptr; }
};
declcall(T{}(2)); // Error, T{} would need to be evaluated
```

If the `declcall(@_expression_@)` is evaluated and not a constant
expression, the program is ill-formed (but SFINAE-friendly).

However, if it is unevaluated, it's not an error.

Example:

```cpp
int f(int);
using fptr_t = int (*)(int);
constexpr fptr_t fptr = declcall(f(2));
static_cast<declcall(fptr(2))>(fptr); // OK, fptr, though redundant
struct T {
    constexpr operator fptr_t() const { return fptr; }
};
static_cast<declcall(T{}(2))>(T{}); // OK, fptr
```

This pattern covers all cases that need evaluated base operands, while making
it explicit that the operand is evaluated due to the `static_cast`.

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
  auto operator fptr const { return &g; }                  // #13
  auto operator<=>(S const&) = default;                    // #14
};
S f(int, long) { return S{}; }                             // #15
struct U : S {}

void h() {
  S s;
  U u;
  declcall(f());                     // ok, &#1             (A)
  declcall(f(1));                    // ok, &#2             (B)
  declcall(f(std::declval<int>()));  // ok, &#2             (C)
  declcall(f(1s));                   // ok, &#2 (!)         (D)
  declcall(s + s);                   // ok, &#3             (E)
  declcall(-s);                      // ok, &#4             (F)
  declcall(-u);                      // ok, &#4 (!)         (G)
  declcall(s - s);                   // ok, &#5             (H)
  declcall(s.f());                   // ok, &#6             (I)
  declcall(u.f());                   // ok, &#6 (!)         (J)
  declcall(s.f(2));                  // ok, &#7             (K)
  declcall(s);                       // error, constructor  (L)
  declcall(s.S::~S());               // error, destructor   (M)
  declcall(s->f());                  // ok, &#6 (not &#10)  (N)
  declcall(s.S::operator->());       // ok, &#10            (O)
  declcall(s[1]);                    // ok, &#11            (P)
  declcall(S::f(S{}));               // ok, &#12            (Q)
  declcall(s.f(S{}));                // ok, &#12            (R)
  declcall(s(1l));                   // error, #13          (S)
  static_cast<declcall(s(1l)>(s));   // ok, &13             (S)
  declcall(f(1, 2));                 // ok, &#15            (T)
  declcall(new (nullptr) S());       // error, not function (U)
  declcall(delete &s);               // error, not function (V)
  declcall(1 + 1);                   // error, built-in     (W)
  declcall([]{
       return declcall(f());
    }()());                          // error (unevaluated) (X)
  declcall(S{} < S{});               // error, synthesized  (Y)
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
  function call to `g`, so the type of `g` is returned, but it's not a constant expression.
  We can get it by evaluating the operand with `static_cast`.
- nested calls: (X) the top-level call is a call to a function-pointer to #2,
  so that is what is returned, but since obtaining the value of the function pointer
  requires evaluation, this is ill-formed. Getting the type is fine.
- Synthesized operators (Y) - these are not functions that we can take pointers
  to, so unless we "force-manufacture" one, we can't make this work.


## Alternatives to syntax

We could wait for reflection in which case `declcall` is implementable when we
have expression reflections.

```cpp
namespace std::meta {
  template<info r> constexpr auto declcall = []{
    if constexpr (is_nonstatic_member(r)) {
      return pointer_to_member<[:pm_type_of(r):]>(r);
    } else {
      return entity_ref<[:type_of:]>(r);
    } /* insert additional cases as we define them. */
  }();
}

int f(int); //1 
int f(long); //2
constexpr auto fptr_1 = [: declcall<^f(1)> :]; // 1
```

It's unlikely to be quite as efficient as just hooking directly into the
resolver, but it does have the nice property that it doesn't take up a whole
keyword.

It *also* currently only works for constant expressions, so it's not
general-purpose. For general arguments, one would need to pass reflections of
arguments, and if those aren't constant expressions, this gets really
complicated. `declcall` is far simpler.

Many thanks to Daveed Vandevoorde for helping out with this example.

## Naming

I think `declcall` is a reasonable name - it hints that it's an unevaluated
operand, and it's how I implemented it in clang.

[codesearch for declcall](https://codesearch.isocpp.org/cgi-bin/cgi_ppsearch?q=declcall&search=Search)
comes up with zero hits.

For all intents and purposes, this facility grammatically behaves in the same
way as `sizeof`, except that we should require the parentheses around the
operand.

We could call it something other unlikely to conflict, but I like `declcall`

- `declcall`
- `declinvoke`
- `calltarget`
- `expression_targetof`
- `calltargetof`
- `decltargetof`
- `resolvetarget`

# Usecases

Broadly, anywhere where we want to type-erase a call-expression. Broad uses in
any type-erasure library, smart pointers, ABI-stable interfaces, compilation
barriers, task-queues, runtime lifts for double-dispatch, and the list goes on
and on and on and ...

## What does this give us that we don't have yet

### Resolving overload sets for callbacks without lambdas

```cpp
// generic context
std::sort(v.begin(), v.end(), [](auto const& x, auto const& y) {
    return my_comparator(x, y); // some overload set
});
```

becomes

```cpp
// look ma, no lambda, no inlining, and less code generation!
std::sort(v.begin(), v.end(), declcall(my_comparator(v.front(), v.front()));
```

Note also, that in the case of a `vector<int>`, the ABI for the comparator is
likely to take those by value, which means we get a better calling convention.

`static_cast<bool(*)(int, int)>(my_comparator)` is not good enough here - the
resolved comparator could take `long`s, for instance.

### Copy-elision in callbacks

We cannot correctly forward immovable type construction through forwarding
function.

Example:

```cpp
int f(nonmovable) { /* ... */ }
struct {
    // doesn't work
    static auto operator()(auto&& obj) const {
        return f(std::forward<decltype(obj)>(obj)); // 1
    }
    // would work if we also had replacement function / expression aliases
    static auto operator()(auto&& obj) const 
       = declcall(f(std::forward<obj>(obj)));      // 2
} some_customization_point_object;

void continue_with_result(auto callback) {
    callback(nonmovable{read_something()});
}

void handler() {
    continue_with_result(declcall(f(nonmovable{}))); // works
    // (1) doesn't work, (2) works
    continue_with_result(some_customization_point_object);
}
```

## That's not good enough to do all that work. What else?

Together with [@P2826R0], the two papers constitute the ability to implement
_expression-equivalent_ in many important cases (not all, that's probably
impossible).

[@P2826R0] proposes a way for a function signature to participate in overload
resolution and, if it wins, be replaced by some other function.

This facility is the key to *finding* that other function. The ability to
preserve prvalue-ness is crucial to implementing quite a lot of the standard
library customization points as mandated by the standard, without compiler
help.

# Needed Guidance

- Do we want to punt the syntax to reflection, or is this basic enough to warrant
  this feature? (Knowing that reflection will need more work from the user to
  get the pointer value).
- Do we care that it only works on unevaluated operands? (With the
  `static_cast` fallback in run-time cases)


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
