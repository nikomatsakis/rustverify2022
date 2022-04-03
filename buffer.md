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
* a scalar `i32` would just be `i32<>`
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

