# NL Specification — Coherence Tracker

This document lists all known inconsistencies, errors, and gaps across the NL specification
documents (`specs.md`, `stdlib.md`, `compiler.md`, `vm.md`). Each item has a checkbox to track
resolution. When an item is fixed, check the box and note the date or version.

Sources: analysis from `docs_internal/coherence_2.md` and `docs_internal/documentation_of_virtual_machine.md`,
plus a full re-audit of `docs/` performed on 2026-03-01.

---

## Summary

| Category | Count | Description |
|----------|-------|-------------|
| [I. Syntax violations](#i-syntax-violations) | 3 | Direct violations of spec rules in spec text itself |
| [II. Nonexistent references](#ii-nonexistent-references) | 3 | Methods, types, or functions used but never defined |
| [III. Cross-document contradictions](#iii-cross-document-contradictions) | 2 | specs.md vs compiler.md, or stdlib.md vs specs.md |
| [IV. Incorrect or misleading examples](#iv-incorrect-or-misleading-examples) | 4 | Code examples that would not compile or behave as shown |
| [V. Under-specified elements](#v-under-specified-elements) | 5 | Features used or referenced but incompletely defined |
| [VI. Missing features](#vi-missing-features) | 5 | Identified gaps (not blocking but tracked) |

---

## I. Syntax violations

The spec explicitly states: *"The suffix `?` is not accepted; the language requires the explicit union with `null`."*
([specs.md § Union types](specs.md#union-types-and-explicit-nullable)). Yet `?` is used in several places.

### I-1. `Self?` in enum tryFrom signature

- [ ] **specs.md:1217** — `static Self? tryFrom(string value)` → `static Self|null tryFrom(string value)`

### I-2. `string?` in FileHandle.readLine

- [ ] **stdlib.md:438** — `string? readLine() throws IOException` → `string|null readLine() throws IOException`
- [ ] **stdlib.md:450** — `string? line;` in the FileHandle example → `string|null line;`

### I-3. `string[]?` in HttpResponse.headers

- [ ] **stdlib.md:104** — `string[]` followed by `?` in the type column of `HttpResponse.headers`. Should be
  `string[]|null` if nullable, or documented differently if the field itself is optional.

---

## II. Nonexistent references

### II-1. `system.Out.writeLine` does not exist

- [ ] **stdlib.md:452** — `system.Out.writeLine(line)` in the FileHandle example. `system.Out` only defines
  `print` and `println` (stdlib.md:143-144). Should be `system.Out.println(line)`.

### II-2. `char` type referenced but never defined

- [ ] **specs.md:729** — "Scalar types (`int`, `float`, `bool`, `char`, `byte`, etc.)" in the
  Parameter passing semantics section. `char` is not listed in native types (specs.md:199) and has no
  definition anywhere. Remove `char` from the list or define it as a native type.

### II-3. `assert()` used but never defined

- [ ] **specs.md:1227-1233** — `assert(status == Status.NotFound)` and similar calls in the enum example.
  `assert` is not a keyword, not in the stdlib, and not defined anywhere. Either:
  - Add `assert` as a built-in statement/keyword, or
  - Replace the example with explicit `if` checks or remove it.

---

## III. Cross-document contradictions

### III-1. Exception inheritance rules — specs.md vs compiler.md

This is the most significant inconsistency. The two documents define **different rules**.

**specs.md:2017** says:
> "the overriding method must declare at least the same checked exceptions (or their subclasses).
> It cannot declare fewer checked exceptions than the base method."

**compiler.md:235-242** says:
> "For each exception type E in the parent's throws clause, the child's throws clause must include
> E **or a subclass of E**."
> "The child may not introduce **new** checked exception types that are not subtypes of exceptions
> declared by the parent."

These are different rules. Consider: parent declares `throws Exception, IOException`.

| Child declaration | specs.md | compiler.md (Liskov) |
|-------------------|----------|----------------------|
| `throws IOException` | **Error** (fewer exceptions) | **Valid** (IOException covers both Exception and IOException since it extends Exception) |
| `throws FileNotFoundException, IOException` | OK (example at specs.md:2033) | Valid |

Additionally, specs.md contradicts **itself**:
- Line 2033 marks `throws FileNotFoundException, IOException` as OK (replacing `Exception` with a sub-sub-class).
- Line 2038-2041 marks `throws IOException` as Error ("Missing Exception"). But by the same logic,
  `IOException` is a subclass of `Exception` and should be valid.

- [ ] **Resolution needed:** adopt the compiler.md rule (Liskov-compatible, matches Java) and rewrite the
  specs.md section + examples to match. Update compiler.md error E016/E017 descriptions if needed.

### III-2. Array built-in methods summary incomplete in stdlib.md

- [ ] **stdlib.md:133** — The summary paragraph lists only `length()`, `slice()`, `map()`, `filter()`.
  Missing: **`forEach()`**, **`sort()`**, **`find()`**.
  These three methods are correctly defined in specs.md:245-247 and referenced in vm.md:943-947.
  The stdlib.md summary must be updated to match.

---

## IV. Incorrect or misleading examples

### IV-1. `MyException` constructor missing `super(message)`

- [ ] **specs.md:1903-1908** — The custom exception example:
  ```nl
  class readonly MyException extends Exception {
      public int customField;
      public construct(string message, int customField) {
          this.customField = customField;
      }
  }
  ```
  The constructor never calls `super(message)`, so `Exception.message` will not be initialized.
  Add `super(message);` as the first statement.

### IV-2. Entry point example — wrong argc check

- [ ] **specs.md:2100-2107** — The example checks `if (argc < 3)` but then accesses `args[1]`, `args[2]`,
  and `args[3]`. Since `args[0]` is the program name, 4 elements are needed (argc >= 4).
  The check should be `if (argc < 4)`.

### IV-3. Non-nullable parameter compared to null

- [ ] **specs.md:1366-1370** — Anonymous function example: `if (input == null)` where `input` is `string`
  (non-nullable). The condition is always false. The parameter type should be `string|null input` to make
  the example meaningful.
- [ ] **specs.md:1390-1395** — Same issue: `if (s == null)` where `s` is `string`.

### IV-4. Enum descriptions in French

- [ ] **specs.md:1220-1221** — The `from()` / `tryFrom()` descriptions are in French while the rest of
  specs.md is in English. Translate to English for consistency.

---

## V. Under-specified elements

### V-1. Exception type for `enum.from()` not specified

- [ ] **specs.md:1220** — Says "lève une exception" (throws an exception) but does not specify which
  exception class. Should be a specific `RuntimeException` subclass (e.g. `IllegalArgumentException`)
  so that user code and the VM know what to catch.

### V-2. Keywords declared but never defined

The following keywords appear in the keywords table (specs.md:155-166) but have no defined semantics.
vm.md:989-1001 has provisional rules (reject at compile time) but the language-level behavior is unspecified.

- [ ] **`virtual`** — Which methods are virtual by default? Is explicit `virtual` required?
- [ ] **`abstract`** — Abstract classes and methods: declaration syntax, rules, interaction with constructors.
- [ ] **`final`** — On a class (prevents inheritance)? On a method (prevents override)? Both?
- [ ] **`clone`** — Deep or shallow? Interface requirement? Syntax?
- [ ] **`delete`** — Manual destruction? Interaction with GC and destructors?

### V-3. Map iteration not defined

- [ ] **stdlib.md:375** — "Iteration over keys or entries is implementation-defined." This leaves `system.Map`
  unusable for any logic that requires iterating entries. Needs concrete API: `keys()` returning `K[]`,
  `values()` returning `V[]`, `entries()`, or for-each support.

### V-4. Multidimensional array creation not specified

- [ ] **specs.md:486,495** — `typedef int[][] Matrix` and `new int[size][size]` are used in the typedef example,
  but the spec only defines `new T[n]` (1D fixed size) and `new T[]{ ... }` (1D initializer list).
  The syntax `new int[size][size]` for 2D arrays is never defined. Either:
  - Define multidimensional array creation syntax, or
  - Replace the example with nested 1D arrays.

### V-5. Naming convention inconsistency — lowercase modules

- [ ] **stdlib.md** — `system.env`, `system.ps`, `system.process` use lowercase names while all other
  stdlib entities use PascalCase (`system.Out`, `system.Int`, `system.io.File`, etc.). Decision needed:
  - If these are **namespaces** containing static functions (not classes), document this distinction.
  - If these are **classes**, rename to `system.Env`, `system.Ps`, `system.Process`.

---

## VI. Missing features (tracked, not blocking)

These are identified gaps — not bugs or inconsistencies, but features whose absence has been noted. They are
tracked here for completeness. Some may be intentional design choices.

### VI-1. `do-while` loop

- [ ] No `do { ... } while (condition);` loop. Not in keywords, not in control structures. Common in most
  C-family languages.

### VI-2. try-with-resources / RAII

- [ ] No mechanism for automatic resource cleanup at scope exit (like Java's `try-with-resources`, C#'s
  `using`, or C++ RAII). The only cleanup mechanism is the destructor, whose timing is
  implementation-defined (vm.md:983-985).

### VI-3. `equals()` / `hashCode()` convention

- [ ] No convention for structural equality of objects. `==` on references compares identity (vm.md:442).
  No standard mechanism for using objects as `system.Map` keys with value-based equality.

### VI-4. Bounded generics

- [ ] No syntax like `template <type T extends Comparable>` to constrain type parameters. Verification is
  purely structural at instantiation (compiler.md:130-138). This limits compile-time error messages and
  documentation of template contracts.

### VI-5. Multiple return values / tuples

- [ ] No tuple type or multiple return values. Workarounds: return a custom class, or use `ref` parameters.

---

## Previously resolved (for reference)

The following issues from `docs_internal/coherence_2.md` have been verified as resolved:

| Issue | Resolution |
|-------|-----------|
| `self` vs `Self` ambiguity | specs.md § Self and type keywords — `Self` (type alias, mutation), `this` (instance), `type` (new instance) |
| `undefined` status | Removed from type system; reserved keyword only; definite assignment analysis + default values |
| `swap` / parameter passing semantics | specs.md § Parameter passing semantics — value, reference-value, `ref`, `const ref` |
| `enum.from()` missing `static` | Corrected to `static Self from(...)` |
| `FileHandle` with no read/write | stdlib.md § FileHandle — `read`, `readLine`, `write`, `flush` added |
| No string instance methods | stdlib.md § Instance methods on string — `length`, `charAt`, `substring`, `indexOf`, etc. |
| No int/float to string conversion | `system.Int.toString`, `system.Float.toString`, `system.Bool.toString` |
| No `continue` in loops | specs.md § Loops — `continue` documented |
| String concatenation rules | specs.md § String concatenation — conversion rules defined |
| Cast / type conversions | specs.md § Type conversions and casting — full section with rules |
