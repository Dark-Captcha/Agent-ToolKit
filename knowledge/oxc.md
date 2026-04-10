# OXC Reference

> **Version:** 3.0.0 | **Status:** Active | **Updated:** 2026-04-10

OXC (Oxford Compiler Collection) crate ecosystem for JavaScript/TypeScript parsing, analysis, transformation, and code generation in Rust. Organized bottom-up by dependency layer.

---

## Invariants

These apply to every layer. Violating any causes UB, panics, or silent corruption.

| #   | Rule                              | Detail                                                             |
| --- | --------------------------------- | ------------------------------------------------------------------ |
| 1   | Arena allocation                  | All AST nodes live in `Allocator`. No individual `Drop`.           |
| 2   | `clone_in`/`take_in` preserve IDs | `SymbolId`/`ReferenceId`/`ScopeId` survive. `.clone()` forbidden. |
| 3   | "Without" type safety             | Compile-time prevents child-field aliasing in traversal.           |
| 4   | Generated code is authoritative   | `*/generated/` files are auto-generated. Never edit.               |
| 5   | IDs are `NonMaxU32`               | Valid IDs are never `u32::MAX`. Stored in `Cell<Option<_>>`.       |

---

## Layer 0 — Memory

Foundation. Everything allocates here.

### Allocator

```rust
let mut allocator = Allocator::default();           // Lazy
let mut allocator = Allocator::with_capacity(1 << 20);  // Pre-allocate
allocator.reset();                                  // Reuse memory
```

**MUST** recycle with `reset()` across units of work. New allocator per file is expensive. No reset causes unbounded growth.

### Arena Types

| Type                | Creation                         |
| ------------------- | -------------------------------- |
| `Box<'a, T>`        | `Box::new_in(value, &allocator)` |
| `Vec<'a, T>`        | `Vec::new_in(&allocator)`        |
| `HashMap<'a, K, V>` | `HashMap::new_in(&allocator)`    |
| `StringBuilder<'a>` | `StringBuilder::new_in(&allocator)` |

`Sync` if contents are. **Not `Send`**.

### CloneIn / TakeIn

Both preserve semantic IDs (`SymbolId`, `ReferenceId`, `ScopeId`).

| Operation  | Effect           | Leaves Behind       | Use When               |
| ---------- | ---------------- | ------------------- | ---------------------- |
| `clone_in` | Deep arena copy  | Original intact     | Need original + copy   |
| `take_in`  | Move out of node | Dummy (NullLiteral / EmptyStatement / empty Vec) | Moving to new location |
| Direct `*` | In-place replace | N/A                 | Simple replacement     |

**MUST** replace or remove dummy after `take_in`.

```rust
self.stored.insert(id, expr.clone_in(ctx.ast.allocator));   // Clone for storage
*expr = replacement.take_in(ctx.ast.allocator);              // Take for movement

// Vector rebuild
let mut new_stmts = ctx.ast.vec();
for stmt in stmts.take_in(allocator) {
    if keep(&stmt) { new_stmts.push(stmt); }
}
*stmts = new_stmts;
```

### Address Stability

`Box<T>`: always stable. `Vec<T>` items: unstable after push/mutation.

### AllocatorPool

```rust
let pool = AllocatorPool::new(num_threads);
let guard = pool.get();  // Auto-reset on drop
```

---

## Layer 1 — Primitives

Foundational types used by every higher layer.

### Span

```rust
pub struct Span { pub start: u32, pub end: u32 }  // Byte range, 8 bytes
pub const SPAN: Span = Span::new(0, 0);            // ALL generated nodes use this
```

| Method             | Purpose                               |
| ------------------ | ------------------------------------- |
| `size()`           | `end - start`                         |
| `is_unspanned()`   | `self == SPAN` (generated node)       |
| `merge(other)`     | Smallest span containing both         |
| `source_text(src)` | Extract source substring              |

Traits: `GetSpan` on all AST nodes. `GetSpanMut` for rare location modification.

### Atom<'a>

Arena-allocated interned string. O(1) equality. Lifetime tied to allocator.

```rust
Atom::from("str")                                  // Borrows &str
Atom::from_in("str", &allocator)                   // Allocates in arena
format_atom!(&allocator, "prefix_{}", var)          // Format into arena
```

Used for: identifier names, string literal values, property names.

### CompactStr

Owned string, no lifetime. Inline storage for 16 bytes or less.

```rust
CompactStr::new("short")
atom.into_compact_str()  // Atom -> CompactStr
```

### ContentEq

```rust
expr1.content_eq(&expr2)  // Structural equality, ignores spans
```

`NaN.content_eq(NaN) == true`. `0.0.content_eq(-0.0) == false`.

### SourceType

```rust
pub struct SourceType { language: Language, module_kind: ModuleKind, variant: LanguageVariant }
```

| Constructor                 | Result                     |
| --------------------------- | -------------------------- |
| `SourceType::from_path(p)`  | Auto-detect from extension |
| `SourceType::unambiguous()` | Parser detects module kind |
| `SourceType::mjs()`         | JavaScript + Module        |
| `SourceType::cjs()`         | JavaScript + Script        |
| `SourceType::ts()`          | TypeScript + Module        |
| `SourceType::tsx()`         | TypeScript + Module + JSX  |

Queries: `is_javascript()`, `is_typescript()`, `is_module()`, `is_jsx()`, `is_strict()`. Modules are always strict mode.

---

## Layer 2 — AST

Depends on: Layer 0 (arena), Layer 1 (Span, Atom).

### Expression (~40 variants, `#[repr(C, u8)]`)

All variants boxed: `Expression::StringLiteral(Box<'a, StringLiteral<'a>>)`.

| Range | Category    | Variants                                                          |
| ----- | ----------- | ----------------------------------------------------------------- |
| 0-6   | Literals    | Boolean, Null, Numeric, String, Template, BigInt, RegExp          |
| 10    | Identifier  | `IdentifierReference`                                             |
| 20-21 | Collections | Array, Object                                                     |
| 25-27 | Functions   | Arrow, Function, Class expression                                 |
| 30-36 | Operations  | Unary, Binary, Logical, Conditional, Assignment, Sequence         |
| 40-42 | Calls       | Call, New, Import                                                 |
| 48-50 | Members     | Computed, Static, PrivateField (via `inherit_variants!`)          |

`inherit_variants!` flattens nested enums. `Expression::ComputedMemberExpression` exists directly. **`Expression::MemberExpression` does NOT exist** — use specific variants or `match_member_expression!()`.

### Statement (~30 variants)

Key: `ExpressionStatement`, `ReturnStatement`, `IfStatement`, `SwitchStatement`, `WhileStatement`, `ForStatement`, `VariableDeclaration`, `FunctionDeclaration`, `BlockStatement`, `EmptyStatement`.

### Identifiers (4 types — MUST NOT mix)

| Type                  | Purpose               | ID Field                                  |
| --------------------- | --------------------- | ----------------------------------------- |
| `IdentifierName`      | Property names        | None                                      |
| `IdentifierReference` | Variable reads/writes | `reference_id: Cell<Option<ReferenceId>>` |
| `BindingIdentifier`   | Declarations          | `symbol_id: Cell<Option<SymbolId>>`       |
| `LabelIdentifier`     | Loop/block labels     | None                                      |

### Key Structs

```rust
pub struct VariableDeclaration<'a> {
    pub span: Span,
    pub kind: VariableDeclarationKind,  // Var, Let, Const, Using
    pub declarations: Vec<'a, VariableDeclarator<'a>>,
}

pub struct Function<'a> {
    pub span: Span,
    pub id: Option<BindingIdentifier<'a>>,
    pub generator: bool,
    pub r#async: bool,
    pub params: Box<'a, FormalParameters<'a>>,
    pub body: Option<Box<'a, FunctionBody<'a>>>,
    pub scope_id: Cell<Option<ScopeId>>,
}
```

### AstBuilder (Generated)

All node creation goes through `AstBuilder` (accessed via `ctx.ast` in traversal):

```rust
// Literals
ctx.ast.expression_string_literal(SPAN, ctx.ast.atom("value"), None)
ctx.ast.expression_numeric_literal(SPAN, 42.0, None, NumberBase::Decimal)
ctx.ast.expression_boolean_literal(SPAN, true)
ctx.ast.expression_null_literal(SPAN)

// Identifiers and operations
ctx.ast.expression_identifier_reference(SPAN, ctx.ast.atom("name"))
ctx.ast.expression_binary(SPAN, left, BinaryOperator::Addition, right)
ctx.ast.expression_unary(SPAN, UnaryOperator::UnaryNegation, argument)

// Statements
ctx.ast.statement_expression(SPAN, expr)
ctx.ast.statement_return(SPAN, Some(expr))
ctx.ast.statement_block(SPAN, stmts)

// Variable declaration
ctx.ast.binding_identifier(SPAN, ctx.ast.atom("x"))
ctx.ast.variable_declarator(SPAN, VariableDeclarationKind::Const, binding, Some(init), false)

// Collections
ctx.ast.vec()       // Empty arena vec
ctx.ast.vec1(item)  // Single-element vec
```

---

## Layer 3 — Parser

Depends on: Layer 0-2.

```rust
let ret = Parser::new(&allocator, source, source_type)
    .with_options(ParseOptions {
        preserve_parens: true,  // Emit ParenthesizedExpression nodes
        ..Default::default()
    })
    .parse();
```

| Field          | Type                 | Notes                     |
| -------------- | -------------------- | ------------------------- |
| `ret.program`  | `Program<'a>`        | Always structurally valid |
| `ret.errors`   | `Vec<OxcDiagnostic>` | Check for issues          |
| `ret.panicked` | `bool`               | `true` = AST empty        |

Expression-only parsing: `Parser::new(...).parse_expression()?`.

**MUST** check `errors` and `panicked`. AST is valid even with errors (recovery parser). Use `preserve_parens: true` when transforming.

---

## Layer 4 — Semantics

Depends on: Layer 2 (AST). Produces symbol table, reference table, scope tree.

```rust
let scoping = SemanticBuilder::build(&program);
```

### Scoping (SoA layout)

Unified structure containing symbols, references, and scope tree. `Send + Sync`. Mutation needs exclusive access.

### IDs

| Type          | Represents           |
| ------------- | -------------------- |
| `SymbolId`    | Declaration (binding) |
| `ReferenceId` | Usage (reference)    |
| `ScopeId`     | Lexical scope        |

### Flags

| Type             | Key Values                                                               | Key Queries                                     |
| ---------------- | ------------------------------------------------------------------------ | ----------------------------------------------- |
| `SymbolFlags`    | `FunctionScopedVariable`, `BlockScopedVariable`, `ConstVariable`, `Function`, `Class`, `Import` | `is_const_variable()`, `is_function()`, `is_value()` |
| `ReferenceFlags` | `Read`, `Write`, `Type`                                                 | `is_read()`, `is_write()`, `is_read_only()`     |
| `ScopeFlags`     | `Top`, `Function`, `Arrow`, `StrictMode`                                | `is_var()` (hoisting boundary)                  |

### Safe ID Resolution

Direct `.reference_id()` / `.symbol_id()` **panic** if not set. Always use `.get()`:

```rust
// IdentifierReference -> SymbolId
let ref_id = ident.reference_id.get()?;
let symbol_id = scoping.get_reference(ref_id).symbol_id()?;

// BindingIdentifier -> SymbolId
let symbol_id = binding.symbol_id.get()?;

// VariableDeclarator -> SymbolId
decl.id.get_binding_identifier().and_then(|b| b.symbol_id.get())

// Function -> SymbolId
func.id.as_ref().and_then(|id| id.symbol_id.get())
```

### Queries

```rust
scoping.symbol_name(id)                              // Name
scoping.symbol_flags(id)                             // Flags
scoping.get_resolved_references(id)                  // Iterator<Reference>
scoping.get_reference(ref_id)                        // Single reference
scoping.root_unresolved_references()                 // Globals: HashMap<&str, Vec<ReferenceId>>
scoping.find_binding(scope_id, "name")               // Scope lookup
```

### Reference Counting

```rust
// Read count
scoping.get_resolved_references(id).filter(|r| r.flags().is_read()).count()

// Write count
scoping.get_resolved_references(id).filter(|r| r.flags().is_write()).count()
```

### Scoping Rebuild Rules

| Situation                | Action  | Reason                      |
| ------------------------ | ------- | --------------------------- |
| Read-only pass           | Share   | IDs still valid             |
| Collect + apply (no decls) | Share | References unchanged        |
| Add/remove declarations  | Rebuild | IDs may invalidate          |
| Rename identifiers       | Rebuild | Binding lookup changes      |
| Before unused detection  | Rebuild | Need fresh reference counts |

Rebuild: `scoping = SemanticBuilder::build(&program);`

---

## Layer 5 — Traversal

Depends on: Layer 0 (allocator), Layer 2 (AST), Layer 4 (semantics).

### Traverse Trait

```rust
impl<'a> Traverse<'a, ()> for MyPass {
    fn enter_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn enter_statements(&mut self, stmts: &mut ArenaVec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_statements(&mut self, stmts: &mut ArenaVec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
    // ... hundreds of enter_*/exit_* hooks
}
```

**Rule**: Collect in `enter_*` (top-down). Transform in `exit_*` (bottom-up, children processed).

Key hooks: `expression`, `statements`, `function`, `variable_declarator`, `call_expression`, `identifier_reference`, `binding_identifier`.

### TraverseCtx

| Access                | Purpose                 |
| --------------------- | ----------------------- |
| `ctx.ast.allocator`   | Get allocator           |
| `ctx.ast.*`           | Create new AST nodes    |
| `ctx.scoping()`       | Immutable scoping       |
| `ctx.scoping_mut()`   | Mutable scoping (rare)  |
| `ctx.parent()`        | Parent ancestor         |
| `ctx.ancestor(level)` | Nth ancestor (0=parent) |
| `ctx.ancestors()`     | Iterator parent -> root |

### Ancestor System

Each variant encodes parent type + child direction. "Without" types prevent accessing the child you descended from:

```rust
match ctx.parent() {
    Ancestor::CallExpressionCallee(call) => { /* in callee position */ }
    Ancestor::BinaryExpressionLeft(bin) => {
        bin.operator()  // OK
        bin.right()     // OK: sibling
        // bin.left()   // Does not exist: compile-time prevented
    }
}
```

### Traverse vs VisitMut

| Feature       | `Traverse` (`oxc_traverse`)  | `VisitMut` (`oxc_ast_visit`) |
| ------------- | ---------------------------- | ---------------------------- |
| Context       | Full `TraverseCtx`           | None                         |
| Ancestors     | Yes                          | No                           |
| Symbol access | Yes                          | No                           |
| Use for       | **Any semantic-aware pass**  | Simple subtree transforms    |

Use `Traverse` for any pass that needs symbol resolution, ancestor context, or scope tracking. Use `VisitMut` only for syntactic subtree operations (e.g., parameter substitution in a cloned body).

### Running a Traverse Pass

```rust
let scoping = traverse_mut(&mut my_pass, allocator, program, scoping, ());
```

---

## Layer 6 — Analysis

Depends on: Layer 2 (AST), Layer 4 (semantics).

### ECMAScript Type Coercion

| Trait        | Method           | Key Results                                           |
| ------------ | ---------------- | ----------------------------------------------------- |
| `ToBoolean`  | `to_boolean()`   | Falsey: `false`, `0`, `""`, `null`, `undefined`, `NaN` |
| `ToNumber`   | `to_number()`    | `true`->1, `""`->0, `null`->0                         |
| `ToJsString` | `to_js_string()` | `null`->"null", arrays joined with ","                |

### Constant Evaluation

```rust
expr.evaluate_value(ctx) -> Option<ConstantValue<'a>>     // May have side effects
expr.get_side_free_number_value(ctx) -> Option<f64>        // None if side effects
```

Use `get_side_free_*` variants before folding — they return `None` if the expression has side effects.

### Side-Effect Analysis

```rust
expr.may_have_side_effects(ctx) -> bool
```

No side effects: literals, identifiers, `this`, pure unary/binary/logical, function/arrow expressions. Has side effects: calls, assignments, `delete`, `new`, property access (configurable).

### Value Type

```rust
expr.value_type(ctx) -> ValueType  // Undefined, Null, Number, String, Boolean, Object, Undetermined
```

### Operators

| Category | Enum               | Key Methods                                            |
| -------- | ------------------ | ------------------------------------------------------ |
| Binary   | `BinaryOperator`   | `is_equality()`, `is_arithmetic()`, `compare_inverse_operator()` |
| Unary    | `UnaryOperator`    | `is_keyword()` (typeof, void, delete)                  |
| Logical  | `LogicalOperator`  | `Or`, `And`, `Coalesce`, `to_assignment_operator()`    |
| Update   | `UpdateOperator`   | `Increment`, `Decrement`                               |

Precedence: `expr.precedence()` or `op.precedence()` — used by codegen for parenthesization.

### Control Flow Graph

```rust
let cfg = ControlFlowGraphBuilder::build(&program);
cfg.is_reachable(from, to)
cfg.basic_block(node).is_unreachable()
cfg.is_cyclic(node)
```

Edge types: `Jump`, `Normal`, `Backedge` (loop), `Unreachable` (dead code), `Error`.

---

## Layer 7 — Codegen

Depends on: Layer 2 (AST), optionally Layer 4 (semantics for mangling).

```rust
Codegen::new().build(&program).code                                         // Pretty
Codegen::new().with_options(CodegenOptions::minify()).build(&program).code   // Minified
Codegen::new().with_scoping(Some(scoping)).build(&program).code             // Mangled
```

---

## Common Patterns

### Expression Replacement (in exit_*)

```rust
fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {
    if let Expression::Identifier(ident) = expr {
        if let Some(new_expr) = self.try_replace(ident, ctx) {
            *expr = new_expr;
            self.modifications += 1;
        }
    }
}
```

### Statement List Rebuild (in exit_statements)

```rust
fn exit_statements(&mut self, stmts: &mut ArenaVec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {
    let allocator = ctx.ast.allocator;
    let mut new_stmts = ctx.ast.vec();
    for stmt in stmts.take_in(allocator) {
        if should_keep(&stmt) { new_stmts.push(stmt); }
    }
    *stmts = new_stmts;
}
```

### Value Extraction (handles ParenthesizedExpression)

```rust
fn extract_string(expr: &Expression) -> Option<&str> {
    match expr {
        Expression::StringLiteral(lit) => Some(lit.value.as_str()),
        Expression::ParenthesizedExpression(p) => extract_string(&p.expression),
        _ => None,
    }
}
```

### Inlining Safety Check

```rust
let safe_to_inline = ident.reference_id.get()
    .and_then(|ref_id| scoping.get_reference(ref_id).symbol_id())
    .map(|sym_id| !scoping.get_resolved_references(sym_id).any(|r| r.flags().is_write()))
    .unwrap_or(false);  // Unknown/global = not safe
```

### Subtree Substitution (VisitMut)

```rust
let mut visitor = ParamSubstituter { subs: &map, alloc: allocator };
visitor.visit_expression(&mut cloned_body);  // Auto-recursion via walk_mut
```

---

## Anti-Patterns

| #  | Mistake                              | Fix                                     |
| -- | ------------------------------------ | --------------------------------------- |
| 1  | `.reference_id()` / `.symbol_id()`  | Use `.get()?` (direct accessors panic)  |
| 2  | Stale scoping after structural changes | Rebuild: `SemanticBuilder::build()`  |
| 3  | `.clone()` on AST nodes             | `clone_in(ctx.ast.allocator)`           |
| 4  | Storing `&Expression` references     | Clone into `HashMap<SymbolId, Expression>` |
| 5  | Replace in `enter_*`                | Use `exit_*` (children not yet processed) |
| 6  | Manual recursion over ~50 variants   | Use `VisitMut` + `walk_mut::*`         |
| 7  | Skip `ParenthesizedExpression`       | Always recurse into parens             |
| 8  | Match `Expression::MemberExpression` | Does not exist — use specific variants |
| 9  | `VisitMut` for semantic transforms   | Use `Traverse` (need symbols/ancestors) |

---

## Safety Summary

**MUST**:

| Rule                            | Reason                           |
| ------------------------------- | -------------------------------- |
| `SPAN` for generated nodes      | Identifies synthetic nodes       |
| Create nodes via `ctx.ast`      | Arena allocation                 |
| `clone_in` / `take_in` for AST | Preserves semantic IDs           |
| Dummy cleanup after `take_in`   | Prevents invalid AST             |
| Recycle allocator via `reset()` | Performance, no unbounded growth |
| `Traverse` for semantic passes  | Need symbols/ancestors           |
| Transform in `exit_*`          | Children processed first         |
| `.get()` for ID resolution      | Direct accessors panic           |
| `ContentEq` for AST comparison  | Ignores spans                    |
| Rebuild scoping after changes   | Stale IDs corrupt results        |

**MUST NOT**:

| Rule                               | Reason                   |
| ---------------------------------- | ------------------------ |
| `.clone()` on AST nodes            | Breaks lifetimes/IDs     |
| `Drop` types in arena              | Compile-time forbidden   |
| Match `MemberExpression` directly  | Does not exist as variant |
| `VisitMut` for semantic transforms | No symbol/ancestor access |
| Hold ancestors across calls        | Stack invalidation       |
| Assume `Vec` addresses stable      | Unstable after mutation  |
