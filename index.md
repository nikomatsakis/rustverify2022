class: center
name: title
count: false

# A MIR Formality

.formalferris[![Ferris](./images/rustacean-orig-noshadow.svg) ![Black tie](./images/Man_wearing_black_tie.jpg)]

.me[.grey[*by* **Nicholas Matsakis**]]
.left[.citation[View slides at `https://nikomatsakis.github.io/rustverify2022/` <br/> Tuxedo image by KoeppiK - Own work, CC BY-SA 4.0, [wikimedia url](https://commons.wikimedia.org/w/index.php?curid=85669777)]]

---

# What is "a MIR formality"?

--

You mean *besides* a world-class pun?

--

![golf-clap](images/golf-clap.gif)

---

# What is "a MIR formality"?

No, seriously:

> This repository is an **early-stage experimental project** that aims to be a **complete, authoritative formal model** of the Rust MIR. **Presuming** these experiments bear fruit, the intention is to bring this model into Rust as an RFC and develop it as an **official part of the language definition**.

(emphasis mine)

---

# The bigger picture

Rust **raising the bar** on reliability for users:

* Start off with the borrow checker.
* Tooling that makes thorough testing easy and accessible.
    * Including detecting UB in unsafe code.
* Ready for more? 
    * Add in some pre- and post-conditions, maybe test dynamically.
* Not enough for ya? 
    * Try this model checker! I heard it's nice!

TL;DR? Rust: Making the **fancy stuff** easy and accessible!

---

# Raising the bar on reliability: internally

Within Rust org:

* Definitive model of Rust type system and operational semantics
    * Including unsafe code
* New language features added before stabilization
* A team dedicated to owning the model and its implementation
* Standard library proven sound
* Fuzzing of the compiler

---

# I know what you're thinking

--

![Reality check](./images/reality-check.gif)

---

# How we get there

Lots of work to do, but it all starts with a firm knowledge of how *safe Rust* works

That's where "a MIR formality" comes in.

---

# What I'm hoping for from y'all

**Nitty gritty:**

I'd love to sit down with people and go over the specifics of what I've done so far. I'm here till Thursday.

**Bigger picture:**

Let's brainstorm about that picture I was painting and the best things we can do to move it forward!

---

# What I'm hoping for from y'all

Or maybe you think this idea will never work?

--

![Never tell me the odds](./images/never-tell-me-the-odds.gif)

.footnote[(Yes, I just wanted to sneak in this GIF for some reason. Sue me.)]

---

# What is "a MIR formality"?

* PLT Redex¹ model
    * accessible, hackable, fun
* May change later

.footnote[¹ "Semantics Engineering with PLT Redex" by Felleisen et al.]

---

# Structured in several layers

* **Core logic**

----------

Proving abstract predicates like

* `∀X ∃Y (X == Y)` 
* `((P ⇒ Q) ∧ P) ⇒ Q`

Entirely independent from Rust

---

# Structured in several layers

* Core logic
* **Rust Types**

----------

Defining Rust types and their relationships to one another

e.g. `&'a u32 <: &'b u32` if `'a: 'b`

---

# Structured in several layers

* Core logic
* Rust Types
* **Rust Declarations**

----------

Defines syntax for declarations of things, but no function bodies

```rust
impl<T: Debug> Debug for Vec<T> { ... }
```

Converting them into *clauses* and *well-formedness goals*:

* Clauses: inference rules implies by the code
    * e.g., `HasImpl(Vec<T>: Debug) :- Implemented(T: Debug)`
* WF Goals: must prove these to show program is valid
    * e.g., `Implemented(T: Debug) ⇒ Implemented(Vec<T>: Debug)`

---

# Structured in several layers

* Core logic
* Rust Types
* Rust Declarations
* **MIR Function Bodies**
* **MIR Operational Semantics**
* **... probably some other stuff ...**
* **Rust surface syntax**

---

# Why these layers?

* Each layer is simple on its own.
* Syntax-directed translation from one to the next.
* Clarify "how complex" of a change a given feature is.

---

# Which Rust?

* Current goal: *Rust of the near future*
* Eventually:
    * Stable Rust
    * Nightly Rust

---

# Core logic

---

* Start with **Horn clauses**, like Prolog

------

```rust
impl<T: Debug> Debug for Vec<T> { ... }
```

becomes

```rust
forall<T> {
    HasImpl¹(Vec<T>: Debug) :-
        Implemented(T: Debug)
}
```

.footnote[¹ I will explain HasImpl vs Implemented, I promise!]

---

* Start with **Horn clauses**, like Prolog

------

**Problem:** Horn clauses are pretty limited. You can have a `forall` in a *clause*, for example, but not in a *goal*.

Consider:

```rust
fn foo<T: Clone>(t: T) -> T {
    t.clone()
}
```

We want to prove this goal, but that's beyond Horn clauses

```rust
forall<T> {
    if Implemented(T: Clone) {
        Implemented(T: Clone)
    }
}
```

---

* Start with **Horn clauses**, like Prolog
* Blend in **Hereditary Harrop predicates**, like λProlog¹

.footnote[¹ "Programming with Higher-Order Logic" by Dale Miller, Gopalan Nadathur]

------

Hederitary Harrop predicates permit goals with `forall` and `implication`, not just `exists`.

We can now express generic functions in Rust.

---

* Start with **Horn clauses**, like Prolog
* Blend in **Hereditary Harrop predicates**, like λProlog

-------

**Problem:** Cycles and auto traits like `Send`

```rust
struct MyList<T> {
    data: T,
    next: Option<Box<MyList<T>>>
}
```

`MyList<X>` is `Send` if all of reachable data is `Send`. Most obvious formulation has one goal per field:

```rust
forall<T> {
    Implemented(MyList<X>: Send) :-
        Implemented(T: Send),
        Implemented(Option<Box<MyList<T>>>: Send).
}
```

---

* Start with **Horn clauses**, like Prolog
* Blend in **Hereditary Harrop predicates**, like λProlog

-------

```rust
forall<T> {
    Implemented(MyList<X>: Send) :-
        Implemented(T: Send),
        Implemented(Option<Box<MyList<T>>>: Send).
}
```

But apply that to `MyList<u32>`...

* `MyList<u32>: Send`
--

* `u32: Send` ✅
--

* `Option<Box<MyList<u32>>>: Send`
--

* `Box<MyList<u32>>: Send`
--

* `MyList<u32>: Send` ❌ cycle!

---

* Start with **Horn clauses**, like Prolog
* Blend in **Hereditary Harrop predicates**, like λProlog
* Add a dash of **coinduction**, a la CoLP², to taste

-------

With CoLP, cycles of coinductive predicates are generally accepted. (We don't, however, need or want infinite terms.)

.footnote[² "Coinductive Logic Programming" by Luke Simon et al.]

---

* Start with **Horn clauses**, like Prolog
* Blend in **Hereditary Harrop predicates**, like λProlog
* Add a dash of **coinduction**, a la CoLP, to taste

-------

Current solver uses co-SLD algorithm and solves goals with:

* "builtin" operations (∧, ∨, ∀, ∃, ⇒) 
* opaque **predicates**, defined by upper layers
* **relations** like `T1 <: T2` or `T: R`, defined by upper layers

Will eventually model Rust's actual solver.

---

name:goal-definition

```rust
(Goal ::=
      Predicate
      Relation
      BuiltinGoal)
(BuiltinGoal ::=
             (All (Goal ...))
             (Any (Goal ...))
             (Implies Hypotheses Goal)
             (Quantifier KindedVarIds Goal)
             )
(Quantifier ::= ForAll Exists)
(Relation ::= (Parameter RelationOp Parameter))
(RelationOp ::= == <= >=)
(Predicate Parameter ::= Term)
```

.footnote[[Source](https://github.com/nikomatsakis/a-mir-formality/blob/195a04db4d4fb7809df84d97f5899a29f65eb407/src/logic/grammar.rkt#L64-L73)]

---

template:goal-definition

.GoalArrow[![Arrow](./images/Arrow.png)]

---

template:goal-definition

.PredicateReferenceArrow[![Arrow](./images/Arrow.png)]

---

template:goal-definition

.PredicateDefinitionArrow[![Arrow](./images/Arrow.png)]

---

template:goal-definition

.RelationReferenceArrow[![Arrow](./images/Arrow.png)]

---

template:goal-definition

.RelationDefinitionArrow[![Arrow](./images/Arrow.png)]

---

template:goal-definition

.BuiltinGoalDefinitionArrow[![Arrow](./images/Arrow.png)]

---

name:solver-definition

```racket
(define-judgment-form formality-logic
  #:mode (prove I I I O)
  #:contract (prove Env Prove/Stacks Goal Env_out)

  ...

  [(prove Env Prove/Stacks Goal_1 Env_out)
   ------------------------------------------ "prove-any"
   (prove Env Prove/Stacks (Any (Goal_0 ... Goal_1 Goal_2 ...)) Env_out)
   ]

  ...
  )
```

---

template:solver-definition

.ProveArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveModeArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveContractArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveContractEnvArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveContractStackArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveContractGoalArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveContractEnvOutArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveAnyArrow[![Arrow](./images/Arrow.png)]

---

template:solver-definition

.ProveGoalIArrow[![Arrow](./images/Arrow.png)]

---

name:prove-clause

```racket
(define-judgment-form formality-logic
  #:mode (prove I I I O)
  #:contract (prove Env Prove/Stacks Goal Env_out)

  ...

  [(not-in-stacks Env Predicate Prove/Stacks)
   (where (_ ... Clause _ ... ) (env-clauses-for-predicate Env Predicate))
   (clause-proves Env Prove/Stacks Clause Predicate Env_out)
   --------------- "prove-clause"
   (prove Env Prove/Stacks Predicate Env_out)
   ]

  ...
  )
```

---

template:prove-clause

.ProvePredicateArrow[![Arrow](./images/Arrow.png)]

---

template:prove-clause

.ProveNotInStacksArrow[![Arrow](./images/Arrow.png)]

---

template:prove-clause

.ProveEnvClausesArrow[![Arrow](./images/Arrow.png)]

---

name:env-clauses-for-predicate

```racket
(define-metafunction formality-logic
  ;; Returns all program clauses in the environment that are
  ;; potentially relevant to solving `Predicate`
  env-clauses-for-predicate : Env Predicate -> Clauses

  [(env-clauses-for-predicate Env Predicate)
   ,(let ((clauses-fn (formality-logic-hook-clauses (term any))))
      (clauses-fn (term Predicate)))
   (where/error (Hook: any) (env-hook Env))
   ]
  )
```

.footnote[[Source](https://github.com/nikomatsakis/a-mir-formality/blob/195a04db4d4fb7809df84d97f5899a29f65eb407/src/logic/hook.rkt#L20-L25)]

---

template:env-clauses-for-predicate

.EnvClausesArrow[![Arrow](./images/Arrow.png)]

---

template:env-clauses-for-predicate

.EnvClausesScreamFace[😱]

.EnvClausesScreamArrow[![Arrow](./images/Arrow.png)]

---

# ~~Core logic~~

# Rust types

---

# Rust types

Defines:

* Grammar of types, lifetimes, and where clauses
* Subtyping rules
* Type inference

---

```rust
(define-extended-language formality-ty formality-logic
  ...

  (Parameter ::= Ty Lt)
  (Predicate ::=
             (Implemented TraitRef)
             (HasImpl TraitRef)
             (WellFormed (ParameterKind Parameter))
             )
  (TraitRef ::= (TraitId (Parameter ...)))
  (ParameterKind ::= TyKind LtKind)

  ...
  )
```

---

# Rust types

```racket
(define-metafunction formality-ty
  ty:relate-parameters : Env Relation -> (Env Goals) or Error

  [(ty:relate-parameters Env Relation)
   (relate/one (existential-vars-in-env Env) Env Relation)]
  )
```

---

# ~~Core logic~~

# ~~Rust types~~

# Rust declarations

---

# Rust declarations

Defines the semantics and type rules for most Rust items:

* Struct, enum declarations
* Traits
* Impls (inherent, trait)
* Functions (but no function bodies)

---

name:formality-decl-grammar

```rust
(define-extended-language formality-decl formality-ty
  (DeclProgram ::= (CrateDecls CrateId))
  
  (CrateDecls ::= (CrateDecl ...))
  (CrateDecl ::= (CrateId CrateContents))
  (CrateContents ::= (crate (CrateItemDecl ...)))
  (CrateItemDecl ::= AdtDecl TraitDecl TraitImplDecl)

  ...
  )
```

---

template:formality-decl-grammar

.DeclProgramArrow[![Arrow](./images/Arrow.png)]

---

template:formality-decl-grammar

.CrateItemDeclArrow[![Arrow](./images/Arrow.png)]

---

# Rust declarations

Given a `DeclProgram`, two key tasks:

* Create **program clauses** that define what is *true*
* Create **wellformedness goals** that define what a legal Rust program is

---

# Clauses

Predicates we use:

* **`WellFormed(Ty)`** -- true if the type is well formed
* **`HasImpl(P0: Trait<P1..Pn>)`** -- true if an impl exists and its where clauses are satisfied
* **`Implemented(P0: Trait<P1..Pn>)`** -- true if an impl exists and all where clauses declared on the trait are satisfied

---

# Clauses from a struct

```rust
struct BinaryTree<T: Ord> { ... }
```

generates the clause

```rust
forall<T> {
    WellFormed(BinaryTree<T>) :-
        WellFormed(T),
        Implemented(T: Ord),
        Implemented(T: Sized)
}
```

---

# Clauses from an impl

```rust
impl<T: Ord> Ord for BinaryTree<T> { ... }
```

generates the clause

```rust
forall<T> {
    HasImpl(BinaryTree<T>: Ord) :-
        WellFormed(BinaryTree<T>),
        Implemented(T: Ord),
        Implemented(T: Sized)
}
```

---

# Clauses from a trait

```rust
trait Ord: Eq¹ { ... }
```

generates the clause

```rust
forall<T> {
    Implemented(T: Ord) :-
        HasImpl(T: Ord),
        Implemented(T: Eq)
}
```

.footnote[¹ I've simplified Rust's trait hierachy here.]

---

# Putting it all together

Given this program:

```rust
trait Ord: Eq { ... }
trait Eq { ... }

struct BinaryTree<T: Ord> { ... }
impl<T: Ord> Ord for BinaryTree<T> { ... }
impl<T: Ord> Eq for BinaryTree<T> { ... }

impl Ord for u32 { ... }
impl Eq for u32 { .. }
```

...can we prove `Implemented(BinaryTree<u32>: Ord)`?

---

# Putting it all together

* `Implemented(BinaryTree<u32>: Ord)`

.ProveImplementedArrow[![Arrow](./images/Arrow.png)]

---------

--

Use clause 

```
forall<T> {
    Implemented(T: Ord) :-
        HasImpl(T: Ord),
        Implemented(T: Eq)
}
```

from 

```rust
trait Ord: Eq { ... }
```

---

# Putting it all together

* `Implemented(BinaryTree<u32>: Ord)` holds if...
    * `HasImpl(BinaryTree<u32>: Ord)`
    * `Implemented(BinaryTree<u32>: Eq)`

.ProveHasImplArrow[![Arrow](./images/Arrow.png)]

---------

--

use clause 

```
forall<T> {
    HasImpl(BinaryTree<T>: Ord) :-
        WellFormed(BinaryTree<T>),
        Implemented(T: Ord),
        Implemented(T: Sized)
}
```

from `impl<T: Ord> Ord for BinaryTree<T> { ... }`

---

# Putting it all together

* `Implemented(BinaryTree<u32>: Ord)` holds if...
    * `HasImpl(BinaryTree<u32>: Ord)` holds if...
        * `WellFormed(BinaryTree<u32>)`
        * `Implemented(u32: Ord)`
        * `Implemented(u32: Sized)`
    * `Implemented(BinaryTree<u32>: Eq)`

.ProveWellFormedArrow[![Arrow](./images/Arrow.png)]

---------

--

Use clause from `struct BinaryTree<T: Ord>`...

```
forall<T> { WellFormed(BinaryTree<T>) :- WellFormed(T), 
                                         Implemented(T: Ord),
                                         Implemented(T: Sized) }
```

---

# Putting it all together

* `Implemented(BinaryTree<u32>: Ord)` holds if...
    * `HasImpl(BinaryTree<u32>: Ord)` holds if...
        * `WellFormed(BinaryTree<u32>)` holds if...
            * `WellFormed(u32)`
            * `Implemented(u32: Ord)`
            * `Implemented(u32: Sized)`
        * `Implemented(u32: Ord)`
        * `Implemented(u32: Sized)`
    * `Implemented(BinaryTree<u32>: Eq)`

---

# Putting it all together

* `Implemented(BinaryTree<u32>: Ord)` holds if...
    * `HasImpl(BinaryTree<u32>: Ord)` holds if...
        * `WellFormed(BinaryTree<u32>)` holds if...
            * `WellFormed(u32)` ✅ (builtin rule)
            * `Implemented(u32: Ord)` ✅ (due to impl)
            * `Implemented(u32: Sized)` ✅ (builtin rule)
        * `Implemented(u32: Ord)`  ✅ (due to impl)
        * `Implemented(u32: Sized)`  ✅ (due to impl)
    * `Implemented(BinaryTree<u32>: Eq)` ...✅ (similar to above)

---

# Key takeaways

* Traits define `Implemented` in terms of `HasImpl`
* Impls define `HasImpl` in terms of `WellFormed`, `Implemented`
* Structs define `WellFormed` in terms of `WellFormed`, `Implemented`
* Builtin rules for `Sized`, scalars

---

# Rust declarations

Given a `DeclProgram`, two key tasks:

* ~~Create **program clauses** that define what is *true*~~
* Create **wellformedness goals** that define what a legal Rust program is

---

# Wellformedness goals for a struct

```rust
struct BinaryTree<T: Ord> {
    field1: Type1,
    field2: Type2,
}
```

generates the **goal**

```rust
forall<T> {
    if Implemented(T: Ord), WellFormed(T) {
        WellFormed(Type1),
        WellFormed(Type2),
    }
}
```

???

* This is a goal we have to prove
* We can assume where clauses hold, `T` is well-formed
* Have to show that field types are well-formed

---

# Wellformedness goals for an impl

```rust
impl<T: Ord> Ord for BinaryTree<T> { ... }
```

generates the **goal**

```rust
forall<T> {
    if Implemented(T: Ord), WellFormed(BinaryTree<T>) {
        Implemented(BinaryTree<T>: Ord)
    }
}
```

???

* Impl is well-formed if the trait is implemented
* In other words, if all where clauses on trait are satisfied

---

# Modeling Rust of the future

Use Formality to model future Rust features

* Understand impact of a feature (how fundamental is it)
* Identify and resolve conflicts before they happen

---

# Perfect derive

```rust
#[derive(Clone)]
struct List<T> {
    data: Rc<T>,
    next: Option<Rc<List<T>>>
}
```

gives

```rust
impl<T: Clone> Clone for List<T> { ... }
//    ^^^^^^^ not really necessary
```

---

# Perfect derive

```rust
#[derive(Clone)]
struct List<T> {
    data: Rc<T>,
    next: Option<Rc<List<T>>>
}
```

better expansion

```rust
impl<T> Clone for List<T>
where
    Rc<T>: Clone,
    Option<Rc<List<T>>>: Clone,
```

---

# Perfect derive

Problem: cycles!

Solution: all traits coinductive, not just auto traits like `Send`.

---

# Implied bounds

```rust
struct BinaryTree<T: Ord> { ... }
impl<T: Ord> Ord for BinaryTree<T> { ... }
//    ^^^^^ Why do I have to write this?
```

---

# Implied bounds

```rust
struct BinaryTree<T: Ord> { ... }
impl<T> Ord for BinaryTree<T> { ... }
```

would desugar to

```rust
forall<T> {
    if WellFormed(BinaryTree<T>) {
    // ^^^^^^^^^^^^^^^^^^^^^^^^^ infer `T: Ord` from this?
        Implemented(T: Ord)
    }
}
```

---

# Implied bounds

Most obvious idea: Add program clauses from struct definition.

```rust
struct BinaryTree<T: Ord> { }
```

add "if and only if" rules like so:

```rust
forall<T> {
    WellFormed(BinaryTree<T>) :- WellFormed(T), 
                                 Implemented(T: Ord),
                                 Implemented(T: Sized)

    Implemented(T: Ord) :- WellFormed(BinaryTree<T>)
}
```
---

# Implied bounds

In fact, we already have similar rules:

```rust
trait Ord: Eq { }
```

you can write `T: Ord` and infer that `T: Eq`...

```rust
forall<T> {
    Implemented(T: Ord) :- HasImpl(T: Ord), Implemented(T: Eq)

    Implemented(T: Eq) :- Implemented(T: Ord)
}
```

---

# Perfect derive + implied bounds

Problem: unsound when combined with coinduction! 👹

---

# Perfect derive + implied bounds

Given

```rust
struct X { }
impl Ord for X { }
```

can prove `Implemented(X: Eq)` by using the clause 

```rust
forall<T> { Implemented(T: Eq) :- Implemented(T: Ord) }
```

* `Implemented(X: Eq)` (by clause above) holds if
    * `Implemented(X: Ord)` (by clause from trait) holds if
        * `HasImpl(X: Ord)` ✅
        * `Implemented(X: Eq)` ✅ -- cycle is ok!

---

# Clauses + Invariants: How Formality models it

The logic layer actually has

* clauses (rules that define what is true)
* invariants (assertions of form `P ⇒ Q` that should hold)

---

# Clauses + Invariants: How Formality models it

```rust
trait Ord: Eq { }
```

gives the **clause**

```rust
forall<T> {
    Implemented(T: Ord) :- HasImpl(T: Ord), Implemented(T: Eq)
}
```

and the **invariant**

```rust
Implemented(T: Ord) => Implemented(T: Eq)
```

---

# Clauses + Invariants: How Formality models it

Invariants are only used to elaborate **hypotheses** when proving an implication like `if H { G }`:

```rust
if Implemented(T: Ord) {
    G
}
``` 

is equivalent to

```rust
if Implemented(T: Ord) ∧ Implemented(T: Eq) {
    G
}
```

---

# Sound?

Have to show

* Logic layer: proof rule is sound assuming invariants are accurate
* Decl layer: invariants generated are accurate

---

# Modeling Rust of the future: summary

Can implement *perfect derive* and *implied bounds* by...

* Extending logic layer to include invariants + coinduction

Matches our intuition:

* Syntax doesn't change at all
* But a fairly fundamental change nonetheless!

---

# Conclusion

Presented **a-mir-formality**:

* Layered modeling of Rust semantics:
    * Core logic that is independent of Rust
    * Semantics of Rust expressed as logical predicates
    * Currently targeting MIR type system + operational semantics
* Formality can be used to:
    * Describe Rust type system with precision
    * Explore and understand proposed Rust extensions
    * Provide vocabulary for compiler, tooling to communicate
