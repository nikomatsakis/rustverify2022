class: center
name: title
count: false

# A MIR Formality

.me[.grey[*by* **Nicholas Matsakis**]]
.citation[`https://github.com/nikomatsakis/rustverify2022`]

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

Nitty gritty:

I'd love to sit down with people and go over the specifics of what I've done so far. I'm here till Thursday.

Bigger picture:

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

.footnote[¹ "Semantics Engineering with PLT Redex" by Felleisen et al. ]

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

* 

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

# ~~Core logic~~

# Rust types

---

# Rust types

Defines:

* Grammar of types, lifetimes, and where clauses
* Subtyping rules

---

# Type grammar

Generalized version of Rust types:

```
Type = RigidType < Parameter ... >
     | ∀X:WhereClause. Type
     | ∃X:WhereClause. Type
     | X

RigidType = StructName
          | tuple/N
          | fn/N
          | ...

Parameter = Type
          | Lifetime
```

---

# Rigid types

Most Rust types are "rigid types":

* a `Vec<T>` would just be `Vec<T>`
* a fn pointer like `fn(u8)` would be `fn/1<u8, ()>`
* a tuple like `(T, U)` would be `tuple/2<T, U)>`

---

# ∀ types

∀ types are used for higher-ranked functions and `dyn` values.

* a type like `for<'a> fn(&'a u8)` would be `∀'a. fn/1<&'a u8, ()>`

---

# ∃ types

∃ types are used for higher-ranked functions and `dyn` values.

* a type like `dyn Write` would be `∃T:Write. T`

---

# ~~Core logic~~

# ~~Rust types~~

# Rust declarations

---

# Rust declarations


---

# Rust declarations