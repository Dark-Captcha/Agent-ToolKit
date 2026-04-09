# OXC API Reference

> **Version:** 2.0.0 | **Status:** Active | **Updated:** 2026-04-10

API reference for OXC crates in Rust: parsing, AST, traversal, semantic analysis, and code generation. For deobfuscation patterns and recipes, see `handbook.md`.

---

## Critical Assumptions

These are non-negotiable. Violating any causes undefined behavior, panics, or semantic corruption.

| #   | Assumption                        | Consequence                                                        |
| --- | --------------------------------- | ------------------------------------------------------------------ |
| 1   | Arena allocation via `Allocator`  | No individual `Drop`. Memory freed only on `reset()` or drop.     |
| 2   | `clone_in`/`take_in` preserve IDs | `SymbolId`/`ReferenceId`/`ScopeId` survive. Standard `.clone()` forbidden. |
| 3   | "Without" type safety             | Compile-time prevents aliasing: cannot access child field from parent. |
| 4   | Unified `Scoping` structure       | SoA layout. IDs are `NonMaxU32`. `Send + Sync`, mutation needs exclusive access. |
| 5   | Generated code is authoritative   | Files in `*/generated/` are auto-generated. Never edit manually.   |
| 6   | Parser produces valid AST         | Even with errors. `panicked == true` means empty AST.              |
| 7   | Deobfuscation needs semantics     | Symbol resolution, reference tracking, side-effect analysis required. |

---

## Core Types

### Span

```rust
pub struct Span { pub start: u32, pub end: u32 }  // Byte range, 8 bytes
pub const SPAN: Span = Span::new(0, 0);            // Use for ALL generated nodes
```

| Method              | Purpose                         |
| ------------------- | ------------------------------- |
| `size()`            | `end - start`                   |
| `is_unspanned()`    | `self == SPAN` (generated node) |
| `merge(other)`      | Union of two spans              |
| `source_text(src)`  | Extract substring               |
| `&source[span]`     | Direct extraction (panics on invalid) |

Traits: `GetSpan` (all AST nodes), `GetSpanMut` (rare).

### Atom<'a>

Arena-allocated interned string. O(1) equality. Tied to allocator lifetime.

```rust
Atom::from("string")                              // Borrows &str
Atom::from_in("string", &allocator)               // Allocates in arena
format_atom!(&allocator, "prefix_{}", var)         // Format macro
Atom::empty()                                      // Empty
```

Used for: `IdentifierReference.name`, `BindingIdentifier.name`, `StringLiteral.value`.

### CompactStr

Owned, lifetimeless string. Inline for 16 bytes or less.

```rust
CompactStr::new("string")
atom.into_compact_str()  // Atom -> CompactStr
```

### ContentEq

```rust
expr1.content_eq(&expr2)  // Structural comparison ignoring spans
```

Special: `NaN.content_eq(NaN) == true`, `0.0.content_eq(-0.0) == false`.

---

## SourceType

```rust
pub struct SourceType { language: Language, module_kind: ModuleKind, variant: LanguageVariant }
```

| Constructor                | Result                        |
| -------------------------- | ----------------------------- |
| `SourceType::from_path(p)` | Auto-detect from extension    |
| `SourceType::unambiguous()` | Parser detects module kind   |
| `SourceType::mjs()`        | JavaScript + Module           |
| `SourceType::cjs()`        | JavaScript + Script           |
| `SourceType::ts()`         | TypeScript + Module           |
| `SourceType::tsx()`        | TypeScript + Module + JSX     |

Queries: `is_javascript()`, `is_typescript()`, `is_module()`, `is_jsx()`, `is_strict()`.

Modules are always strict mode.

---

## Memory Model

### Allocator

```rust
let mut allocator = Allocator::default();       // Lazy
let mut allocator = Allocator::with_capacity(1 << 20);  // Pre-allocate

// MUST recycle across files
for file in files {
    process(&allocator, file);
    allocator.reset();  // Reuse memory, warm cache
}
```

| Arena Type          | Creation                         |
| ------------------- | -------------------------------- |
| `Box<'a, T>`        | `Box::new_in(value, &allocator)` |
| `Vec<'a, T>`        | `Vec::new_in(&allocator)`        |
| `HashMap<'a, K, V>` | `HashMap::new_in(&allocator)`    |

Thread safety: `Sync` if contents are. **Not `Send`**.

### CloneIn and TakeIn

Both preserve `SymbolId`/`ReferenceId`/`ScopeId`.

| Operation  | Effect                     | Leaves Behind   | Use When                  |
| ---------- | -------------------------- | --------------- | ------------------------- |
| `clone_in` | Deep copy in arena         | Original intact | Need original + copy      |
| `take_in`  | Move ownership out         | Dummy value     | Moving to new location    |
| Direct `*` | In-place replacement       | N/A             | Simple replacement        |

Dummy values: Expression -> `NullLiteral`, Statement -> `EmptyStatement`, Vec -> empty.

**MUST** replace or remove dummy after `take_in`.

```rust
self.values.insert(symbol_id, init.clone_in(ctx.ast.allocator));  // Clone for storage
*expr = cond.consequent.take_in(ctx.ast.allocator);               // Take for movement

// Vector rebuild
let mut new_stmts = ctx.ast.vec();
for stmt in stmts.take_in(allocator) {
    if keep(&stmt) { new_stmts.push(stmt); }
}
*stmts = new_stmts;
```

Address stability: `Box<T>` always stable. `Vec<T>` items unstable after mutation.

---

## AST Structure

### Expression Enum (~40 variants, `#[repr(C, u8)]`)

All variants boxed: `Expression::StringLiteral(Box<'a, StringLiteral<'a>>)`.

| Range | Category   | Examples                                                |
| ----- | ---------- | ------------------------------------------------------- |
| 0-6   | Literals   | Boolean, Null, Numeric, String, Template, BigInt, RegExp |
| 10    | Identifier | `IdentifierReference`                                   |
| 20-21 | Collections | Array, Object                                          |
| 25-27 | Functions  | Arrow, Function, Class expression                       |
| 30-36 | Operations | Unary, Binary, Logical, Conditional, Assignment, Sequence |
| 40-42 | Calls      | Call, New, Import                                       |
| 48-50 | Members    | Computed, Static, PrivateField (inherited via `inherit_variants!`) |

### Statement Enum (~30 variants)

Key: `ExpressionStatement`, `ReturnStatement`, `IfStatement`, `SwitchStatement`, loops, `VariableDeclaration`, `FunctionDeclaration`.

### Type-Safe Identifiers (4 types, MUST NOT mix)

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
    pub declare: bool,
}

pub struct Function<'a> {
    pub span: Span,
    pub id: Option<BindingIdentifier<'a>>,
    pub generator: bool,
    pub r#async: bool,
    pub params: Box<'a, FormalParameters<'a>>,
    pub body: Option<Box<'a, FunctionBody<'a>>>,
    pub scope_id: Cell<Option<ScopeId>>,
    pub pure: bool,  // @__NO_SIDE_EFFECTS__
}
```

### Enum Inheritance

`inherit_variants!` flattens nested enums. `Expression::ComputedMemberExpression` exists directly.

**MUST NOT** match `Expression::MemberExpression` — does not exist. Use specific variants or `match_member_expression!()`.

### AstBuilder (Generated)

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

// Collections
ctx.ast.vec()       // Empty arena vec
ctx.ast.vec1(item)  // Single-element vec
```

---

## Parser

```rust
let ret = Parser::new(&allocator, source, source_type)
    .with_options(ParseOptions {
        preserve_parens: true,  // Emit ParenthesizedExpression
        ..Default::default()
    })
    .parse();
```

| Field           | Type                  | Notes                            |
| --------------- | --------------------- | -------------------------------- |
| `ret.program`   | `Program<'a>`        | Always structurally valid        |
| `ret.errors`    | `Vec<OxcDiagnostic>` | Check for semantic issues        |
| `ret.panicked`  | `bool`               | `true` = AST empty               |

Use `preserve_parens: true` for transformations. Expression-only: `Parser::new(...).parse_expression()?`.

---

## Traversal System

### Traverse Trait

```rust
impl<'a> Traverse<'a, ()> for MyPass {
    fn enter_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn enter_statements(&mut self, stmts: &mut ArenaVec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_statements(&mut self, stmts: &mut ArenaVec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
}
```

**Rule**: Collect in `enter_*`, transform in `exit_*` (children processed first).

Key hooks: `enter/exit_expression`, `enter/exit_statements`, `enter/exit_function`, `enter/exit_variable_declarator`, `enter/exit_call_expression`, `enter_identifier_reference`, `enter_binding_identifier`.

### TraverseCtx

| Access                | Purpose                |
| --------------------- | ---------------------- |
| `ctx.ast.allocator`   | Get allocator          |
| `ctx.ast.*`           | Create new AST nodes   |
| `ctx.scoping()`       | Immutable scoping      |
| `ctx.scoping_mut()`   | Mutable scoping (rare) |
| `ctx.parent()`        | Parent ancestor        |
| `ctx.ancestor(level)` | Nth ancestor (0=parent) |
| `ctx.ancestors()`     | Iterator parent->root  |

### Ancestor System (Without Types)

Each variant encodes parent type + child direction. "Without" types prevent accessing the descent child:

```rust
match ctx.parent() {
    Ancestor::CallExpressionCallee(call) => { /* callee context */ }
    Ancestor::BinaryExpressionLeft(bin) => {
        bin.operator()  // OK: sibling access
        bin.right()     // OK: sibling access
        // bin.left()   // Does not exist: compile-time prevented
    }
}
```

### Traverse vs VisitMut

| Feature         | `Traverse`                     | `VisitMut`                   |
| --------------- | ------------------------------ | ---------------------------- |
| Context         | Full `TraverseCtx`             | None                         |
| Ancestors       | Yes                            | No                           |
| Symbol access   | Yes                            | No                           |
| Use for         | **All deobfuscation passes**   | Simple subtree transforms    |

---

## Semantic Analysis

```rust
let scoping = SemanticBuilder::build(&program);
```

### Core ID Types (all `NonMaxU32`)

`SymbolId` (declaration), `ReferenceId` (usage), `ScopeId` (scope).

### Flags

| Flag Type        | Key Values                                                      | Key Queries              |
| ---------------- | --------------------------------------------------------------- | ------------------------ |
| `SymbolFlags`    | `FunctionScopedVariable`, `BlockScopedVariable`, `ConstVariable`, `Function`, `Class`, `Import` | `is_const_variable()`, `is_function()`, `is_value()`, `is_type()` |
| `ReferenceFlags` | `Read`, `Write`, `Type`                                        | `is_read()`, `is_write()`, `is_read_only()` |
| `ScopeFlags`     | `Top`, `Function`, `Arrow`, `StrictMode`                       | `is_var()` (hoisting boundary), `is_function()` |

### Key Operations

```rust
scoping.symbol_name(id)                              // Get name
scoping.symbol_flags(id)                             // Get flags
scoping.get_resolved_references(id)                  // Iterator of references
scoping.get_reference(ref_id) -> &Reference          // Single reference
scoping.root_unresolved_references()                 // Globals map
scoping.find_binding(scope_id, "name")               // Scope lookup
```

---

## ECMAScript Helpers

### Type Coercion

| Trait        | Method          | Falsey values                                        |
| ------------ | --------------- | ---------------------------------------------------- |
| `ToBoolean`  | `to_boolean()`  | `false`, `0`, `""`, `null`, `undefined`, `NaN`       |
| `ToNumber`   | `to_number()`   | `true`->1, `""`->0, `null`->0                        |
| `ToJsString` | `to_js_string()` | `null`->"null", arrays joined with ","              |

### Constant Evaluation

```rust
expr.evaluate_value(ctx) -> Option<ConstantValue<'a>>
expr.get_side_free_number_value(ctx) -> Option<f64>  // None if side effects
```

**MUST** use `get_side_free_*` variants before folding.

### Side Effect Analysis

```rust
expr.may_have_side_effects(ctx) -> bool
```

No side effects: literals, `this`, identifiers, pure unary/binary/logical, function expressions.

### Value Type

```rust
expr.value_type(ctx) -> ValueType  // Undefined, Null, Number, String, Boolean, Object, Undetermined
```

---

## Code Generation

```rust
Codegen::new().build(&program).code                              // Pretty
Codegen::new().with_options(CodegenOptions::minify()).build(&program).code  // Minified
Codegen::new().with_scoping(Some(scoping)).build(&program).code  // Mangled
```

---

## Operator Quick Reference

### BinaryOperator

| Operator     | Category   | Inverse | Precedence     |
| ------------ | ---------- | ------- | -------------- |
| `===`/`!==`  | Equality   | Each other | Equals      |
| `<`/`<=`/`>`/`>=` | Compare | Inverse pair | Compare |
| `+`/`-`      | Arithmetic | Each other | Add         |
| `*`/`/`/`%`  | Arithmetic | —       | Multiply       |
| `**`         | Arithmetic | —       | Exponentiation (right-assoc) |
| `<<`/`>>`/`>>>` | Bitshift | —    | Shift          |
| `\|`/`^`/`&` | Bitwise   | —       | BitwiseOr/Xor/And |
| `in`/`instanceof` | Relational | — | Relational   |

### UnaryOperator

`+`, `-`, `!`, `~`, `typeof`, `void`, `delete`

### LogicalOperator

`||`, `&&`, `??` — assignment variants: `||=`, `&&=`, `??=`

---

## Safety Rules Summary

**MUST**:

| Rule                                  | Reason                              |
| ------------------------------------- | ----------------------------------- |
| Use `SPAN` for generated nodes        | Identifies synthetic nodes          |
| Create nodes via `ctx.ast`            | Arena allocation required           |
| Use `clone_in`/`take_in`             | Preserves semantic IDs              |
| Replace dummy after `take_in`         | Prevents invalid AST                |
| Recycle allocator with `reset()`      | Performance, no unbounded growth    |
| Use `Traverse` for semantic passes    | Need symbol/ancestor access         |
| Transform in `exit_*` hooks          | Children processed first            |
| Use `.get()` for safe ID resolution   | Direct accessors panic if not set   |
| Use `ContentEq` for AST comparison    | Ignores spans correctly             |
| Rebuild scoping after structural changes | Stale IDs cause wrong results   |

**MUST NOT**:

| Rule                                  | Reason                              |
| ------------------------------------- | ----------------------------------- |
| Standard `.clone()` on AST nodes      | Breaks lifetimes, loses IDs         |
| `Drop` types in arena                 | Compile-time forbidden              |
| Match `Expression::MemberExpression`  | Does not exist                      |
| Use `VisitMut` for semantic transforms | No symbol/ancestor access          |
| Hold ancestors across visitor calls   | Stack invalidation                  |
| Assume `Vec` addresses stable         | May move on push                    |
