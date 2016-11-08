- Feature Name: trait_upcasting
- Start Date: YYYY-MM-DD
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

To implement RFC 401/982 supertrait coercions, lay out vtables of trait objects
so they embed all supertrait vtables in `prefix` or `prefix + constant offset`
position. Keep this consistent for all vtables for a single trait, even if
they originate from different impls. Implement upcasts by adding a statically
determined offset to the trait object's vtable pointer.

# Motivation
[motivation]: #motivation

RFC 401 specifies the ability to coerce from a trait object `&Trait` to a trait
object of a supertrait `&Supertrait`, and RFC 982 extends this to custom types
like `Rc<Supertrait>`. However, neither RFC details precisely how this feature
should be implemented.

To implement this feature, the trait object vtable layout must be changed.
Currently, rustc generates vtables containing all methods of a trait and its
supertraits (as well as some metadata). This makes it possible to call
supertrait methods on the trait object, but offers no way to convert a
trait object to one pointing to a supertrait, because supertrait vtables are
not generally contained in the subtrait vtable. This RFC proposes such a vtable
layout, geared towards for run-time efficiency.

Because RFCs 401 and 982 do not detail the general rationale for efficient
trait object upcasting, we will do so here. Chiefly, upcasting would help to
increase code reuse and to bridge the expressiveness gap between trait objects
and generics.

## Code reuse

Trait objects facilitate code reuse in that functions that accept them, and
data structures that store them, can be used with objects created from
different concrete types. Upcasting would extend this substitutability to
different trait objects with common ancestor traits.

## Trait objects and generics

Generics support compile-time polymorphism; trait objects support run-time
polymorphism. Both types of polymorphism are based on the interfaces defined
by traits. In certain cases, the semantics of the two are similar enough that
switching between them can be made trivial for the user.

Rust already does a good job of this for simple polymorphic functions. Often,
only the function signature needs to change.

```rust
trait T { fn trait_method(&self); }

// Compile-time
fn ct<T: Trait>(x: &T) { x.trait_method(); }

// Run-time
fn rt(x: &Trait) { x.trait_method(); }
```

This is currently impossible if one generic function calls into another with
narrower constraints.

```rust
trait Super { fn trait_method(&self); }
trait Trait : Super { }

// Compile-time
fn inner<T: Super>(x: &T) { x.trait_method(); }
fn outer<T: Trait>(x: &T) { inner(x); }
```

Trait object upcasting makes it work:

```rust
Run-time
fn inner(x: &Super) { x.trait_method(); }
fn outer(x: &Trait) { inner(x); }
```

Not every generic function can be trivially de-genericized (e.g. due to object
safety requirements, because the original accepts its parameters by value, or
because the generic bounds cannot be expressed in trait object form), but trait
object upcasting would make it possible in more situations.

This kind of change is one a programmer may want to make late in the
development cycle as an optimization, especially since the trade-offs are not
necessarily clear ahead of time. The expectation is that compile-time
polymorphism produces larger, faster code, but it can be difficult to predict
the exact impact. Rust should make this translation no harder than it needs to
be to facilate experimentation.

(Note that the generic functions above can accept trait objects if a ?Sized
bound is added, but this doesn't always produce the desired result. A generic
function with an `&`-parameter constrained on `Trait` may be separately
instantiated with `&Subtrait1` and `&Subtrait2`. Forcing the instantiation to
use `Trait` instead may be worth exploring in a future RFC.)

## Vtable size

The current representation is not always optimal in terms of memory use, in
that the method list for a given trait will be duplicated in the vtable for
that trait and the vtable for each subtrait, if trait objects are created for
that trait and its subtraits. A new vtable representation should avoid
duplication, to the extent possible without performance compromises.

# Detailed design
[design]: #detailed-design

Implement supertrait coercions from `&Trait` to `&Supertrait` (and similar), as
described in RFC 401 and 982 taken together.

## Vtable layout

Given a concrete type implementing one or more traits, create a directed
acyclic graph. Add one node for each object-safe trait the type implements. Add
an edge from each trait to each of its supertraits.

Prune dead nodes: those that are not reachable from some trait for which we
actually instantiate a trait object.

Prune redundant edges in the graph: given node `A` and direct child node `B`,
remove the edge if there is any other (longer) path from `A` to `B`.

This should produce a version of the graph with the minimal number of nodes
that still ensures that all supertrait nodes of a trait are reachable from that
trait's node.

Traverse the graph in depth-first prefix order, starting at each root node.
Some nodes may be visited multiple times. During the traversal, create vtables
according to the following algorithm (in which `supertraits` corresponds to the
list of child nodes of each node, which must be sorted according to some
canonical order):

```
// In pseudocode:
fn vtable_layout(t: Trait) -> [VtableElement] {
    let vtable = []
    t.supertraits.sort()
    if t.supertraits.is_empty {
        vtable = [drop_glue, alignment, size] // metadata
    } else {
        for st in t.supertraits {
            vtable ++= vtable_layout(st)
        }
    }
    vtable ++= t.methods
    vtable
}
```

Every supertrait vtable is embedded in the vtable of a given trait.
Consequently, coercions from `&Trait` to `&Supertrait` can be implemented
using a constant offset added to the vtable pointer.

When creating trait objects, use the vtable corresponding to the first time
that trait's node is visited. (This isn't strictly necessary).

**TODO: Add diagrams, consider shorter explanation?**

# Drawbacks
[drawbacks]: #drawbacks

This approach to trait object upcasting with a constant offset necessitates a
certain amount of duplicated data, even in code that doesn't itself use
upcasting. As is detailed in the next section, the cost compared to the status
quo depends on how a given program makes use of trait objects, and does not
always involve a penalty.

Committing to upcasting, or to a specific vtable layout, could conceivably
conflict with future extensions to the language. However, I am not aware of a
specific conflicting proposal. Closely related features that do not conflict
with this have been proposed at various times, e.g. RFC 250.

One notable consequence is that two trait objects of the same trait could have
different vptrs even if they were created from the same concrete type,
depending on the path they took up the inheritance chain. Currently, Rust
doesn't make any guarantees about vptr identity, but it should be clear that
this RFC closes the door on one way to efficiently check whether two trait
objects were created from the same concrete type. That said, moving to
a different vtable layout would constitute a backwards-compatible change, which
should mitigate this concern.

# Alternatives
[alternatives]: #alternatives

There are several alternatives to consider.

1. Keep the current vtable layout
2. Keep the current layout, but combine prefix vtables to save space
3. This RFC
4. Dynamic supertrait offsets
5. Use a manual workaround
6. Opt-in-based upcasting

Options 1 and 2 entail giving up on trait upcasting. Option 5 requires
additional work on the part of the user (and would exist alongside 1 or 2), but
is best presented separately to consider its costs.

Several of these options require further explanation before we are ready to
make a comparison.

## Combine prefix vtables

The current implementation produces vtables that are prefixes of one another.
These could easily be combined. In single inheritance scenarios, only the
vtable of the last child needs to be kept, since it contains each of the parent
trait vtables as a prefix. Separate vtables are only needed under multiple
inheritance.

## Dynamic supertrait offsets

Instead of determining the location of each supertrait statically, the metadata
of each vtable could be expanded to include space for the byte offset of each
supertrait. This would guarantee that each method list is stored only once, but
require extra space for the offsets. It would require a dependent read at
run-time, so it would be likely to be slower.

**TODO: would it really be that much of an impact? the offset and supertrait
methods may even sit in the same cache line. don't want to just assume**

## Opt-in

Upcasting could be allowed on an opt-in basis, using an unsized lang item
wrapper: `t as &Upcastable<Trait>` instead of `t as &Trait`. Coercions from
`&Upcastable<Trait>` to `&Upcastable<Supertrait>` would work as described
above, and regular trait objects could remain the way they are now.

This would make sense if allowing upcasting had significant drawbacks, such as
violating the "you don't pay for what you don't use" principle.

## Workaround

It's possible to work around the lack of language-level object upcasting by
manually implementing trait methods instead.

```
trait Trait: Super {
    fn as_super(&self) -> &Super { self }
}
```

This can also be centralized into the supertrait, so it doesn't require
additional code in child traits or implementors:

```
// From http://stackoverflow.com/a/28664881

trait Super : AsSuper {
    // Super methods...
}

trait AsSuper {
    fn as_super(&self) -> &Super;
}

impl<T: Super> AsSuper for T {
    fn as_super(&self) -> &Super { self }
}
```

This approach requires separate functions for each kind of trait object
(`Box<Trait>`, `&Trait`, etc.), and a separate `AsSuper`-style trait for each
supertrait (`AsSuper` cannot be made more generic because
`trait Super : As<Super>` creates an illegal cycle).

Most of this boilerplate could be avoided with a macro, but the need to make an
explicit call to `as_super` cannot be sidestepped.

This adds an additional function pointer for each kind of supported upcasting,
for each vtable. At run-time, the upcast uses as a dynamically dispatched
function call, rather than an offset calculation or no-op.

## Costs

### Vtable size

We will consider four scenarios. In the diagrams below, boxes `[ ]` signify
traits, and the letter `X` signifies that at least one trait object was created
for a certain trait.

A. Three levels of single inheritance

    [X]
     |
    [X]
     |
    [X]

B. Multiple inheritance

    [X] [X] [X] 
     |   |   |
     +---+---+
     |
    [X]

C. Triple diamond

    [X]
     |
     +---+---+
     |   |   |
    [X] [X] [X]
     |   |   |
     +---+---+
     |
    [X]

D. Triple diamond, few trait objects

    [X]
     |
     +---+---+
     |   |   |
    [X] [ ] [ ]
     |   |   |
     +---+---+
     |
    [X]

For simplicity, every trait in these scenarios will have two methods.

We count the number of words needed to represent all the vtables in the
hierarchy, assuming that pointers are a single word, and that the metadata
consists of three words.

The scenarios are numbered as above. Note that the 'workaround' is based on the
'combined' approach, since it is more space-efficient.

| Solution        |  A |  B |  C |  D | Allows upcasting? | Notes      |
| ----------------|----|----|----|----|-------------------|----------- |
| 1. Current      | 21 | 26 | 39 | 21 | No                |            |
| 2. Combined     |  9 | 21 | 27 |  9 | No                |            |
| 3. This RFC     |  9 | 17 | 23 | 23 | Yes               |            |
| 4. Dynamic      | 15 | 20 | 20 | 20 | Yes               | Slower?    |
| 5. Workaround   | 11 | 26 | 35 | 11 | Yes               | Extra code |

Algorithm `1`, the current approach, produces the largest vtables in every case.

The combining approach (`2`) and the proposed approach (`3`) behave quite
similarly in these scenarios, with the proposed approach having a slight edge
except in scenario D.

In fact, `3` will always produce smaller vtables when trait objects are created
for every trait in the hierarchy. If there are `N` paths to reach a given node,
`3` will duplicate the entire subgraph `N` times. But when trait objects are
created for the full hierarchy, `2` actually suffers from the same problem
*and* requires additional space for method lists in the middle part of a
diamond.

A visualization of the 'triple diamond' case might elucidate this. The original
trait hierarchy is shown on the left, followed by a representation of the
elements that remain in the final vtables.

    Original hierarchy x   2. Combined   x   3. This RFC
                       x                 x
                       x                 x    Q        
         Q             x    Q   Q   Q    x    |         
         |             x    |   |   |    x    R         
         +---+---+     x    R   S   T    x    |   Q     
         |   |   |     x    |            x    |   |     
         R   S   T     x    S            x    |   S     
         |   |   |     x    |            x    |   |   Q 
         +---+---+     x    T            x    |   |   | 
         |             x    |            x    +---+---T 
         U             x    U            x    |         
                       x                 x    U         

Note how under solution `2`, method pointers for `S` and `T` are duplicated.

`2` gains an edge when only parts of the hierarchy need vtables. This is
because `2` makes it possible to avoid instantiating some of the vtables that
could be created for the hierarchy.

Under `3` we cannot, in general, remove any of the other nodes if a trait
object is created for `U`. We would need to prove that no trait objects of `U`
are upcast to a trait from that subtree, even in another compilation unit.

Scenario D illustrates the impact of the pruning ability of solution `2`.

* **TODO: Are those numbers right? Under what cases is the new way actually
  worse? Is this comparison misleading and should I use a different approach to
  the analysis?**
* **TODO: Should at least try to talk about size complexity.**
* **TODO: Evaluate 'opt-in'? (How?)**
* **TODO: Is this always true, or am I missing something? (Proof needed?)**

# Unresolved questions
[unresolved]: #unresolved-questions

* Is the proposed level of vtable bloat acceptable?
* How will this be extended for multi-trait objects `&(Foo+Bar)`? (This is
  probably orthogonal.)
* Does this conflict with any known proposals?
* Should we try to prune unused child nodes where possible? (This is more
  difficult if trait objects are passed across compilation units. In any case,
  this can be added backwards-compatibly)
