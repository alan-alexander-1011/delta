# Delta / "What?" Language Specification

**Paradigm:** Multi-paradigm (Procedural + OOP + Functional)  
**Implementation Target:** C++ + LLVM  
**Canonical Syntax:** C++-like (aliases are preprocessor sugar only)

---

## Semantics

### Core Principle

Semantics are mutable and may be redefined per scope. Unless explicitly overridden, default semantics apply.

---

### [1] Memory Validity

**Default:**

- Invalid memory access results in undefined behavior

**Definition:**

Invalid memory includes:
- Dangling references
- References to moved-from values
- References to destroyed objects

---

**Overrides (scoped):**

- `memory = unsafe` → invalid access results in undefined behavior (default)
- `memory = null` → invalid references evaluate to null
- `memory = trap` → invalid access triggers runtime trap

---

**Rules:**

- Move operations do NOT update existing references
- References to moved or destroyed values are considered invalid
- Behavior of invalid references is determined solely by the current memory mode

---

**Example:**

```cpp
a = 10;
r = &a;
b = move(a);

// r is now invalid

// behavior depends on memory mode:
// unsafe → UB
// null   → r == null
// trap   → runtime error
```

---

**Notes:**

- This model unifies dangling, move, and destruction behavior
- Eliminates conflicting semantic combinations
- Simplifies reasoning about memory safety

---

### [2] Mutability and References

**Default:**

- Follows C++-like semantics
- References may be mutable or const
- Mutation through non-const reference is allowed

**Overrides:**

- `mutability = cxx` → default behavior
- `mutability = strict` → at most one mutable reference per object at a time
- `mutability = const_only` → all references are read-only

**Notes:**

- Mutability only controls write permissions
- It does NOT control aliasing or concurrency behavior
- When `borrow_check = on`, the borrow checker takes precedence over mutability enforcement. Strict mutability only has independent meaning if `borrow_check = off`

---

### [3] Aliasing

**Default:**

- Unrestricted aliasing (multiple references to same memory allowed)

**Overrides:**

- `aliasing = free` → unrestricted aliasing (default)
- `aliasing = restricted` → compiler enforces limited aliasing rules

**Notes:**

- Aliasing controls how many references may exist
- It does NOT control whether references can mutate
- Aliasing rules affect optimization and memory behavior

**Interaction with Mutability:**

Mutability and aliasing rules combine to create write/access patterns:

| mutability | aliasing | behavior |
|------------|----------|----------|
| `cxx` | `free` | Standard C++: multiple mutable refs possible (UB if used concurrently) |
| `cxx` | `restricted` | Compiler enforces: if one mutable ref exists, no other refs allowed |
| `strict` | `free` | At most one mutable ref; multiple const refs allowed |
| `strict` | `restricted` | Most restrictive: one mutable OR many const refs, not mixed |
| `const_only` | `free` | All refs read-only regardless of aliasing |
| `const_only` | `restricted` | All refs read-only, no aliasing benefits |

**Resolution order:** Apply borrow checking first, then apply mutability constraints (write permissions), then apply aliasing constraints (number of refs allowed).

---

### [4] Type Interaction

**Default:**

- Implicit conversions allowed where safe
- Numeric promotion follows standard rules

**Overrides:**

- `implicit_casts = off` → all casts must be explicit
- `implicit_casts = safe` → only non-lossy conversions allowed (default)
- `implicit_casts = full` → any conversion allowed

**Notes:**

- `implicit_casts` semantic overrides are bounded by the active type strictness tier
- Semantic overrides can only loosen casting within what the current strictness tier permits, never beyond it
- See Type System §7 for strictness tier definitions

---

### [5] Error Handling

**Default:**

- Errors are not automatically handled

**Overrides:**

- `errors = ignore` → no checks performed
- `errors = trap` → runtime trap on error
- `errors = propagate` → errors must be handled or propagated

**Rules:**

- Functions using the `!` postfix must return `Result`.
- Error handling is checked and enforced in compile time.
- Calling a non-Result function inside an `errors = propagate` scope with the `!` postfix is illegal.
- The `!` postfix is illegal outside of an `errors = propagate` scope.

**Propagation syntax:**

Functions that may fail return `Result<T, E>`. The `!` postfix operator either unwraps the value or propagates the error up the call stack. 

```cpp
Result<int, Error> foo() {
    int x = might_fail()!; // propagates if error
    return x;
}
```

---

### [6] Concurrency Safety

**Default:**

- No guarantees for data races

**Overrides:**

- `concurrency = unsafe` → no checks (default)
- `concurrency = checked` → runtime detection of race conditions
- `concurrency = restricted` → compile-time restrictions on concurrent mutation

**Notes:**

- Concurrency rules only apply across threads
- Data races may still occur unless restricted mode is enabled

---

### [7] Scope Rules

- Semantics may be redefined at any scope level
- Inner scope overrides outer scope
- Semantics revert after scope ends

**Example:**

```cpp
#semantics(memory = trap) {
    #semantics(memory = null) { } // inner rules apply
} // outer rules restored
```

---

## Type System

### Core Principle

The type system is primarily static with partial inference and explicit ownership. Certain behaviors may be modified per scope under restricted rules.

---

### [1] Typing Model

**Default:**

- Hybrid typing system
- Compile-time type checking is always enforced
- Runtime type checking is optional (disabled by default); it covers type checking steps that compile-time type checking doesn't cover, but enabling kills performance

**Notes:**

- All code must pass compile-time checks (like C/C++ and Rust)
- Runtime checks may be enabled per scope or configuration

---

### [2] Type Inference

**Default:**

- Partial type inference
- Compiler infers types where unambiguous
- Full inference is only applied in functional/lambda contexts

**Restrictions:**

- Inference behavior is fixed and cannot be modified by user

---

### [3] Mutability

**Default:**

- Follows C++-like const semantics
- `const` means no mutation allowed under any condition

**Restrictions:**

- Mutability rules are fixed and cannot be modified

---

### [4] Ownership

**Default:**

- Ownership is encoded in types

**Forms:**

- `T` → owned value
- `&T` → reference
- `&mut T` → mutable reference

**Notes:**

- Ownership semantics are fixed and cannot be redefined
- References do not imply lifetime guarantees → references can dangle unless memory mode says otherwise
- References may evaluate to null depending on memory mode

**Borrow Checking:**

Compile-time borrow checking enforces reference validity:

- Default: borrow checking enabled (like Rust, simplified)
- One mutable reference OR many immutable references at a time (scoped to borrow lifetime)
- References are valid for their lexical scope unless moved/destroyed
- Compiler tracks ref validity; violations caught at compile-time

**Overrides: (scoped)**

- `borrow_check = on` → compile-time borrow checking enabled (default)
- `borrow_check = off` → no borrow checking; references can dangle

**Note:**

- Memory mode controls what happens on invalid access
- Borrow check controls whether invalid access is caught at compile-time

**Validity Rules:**

- Mutable refs exclude all other refs (mutable or immutable)
- Immutable refs may coexist with other immutable refs
- Moving a value invalidates all outstanding references to it
- Destroying a value invalidates all outstanding references to it

---

### [5] Generics / Templates

**Default:**

- Generics are supported with a fixed core model similar to C++-style templates
- Instantiation and type identity rules are defined by the language and cannot be modified

---

**Allowed customization (scoped):**

Users may extend and restrict generic behavior through:
- Additional constraints on type parameters
- Specialization rules
- Inference guidance within generic contexts

---

**Specialization Ranking System:**

Specializations follow a ranking from most to least specific. When multiple specializations match a call, the compiler picks the highest rank.

**Ranking (Least to Most Specific):**

1. **Unconstrained generic**
   `generic<T>`

2. **Single constraint**
   `generic<T where T: Numeric>`

3. **Multiple constraints (More constraints = Higher priority)**
   `generic<T where T: Numeric, T: Comparable>`

4. **Nested/Complex constraints**
   `generic<U where U: Container<T where T: Numeric>>`

5. **Concrete type (with or without constraints)**
   `generic<int>`
   `generic<int where int: Numeric, int: Comparable>`

**Comparison Rules:**

- Concrete types always beat generic types
- More constraints beat fewer constraints
- Nested constraints beat flat constraints
- A specialization with all constraints satisfied beats one with only partial constraints satisfied

**Examples:**

**Ranking Example:**

```cpp
generic<T> impl() { }                                  // Rank 1
generic<T where T: Numeric> impl() { }                 // Rank 2
generic<T where T: Numeric, T: Comparable> impl() { }  // Rank 3
generic<int> impl() { }                                // Rank 5

impl<int>();           // Picks Rank 5 (concrete)
impl<float>();         // Picks Rank 2 (float is Numeric)
impl<string>();        // Picks Rank 1 (fallback, no constraints match)
```

**Constraint Syntax:**

```
// basic constraint
generic<T where T: Numeric> { }

// multiple constraints (AND)
generic<T where T: Numeric, T: Comparable> { }

// trait bounds
generic<T where T: has_method(operator+)> { }
```

---

**Restrictions:**

Users CANNOT:
- Redefine the core instantiation model
- Change how type parameters (`T`) are interpreted
- Alter type identity or compatibility rules
- Switch between compile-time and runtime generic resolution

---

**Notes:**

- Generics remain consistent across scopes in their core behavior
- Customization only affects validation and constraint checking
- This ensures interoperability and prevents incompatible generic systems

---

### [6] Type Structure (OOP / Traits)

**Default:**

- Hybrid model combining Rust-like traits and C++-like classes

**Rules:**

- `object` types → support OOP features (methods, inheritance if defined)
- `class` types → behave as structured types (no implicit OOP)

**Restrictions:**

- Type structure rules are fixed and cannot be modified

---

### [7] Type Strictness

**Default:**

- Three levels: `loose`, `semi-strict`, `strict`

**Rules:**

`loose`:
- Accepts any type
- Implicit casts allowed freely (compiler-defined)

`semi-strict`:
- Accepts semi-strict and strict types
- Rejects loose types unless explicitly forced via unsafe flag

`strict`:
- Accepts only strict types
- No implicit casts unless explicitly forced via unsafe flag

**Notes:**

- Demotion rules apply based on scope configuration
- Type strictness always wins over `implicit_casts` semantic overrides
- Semantic overrides cannot exceed the ceiling set by the active strictness tier
- Precedence rule: check strictness tier first, THEN apply implicit_casts rules within that boundary

**Precedence Rules:**

- `loose` tier + `implicit_casts=off` → only explicit casts allowed
- `semi-strict` tier + `implicit_casts=full` → still rejects loose types, allows semi-strict↔strict conversions implicitly
- `strict` tier + `implicit_casts=safe` → only non-lossy conversions, strict types only

**Decision tree:** Evaluate strictness tier constraints first. If the conversion passes strictness, then apply implicit_casts rules. If strictness rejects it, implicit_casts cannot override.

---

### [8] Nullability

**Default:**

- Types are nullable by default
- Explicit syntax: `T?` → explicitly nullable

**Rules:**

- Variables may be implicitly nullable
- In contexts with guaranteed initialization (e.g. global scope), values are non-null by default
- When a value is null and `memory = trap` is active, a runtime exception is raised

**Restrictions:**

- Nullability behavior is fixed and cannot be modified

---

### [9] Type System Mutability & Customization Boundaries

**Core Principle:**

The type system has three categories of features: immutable core rules, customizable semantic overrides, and partially customizable extensions.

**IMMUTABLE (cannot be overridden):**

These are fixed language rules that maintain core semantics:
- Ownership forms: `T` (owned), `&T` (immutable ref), `&mut T` (mutable ref)
- Type inference model
- Generics instantiation model (core behavior)
- Nullability semantics
- Compile-time type checking (always enforced)

**CUSTOMIZABLE PER SCOPE (semantic overrides):**

These may be modified within a scope and revert after:
- Memory validity behavior: `memory = unsafe | null | trap`
- Type strictness tier: `loose | semi-strict | strict`
- Implicit cast rules: `implicit_casts = off | safe | full` (bounded by strictness)
- Error handling: `errors = ignore | trap | propagate`
- Concurrency checks: `concurrency = unsafe | checked | restricted`
- Runtime type checking: `runtime_checks = on | off`
- Borrow checking: `borrow_check = on | off`

**PARTIALLY CUSTOMIZABLE:**

These allow controlled extension without breaking core semantics:
- Generic constraints: add restrictions via `where` clauses (cannot remove core instantiation)
- Operator overloading: customize for user-defined types only (built-ins remain fixed)
- Mutability enforcement models: choose `cxx | strict | const_only` (cannot redefine what const means)
- Aliasing enforcement models: choose `free | restricted` (cannot redefine ownership)

**Scope rules:**

- Inner scope overrides outer scope
- Changes are limited to customizable and partially customizable features
- Immutable features remain constant across all scopes
- Reverting to outer scope rules happens automatically when scope ends

---

### [10] Operator Overloading

**Default:**

- User-defined types (`class`, `object`) may overload operators

**Supported operators include (but not limited to):**

`=`, `+=`, `-=`, `*=`, `/=`, `==`, `!=`, `<`, `>`, `<=`, `>=`

---

**Rules:**

- Operator overloading is only allowed for user-defined types
- Built-in types cannot have their operator behavior modified
- Operator definitions must be explicitly declared

**Example:**

```cpp
class Vec {
    x: float
    y: float

    operator +=(other: Vec) {
        this.x += other.x
        this.y += other.y
    }
}
```

---

**Assignment (`=`):**

- May be overloaded for custom types
- Must define behavior for copying or moving explicitly

```cpp
operator =(other: T) {
    // define copy or move behavior
}
```

---

**Constraints:**

- Operator behavior must remain consistent with type safety rules
- Overloading cannot violate ownership semantics
- Overloading cannot redefine core language behavior

---

**Optional restrictions (implementation-defined):**

- Operators may be required to follow expected semantics (e.g. `+` should not perform unrelated operations)

---

**Notes:**

- Operator overloading is resolved at compile-time
- No runtime operator dispatch unless explicitly enabled

---

## Syntax and Style

### Core Principle

- Syntax is C++-like; this is the canonical form
- Keyword aliases are allowed for accessibility across different programming backgrounds
- All aliases are preprocessor sugar that maps to canonical C++-like form before parsing
- All aliases map to a single canonical internal form
- Blocks always use `{}`

---

### [1] Canonical Form

- The preferred syntax is compact and C++-like
- Canonical syntax is the source of truth for parsing, tooling, and style
- Aliases are alternate spellings of the same construct, resolved before parsing — not separate constructs
- Mixing aliases is allowed but discouraged for readability

**Example:**

```cpp
int add(int a, int b) {
    return a + b;
}
```

---

### [2] Keyword Aliases

- Common keywords may have aliases globally
- Aliases must remain recognizable and must not introduce ambiguity
- Aliases behave exactly like their canonical counterparts

**Examples:**

- `if` ↔ `when`
- `fn` ↔ `function`
- `int` ↔ `i32`
- `float` ↔ `f64`

---

### [3] Function Declaration

- Canonical form: `return_type name(parameters)`

**Example:**

```cpp
int compute(int x) {
    return x + 1;
}
```

- Alias-based forms must still declare the return type explicitly

**Examples:**

```cpp
fn compute(x: i32) : i32
function compute(x) -> int
```

- All alias forms are normalized to canonical C++-like form via preprocessor pass
- Mixing function declaration styles within the same module is allowed, but consistency is recommended

---

### [4] Control Flow

- Canonical control flow uses C++-like block syntax
- Aliases are allowed for familiar constructs

**Examples:**

- `if / else` (`when` is an alias)
- `switch`
- `for`
- `while`

---

### [5] Type Names

- Canonical built-in type names are preferred in standard code
- Aliases are fully interchangeable

**Examples:**

- `int` ↔ `i32`
- `float` ↔ `f64`
- `bool`

---

### [6] Block Structure

- All blocks must use `{}`

**Example:**

```cpp
if (x) {
    do_something();
}
```

- Indentation is implementation-defined but must be consistent

---

### [7] Semantics Overrides

- Semantics overrides must be explicit and scoped
- Excessive nesting is discouraged

**Example:**

```cpp
#semantics(memory = trap) {
    ...
}
```

---

### [8] Style Consistency

- One dominant style per module is preferred
- Mixing aliases is allowed, but readability has higher priority than novelty
- Tooling may enforce a canonical subset

---

### [9] Readability Rule

- Clarity is preferred over brevity
- Aliases exist to help users from other languages, not replace canonical style
- Code should remain understandable without decoding multiple alternate forms
