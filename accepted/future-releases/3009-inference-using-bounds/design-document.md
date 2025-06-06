# Inference using bounds

Author: Chloe Stefantsova

Status: Accepted

Version 1.0 (see [CHANGELOG](#CHANGELOG) at end)

Experiment flag: inference-using-bounds

## Changes in inference.md

This document discusses the effect and the motivation of the following changes
in [inference.md][]. Steps are added to the algorithms described in sections
[Constraint solution for a set of type variables][] and [Grounded constraint
solution for a set of type variables][].

```diff
@@ -714,24 +714,30 @@ occurences of the unknown type.
 #### Constraint solution for a set of type variables

 The constraint solution for a set of type variables `{X0, ..., Xn}` with respect
 to a constraint set `C` and partial solution `{T0, ..., Tn}`, is defined to be
 the set of type schemas `{U0, ..., Un}` such that:
   - If `Ti` is known (that is, does not contain `_`), then `Ui = Ti`.  _(Note
     that the upcoming "variance" feature will relax this rule so that it only
     applies to type variables without an explicitly declared variance.)_
   - Otherwise, let `Vi` be the constraint solution for the type variable `Xi`
     with respect to the constraint set `C`.
   - If `Vi` is not known (that is, it contains `_`), then `Ui = Vi`.
   - Otherwise, if `Xi` does not have an explicit bound, then `Ui = Vi`.
+  - Otherwise, let `Mb <: Xi <: Mt` be the merge of `C` with respect to `Xi`. If
+    `Mb` is not `_`, let `C1` be the constraint set produced by the subtype
+    constraint generation algorithm for `P = Mb`, `Q = B`, `L = {X0, ..., Xn}`.
+    Then `Ui` is the constraint solution for the type variable `Xi` with respect
+    to the constraint set `C + C1`. *Note that `X` is in `L` and that `Mb`
+    doesn't contain any of `X0, ..., Xn`.*
   - Otherwise, let `Bi` be the bound of `Xi`.  Then, let `Bi'` be the type
     schema formed by substituting type schemas `{U0, ..., Ui-1, Ti, ..., Tn}` in
     place of the type variables `{X0, ..., Xn}` in `Bi`.  _(That is, we
     substitute `Uj` for `Xj` when `j < i` and `Tj` for `Xj` when `j >= i`)._
     Then `Ui` is the constraint solution for the type variable `Xi` with respect
     to the constraint set `C + (X <: Bi')`.

 _This definition can perhaps be better understood in terms of the practical
 consequences it has on type inference:_
   - _Once type inference has determined a known type for a type variable (that
     is, a type that does not contain `_`), that choice is frozen and is not
     affected by later type inference steps.  (Type inference accomplishes this
@@ -762,24 +768,30 @@ occurences of the unknown type.

 #### Grounded constraint solution for a set of type variables

 The grounded constraint solution for a set of type variables `{X0, ..., Xn}`
 with respect to a constraint set `C`, with partial solution `{T0, ..., Tn}`, is
 defined to be the set of types `{U0, ..., Un}` such that:
   - If `Ti` is known (that is, does not contain `_`), then `Ui = Ti`.  _(Note
     that the upcoming "variance" feature will relax this rule so that it only
     applies to type variables without an explicitly declared variance.)_
   - Otherwise, if `Xi` does not have an explicit bound, then `Ui` is the
     grounded constraint solution for the type variable `Xi` with respect to the
     constraint set `C`.
+  - Otherwise, let `Mb <: Xi <: Mt` be the merge of `C` with respect to `Xi`.
+    If `Mb` is not `_`, let `C1` be the constraint set produced by the subtype
+    constraint generation algorithm for `P = Mb`, `Q = B`, `L = {X0, ..., Xn}`.
+    Then `Ui` is the grounded constraint solution for the type variable `Xi`
+    with respect to the constraint set `C + C1`. *Note that `X` is in `L` and
+    that `Mb` doesn't contain any of `X0, ..., Xn`.*
   - Otherwise, let `Bi` be the bound of `Xi`.  Then, let `Bi'` be the type
     schema formed by substituting type schemas `{U0, ..., Ui-1, Ti, ..., Tn}` in
     place of the type variables `{X0, ..., Xn}` in `Bi`.  _(That is, we
     substitute `Uj` for `Xj` when `j < i` and `Tj` for `Xj` when `j >= i`)._
     Then `Ui` is the grounded constraint solution for the type variable `Xi`
     with respect to the constraint set `C + (X <: Bi')`.

 _This definition parallels the definition of the (non-grounded) constraint
 solution for a set of type variables._
```

[inference.md]: https://github.com/dart-lang/language/blob/main/resources/type-system/inference.md
[Constraint solution for a set of type variables]: https://github.com/dart-lang/language/blob/main/resources/type-system/inference.md#constraint-solution-for-a-set-of-type-variables
[Grounded constraint solution for a set of type variables]: https://github.com/dart-lang/language/blob/main/resources/type-system/inference.md#grounded-constraint-solution-for-a-set-of-type-variables

## Motivating example

The motivating example is discovered and discussed in [#3009][] (Type inference
does not solve some constraints involving F-bounds). Consider the following
code:

```dart
class A<X extends A<X>> {}
class B extends A<B> {}
class C extends B {}

void f<X extends A<X>>(X x) {}

void main() {
  f(B()); // OK.
  f(C()); // Inference fails, compile-time error.
  f<B>(C()); // OK.
}
```

The front ends fail to compile this program, complaining that the
inferred type `C` for `X` doesn't satisfy the bound of `X`.

The expression of interest in the example is `f(C())`. During the
downwards phase of the inference, no constraints are collected because
the type context is empty. During the upwards phase, we build the set
of type constraints `{C <# X}`. Then we generate the overall solution
using the algorithm described in [Constraint solution for a set of
type variables][CSSTV].

Both the constraint set `{C <# X}` and the empty preliminary solution
from the downwards phase `{X = _}` are passed to the algorithm
described in [Constraint solution for a set of type
variables][CSSTV]. The algorithm starts by checking whether the
preliminary solution is known (that is, it doesn't contain the unknown
type `_`), and the check fails. Then the algorithm solves the
constraints for `X`, yielding `X = C` as the solution. Since `C`
doesn't contain `_`, the algorithm checks if `X` has a bound (which it
does: the bound of `X` is `A<X>`), computes the best-effort
approximation of that bound (which is `A<_>`), adds the bound to the
existing set of constraints for `X` (resulting in `{C <# X, X <#
A<_>}`), and tries to solve the updated constraint set for `X`
yielding the same result `X = C`.

Note that adding the best-effort approximation of the bound and
solving the constraint set once again didn't change the outcome. In
general, lower bounds are preferred by the type inference, but the
best-effort approximation is added as an upper-bound constraint.

It is notable that even though the example program contains all the
type information needed to infer `B` as the type parameter, the front
ends following the specification infer `C`, resulting in a
compile-time error during the type checks of the type arguments
against the bounds.

### A more complex example

The following is a more complex example, involving mutually recursive
bounds of two type variables. Currently, it's rejected by the Dart
tools with a compile-time error.

```dart
class A1<X extends A1<X, Y>, Y extends A2<X, Y>> {}
class A2<X extends A1<X, Y>, Y extends A2<X, Y>> {}
class B extends A1<B, B> implements A2<B, B> {}
class C1 extends B {}
class C2 extends B {}

void f<X extends A1<X, Y>, Y extends A2<X, Y>>(X x, Y y) {}

void main() {
  f<B, B>(C1(), C2());
  f(C1(), C2());
  print("Done!");
}
```

In Kotlin a similar program is accepted by the compiler and runs as
expected.

```kotlin
open class A1<out X: A1<X, Y>, out Y: A2<X, Y>>() {}
interface A2<out X: A1<X, Y>, out Y: A2<X, Y>> {}
open class B() : A1<B, B>(), A2<B, B> {}
open class C1() : B() {}
open class C2() : B() {}

fun <X: A1<X, Y>, Y: A2<X, Y>>f(x: X, y: Y) {}

fun main() {
  f<B, B>(C1(), C2());
  f(C1(), C2());
  print("Done!");
}
```

Let's take a look at the source of the described type inference
behavior in Dart.

## Constraint solution for a set of type variables

Currently, the entire effect the bounds of the type variables may have
on the outcome of the inference is defined by the algorithm of
[constraint solution for a set of type variables][CSSTV] (CSSTV). It
is limited in two ways:

- it is invoked in-between the phases of type inference and doesn't
  collect new type constraints from the context, and
- instead of the actual bound, it uses its best-effort approximation,
  eliminating the recursive properties of F-bounded type variables.

A convenient way to graphically represent CSSTV is by a rectangle,
where three sides of it take the three inputs relevant to this
feature, and the fourth side produces the output. The relevant inputs
are (1) the preliminary solution found by the previous phase of the
inference, (2) the constraints collected during the current phase of
the inference, and (3) the bounds of the type variables. The output of
the algorithm is the solution for the current phase, which becomes the
preliminary solution for the next phase.

```mermaid
block-beta
  columns 5

  space:2
  previousPhase["Previous phase"]
  space:2

  space:5

  block:current:5
    columns 5

    commentCurrent(["Current phase:"])
    space:4

    constraintsCurrent["(2) Constraints"]
    space
    csstvCurrent["CSSTV"]
    space
    boundsCurrent["(3) Bounds"]
  end

  space:5

  space:2
  nextPhase["Next pahse"]
  space:2

  previousPhase -- "(1) Preliminary solution" --> csstvCurrent

  constraintsCurrent --> csstvCurrent
  boundsCurrent --> csstvCurrent
  csstvCurrent -- "Current solution" --> nextPhase
```

Since the output from CSSTV for one phase becomes an input for CSSTV
for the next phase, the entire process of type inference can be
thought of as multiple runs of the CSSTV chained together, each taking
the collected constraints from its respective phase.

```mermaid
block-beta
  columns 5

  space:2
  overallInput["Input"]
  space:2

  space:5

  block:downwards:5
    columns 5

    commentDownwardsConstraints(["Downwards phase:"])
    space:4

    constraintsDownwards["Constraints"]
    space
    csstvDownwards["CSSTV"]
    space
    boundsDownwards["Bounds*"]
  end

  space:5

  block:horizontal1:5
    columns 5

    commentHorizontal1Constraints(["Horizontal phase 1:"])
    space:4

    constraintsHorizontal1["Constraints"]
    space
    csstvHorizontal1["CSSTV"]
    space
    boundsHorizontal1["Bounds*"]
  end

  space:5

  block:horizontalN:5
    columns 5

    commentHorizontalNConstraints(["Horizontal phase N:"])
    space:4

    constraintsHorizontalN["Constraints"]
    space
    csstvHorizontalN["CSSTV"]
    space
    boundsHorizontalN["Bounds*"]
  end

  space:5

  block:upwards:5
    columns 5

    commentUpwardsConstraints(["Upwards phase:"])
    space:4

    constraintsUpwards["Constraints"]
    space
    csstvUpwards["CSSTV"]
    space
    boundsUpwards["Bounds*"]
  end

  space:5

  space:2
  overallOutput["Output"]
  space:2

  overallInput -- "Empty solution" --> csstvDownwards

  constraintsDownwards --> csstvDownwards
  boundsDownwards --> csstvDownwards
  csstvDownwards --> csstvHorizontal1

  constraintsHorizontal1 --> csstvHorizontal1
  boundsHorizontal1 --> csstvHorizontal1
  csstvHorizontal1 -- "..." --> csstvHorizontalN

  constraintsHorizontalN --> csstvHorizontalN
  boundsHorizontalN --> csstvHorizontalN
  csstvHorizontalN --> csstvUpwards

  constraintsUpwards --> csstvUpwards
  boundsUpwards --> csstvUpwards

  csstvUpwards -- "Overall solution" --> overallOutput
```

_(*) These are all the same type variable bounds, since we infer for the same
type variables._

An important property of CSSTV is the following: if an element of the
preliminary solution from the previous phase is known (that is, it
doesn't contain `_`), it is not changed by the current run of
CSSTV. Effectively, once a known type is produced at the end of any
phase, it becomes "frozen" through the rest of the inference process
and becomes a part of the overall inference result.

The picture below shows the details of the type inference process for the
motivating example using the schematic representation. In addition to the
constraints collected during the upwards phase (1), a new constraint (2) is
generated from the bound of the type variable after completing the first four
steps of the CSSTV algorithm. The new constraint is then added to the set of
existing constraints (1). Then the solution for the type variable is computed
from the updated constraint set (3).

```mermaid
block-beta
  columns 5

  space:2
  overallInput["Input"]
  space:2

  space:5

  block:downwards:5
    columns 5

    commentDownwardsConstraints(["Downwards constraints:"])
    space:3
    commentDownwardsBounds(["Bounds:"])

    constraintsDownwards["∅"]
    space
    csstvDownwards["CSSTV"]
    space
    boundsDownwards["X extends A&lt;X&gt;"]
  end

  space:5

  block:upwards:5
    columns 5

    commentUpwardsConstraints(["Upwards constraints:"])
    space:3
    commentUpwardsBounds(["Bounds:"])

    constraintsUpwards["C <# X"]
    space
    csstvUpwards["CSSTV"]
    space
    boundsUpwards["X extends A&lt;X&gt;"]
  end

  space:5

  space
  block:constraintsUpwardsUpdated:1
    columns 1
    constraintUpwards2["C <# X"]
    constraintUpwards1["X <# A&lt;_&gt;"]
  end
  space
  constraintGeneration["X <# A&lt;_&gt;"]
  space

  space:2
  overallOutput["Output"]
  space:2

  overallInput -- "{X = _}" --> csstvDownwards

  constraintsDownwards --> csstvDownwards
  boundsDownwards --> csstvDownwards
  csstvDownwards -- "{X = _}" --> csstvUpwards

  constraintGeneration --> constraintUpwards1

  constraintsUpwards -- "(1) Existing constraints\nfrom the\nupwards phase" --> constraintUpwards2
  constraintsUpwards --> csstvUpwards
  boundsUpwards --> csstvUpwards
  boundsUpwards -- "(2) Constraint from the best-effort\napproximation of the bound" --> constraintGeneration
  constraintsUpwardsUpdated -- "(3) Proceed with CSSTV using\nthe updated constraint set" --> csstvUpwards

  csstvUpwards -- "{X = C}" --> overallOutput
```

## Rules

The unsatisfactory outcome described in the motivating example is
solved by generating more constraints from the existing ones and from
the bounds of the type parameters being constrained. The current
section describes the application of that solution to the motivating
example and concludes with more examples of improvements.

In the motivating example, adding the best-effort bound approximation
didn't affect the result. The set of constraints for `X` already
contained a lower-bound constraint for `X` (`C <# X`), and that
constraint has the priority over the upper-bound best-effort
approximation constraint, since type `C` is fully known (i.e. doesn't
contain `_`).

We derive a new lower-bound constraint by combining the already existing
lower-bound constraint with the actual bound.

In general, if we have a type constraint `E <# Y` and a subtype relation `Y <:
F`, where `Y` is a type variable, `E` is a type schema, and `F` is a type,
possibly containing `Y`, it follows that `E <# F` holds if the `E <# Y` and `Y
<: F` hold. So, instead of adding the constraint `X <# B'` in step 5 of CSSTV,
where `B'` is the best-effort bound approximation, we generate a new set of
constraints, running the subtype constraint generation algorithm described in
[Subtype constraint generation][subtype-constraint-generation] for `P = C`, `Q
= B`, `L = {..., X, ...}`.

By construction, `C` doesn't contain type variables from `L`. `B` may
contain type variables from `L` (surely, in contains `X` in case where
`X` is F-bounded, but it may also contain other type variables from
the same declaration, whether or not `X` if F-bounded), which makes `C
<# B` a valid input for [Subtype constraint
generation][subtype-constraint-generation] and a good source of
additional type information.

The new constraints, generated by [Subtype constraint
generation][subtype-constraint-generation] from `C <# B` are then
added to the overall set of constraints for the current phase of type
inference. Then, the solution for type variable `X` is computed from
the new constraint set, at the end of the updated step 5 of CSSTV.
Note that when `X` isn't F-bounded and its bound doesn't refer to
other type variables from the same declaration, the suggested changes
for CSSTV work exactly as the unmodified version of that algorithm.

## Re-evaluating the motivating example

Let's see how the updated type inference works in the motivating
example.

The downwards phase isn't affected, since it doesn't provide any
contextual type information in this specific example. In the upwards
phase, before step 5 of CSSTV we have the partial solution `X = _`,
the constraint set `{C <# X}`, and the solution of the constraint set
for `X` is `C`, which is known (that is, it doesn't contain `_`).

At step 5 of the modified CSSTV, `C <# X` is a lower-bound constraint
of `X`, so we run [Subtype constraint
generation][subtype-constraint-generation] with `P = C`, `Q = A<X>`
(where `A<X>` is the bound of `X`), `L = {X}`. The next steps are as
follows: `C <# A<X>` → `A<B> <# A<X>` → `B <# X`, which is the new
constraint to add.

We add `B <# X` to the constraint set for `X` (or, equivalently, merge
it with the already merged constraint set for `X`, depending on the
implementation) — it gives us `{C <# X, B <# X}`, and the solution for
`X` becomes `X = B` at the end of step 5 of CSSTV.

`X = B` is "frozen" since B is known, and the overall solution for `X`
is `B`, allowing the motivating example to compile and run without
errors.

The picture below shows the details of the updated type inference
process in the motivating example. The constraints collected during
the upwards phase (1) contribute the lower bound `C` (2a) as the input
`P` to [Subtype constraint generation][subtype-constraint-generation],
and the type parameter bound contributes the upper bound `A<X>` (2b)
as the input `Q` to the same algorithm.  The newly generated
constraint (2c) is then added to the set of existing constraints (1).
Then the solution for the type variable is computed form the updated
constraint set (3).

```mermaid
block-beta
  columns 5

  space:2
  overallInput["Input"]
  space:2

  space:5

  block:downwards:5
    columns 5

    commentDownwardsConstraints(["Downwards constraints:"])
    space:3
    commentDownwardsBounds(["Bounds:"])

    constraintsDownwards["∅"]
    space
    csstvDownwards["CSSTV"]
    space
    boundsDownwards["X extends A&lt;X&gt;"]
  end

  space:5

  block:upwards:5
    columns 5

    commentUpwardsConstraints(["Upwards constraints:"])
    space:3
    commentUpwardsBounds(["Bounds:"])

    constraintsUpwards["C <# X"]
    space
    csstvUpwards["CSSTV"]
    space
    boundsUpwards["X extends A&lt;X&gt;"]
  end

  space:10

  space
  block:constraintsUpwardsUpdated:1
    columns 1
    constraintUpwards2["C <# X"]
    space
    constraintUpwards1["B <# X"]
  end
  space
  constraintGeneration["C <# A&lt;X&gt; ==> B <# X"]
  space

  space:2
  overallOutput["Output"]
  space:2

  overallInput -- "{X = _}"--> csstvDownwards

  constraintsDownwards --> csstvDownwards
  boundsDownwards --> csstvDownwards
  csstvDownwards -- "{X = _}" --> csstvUpwards

  constraintGeneration -- "(2c) New constraints" --> constraintUpwards1
  constraintUpwards2 -- "(2a) Contribute\nC <# X to C <# A&lt;X&gt;" --> constraintGeneration

  constraintsUpwards -- "(1) Existing constraints\nfrom the\nupwards phase" --> constraintUpwards2
  constraintsUpwards --> csstvUpwards
  boundsUpwards --> csstvUpwards
  boundsUpwards -- "(2b) Contribute A&lt;X&gt; to C <# A&lt;X&gt;\n" --> constraintGeneration
  constraintsUpwardsUpdated -- "(3) Proceed with CSSTV using\nthe updated constraint set" --> csstvUpwards

  csstvUpwards -- "{X = B}" --> overallOutput
```

## More examples enabled by this feature

In addition to both motivating examples, this feature also handles
some cases involving type variables that are not F-bounded.

With a class like `A` below, and with this feature, the constructor
invocation in `test` is inferred as `A<List<num>, num>(<num>[])`.
Without this feature, it is inferred as `A<List<num>, dynamic>(<num>[])`.

Intuitively, this happens because the type inference algorithm without this
feature is unable to detect that there is a relationship between `X` and `Y`.
With this feature one could say that the value of `Y` is "extracted" from
the value of `X`. We have seen cases in practice demonstrating that the
ability to transfer this information during type inference is useful.

```dart
class A<X extends Iterable<Y>, Y> {
  A(X x);
  Y? y;
}

test() {
  // Developer thinks: A<List<num>, num>.
  A(<num>[])..y = "wrong"; // Today: Ok. With the update: Compile-time error.
}
```

[#3009]: https://github.com/dart-lang/language/issues/3009
[CSSTV]: https://github.com/dart-lang/language/blob/main/resources/type-system/inference.md#constraint-solution-for-a-set-of-type-variables
[subtype-constraint-generation]: https://github.com/dart-lang/language/blob/main/resources/type-system/inference.md#subtype-constraint-generation

## Changelog

### 1.0

- Final design document.
