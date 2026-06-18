# OXC Reference

> **Version:** 2.0.0 | **Updated:** 2026-06-12

OXC (the JavaScript Oxidation Compiler) crate ecosystem for JavaScript/TypeScript parsing, analysis, transformation, and code generation in Rust. Verified against **OXC v0.135.0** source. Organized bottom-up by dependency layer.

---

| #   | Section                                      |
| --- | -------------------------------------------- |
| 1   | [Invariants](#invariants)                    |
| 2   | [Layer 0 — Memory](#layer-0--memory)         |
| 3   | [Layer 1 — Primitives](#layer-1--primitives) |
| 4   | [Layer 2 — AST](#layer-2--ast)               |
| 5   | [Layer 3 — Parser](#layer-3--parser)         |
| 6   | [Layer 4 — Semantics](#layer-4--semantics)   |
| 7   | [Layer 5 — Traversal](#layer-5--traversal)   |
| 8   | [Layer 6 — Analysis](#layer-6--analysis)     |
| 9   | [Layer 7 — Codegen](#layer-7--codegen)       |
| 10  | [Common Patterns](#common-patterns)          |
| 11  | [Anti-Patterns](#anti-patterns)              |
| 12  | [Safety Summary](#safety-summary)            |

---

## Invariants

These apply to every layer. Violating any causes UB, panics, or silent corruption.

| #   | Rule                            | Detail                                                                                                                  |
| --- | ------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 1   | Arena allocation                | All AST nodes live in `Allocator`. No individual `Drop`.                                                                |
| 2   | `clone_in` resets semantic IDs  | `clone_in` clears `SymbolId`/`ReferenceId`/`ScopeId`/`NodeId`. `clone_in_with_semantic_ids` preserves them.             |
| 3   | "Without" type safety           | Compile-time prevents child-field aliasing in traversal.                                                                |
| 4   | Generated code is authoritative | `*/generated/` files are auto-generated. Never edit.                                                                    |
| 5   | IDs are `NonMaxU32`             | Valid IDs are never `u32::MAX`. Semantic IDs live in `Cell<Option<_>>`; `NodeId` lives in `Cell<NodeId>`.               |
| 6   | Semantic analysis assigns IDs   | After parsing alone, semantic ID cells are `None` and every `node_id` is `NodeId::DUMMY`. `SemanticBuilder` fills them. |

---

## Layer 0 — Memory

Foundation (`oxc_allocator`). Everything allocates here.

### Allocator

```rust
let mut allocator = Allocator::default();               // Lazy
let mut allocator = Allocator::with_capacity(1 << 20);  // Pre-allocate
allocator.reset();                                      // Reuse memory
```

**MUST** recycle with `reset()` across units of work. New allocator per file is expensive. No reset causes unbounded growth.

### Arena Types

| Type                | Creation                                                      |
| ------------------- | ------------------------------------------------------------- |
| `Box<'a, T>`        | `Box::new_in(value, &allocator)`                              |
| `Vec<'a, T>`        | `Vec::new_in(&allocator)`, `with_capacity_in`, `from_iter_in` |
| `HashMap<'a, K, V>` | `HashMap::new_in(&allocator)`                                 |
| `HashSet<'a, T>`    | `HashSet::new_in(&allocator)`                                 |
| `StringBuilder<'a>` | `StringBuilder::new_in(&allocator)`, `from_strs_array_in`     |

Thread safety:

| Type                          | `Send` | `Sync`                 |
| ----------------------------- | ------ | ---------------------- |
| `Allocator`                   | Yes    | No                     |
| `Box<'a, T>`                  | No     | No                     |
| `Vec` / `HashMap` / `HashSet` | No     | If contents are `Sync` |

### CloneIn / TakeIn

| Operation                    | Effect           | Semantic IDs             | Use When                                      |
| ---------------------------- | ---------------- | ------------------------ | --------------------------------------------- |
| `clone_in`                   | Deep arena copy  | Reset (`None` / `DUMMY`) | Copy feeds a fresh `SemanticBuilder` run      |
| `clone_in_with_semantic_ids` | Deep arena copy  | Preserved                | Reusing copy while current `Scoping` is valid |
| `take_in`                    | Move out of node | Preserved (moved intact) | Moving to a new location                      |
| Direct `*expr = ...`         | In-place replace | N/A                      | Simple replacement                            |

`take_in` leaves a dummy behind: `Expression` → `NullLiteral`, `Statement` → `DebuggerStatement`, `Vec` → empty. **MUST** replace or remove the dummy. `take_in` accepts any `AllocatorAccessor` — pass `&allocator` or `ctx.ast` directly. `take_in_box` returns the moved value boxed.

```rust
// Clone for storage — keep IDs only while scoping stays valid
self.stored.insert(id, expr.clone_in_with_semantic_ids(ctx.ast.allocator));

// Take for movement
*expr = replacement.take_in(ctx.ast);

// Vector rebuild
let mut new_stmts = ctx.ast.vec();
for stmt in stmts.take_in(ctx.ast) {
    if keep(&stmt) { new_stmts.push(stmt); }
}
*stmts = new_stmts;
```

### Address Stability

`Box<T>`: always stable. `Vec<T>` items: unstable after push/mutation.

### AllocatorPool

```rust
let pool = AllocatorPool::new(num_threads);   // Also: AllocatorPool::new_fixed_size(n)
let guard = pool.get();                       // AllocatorGuard — returns to pool on drop
```

---

## Layer 1 — Primitives

`Span` and `SourceType` live in `oxc_span`. String types live in `oxc_str` (split out of `oxc_span` in v0.125). The umbrella `oxc` crate does **not** re-export `oxc_str` — depend on it directly. `oxc_ast::ast` re-exports `Str`.

### Span

```rust
pub struct Span { pub start: u32, pub end: u32 }  // Byte range, 8 bytes
pub const SPAN: Span = Span::new(0, 0);           // ALL generated nodes use this
```

| Method                     | Purpose                         |
| -------------------------- | ------------------------------- |
| `size()`                   | `end - start`                   |
| `is_empty()`               | `start == end`                  |
| `is_unspanned()`           | `self == SPAN` (generated node) |
| `merge(other)`             | Smallest span containing both   |
| `expand(n)` / `shrink(n)`  | Grow or cut both ends           |
| `contains_inclusive(span)` | Range containment               |
| `source_text(src)`         | Extract source substring        |

Traits: `GetSpan` on all AST nodes. `GetSpanMut` for rare location modification.

### Ident<'a> (oxc_str)

Identifier string with a precomputed hash (replaces the former `Atom`). Stores pointer + length + hash — 16 bytes on 64-bit. The hash makes `HashMap` lookups and equality comparisons fast. Lifetime tied to the allocator that owns the text.

```rust
Ident::from("str")                                 // Borrows &str, computes hash
let i: Ident = FromIn::from_in("str", &allocator); // Copies into arena (oxc_allocator::FromIn)
Ident::from_strs_array_in(["a", "b"], &allocator)  // Concatenation in arena
static_ident!("use strict")                        // Compile-time hash, 'static
```

Used for identifier names and property names. Companion collections: `IdentHashMap`, `IdentHashSet`, `ArenaIdentHashMap`.

### Str<'a> (oxc_str)

Plain arena string — a `&'a str` wrapper for non-identifier text: string literal values, raw literal text. `as_str()` plus `Deref<Target = str>`.

### CompactStr (oxc_str)

Owned string, no lifetime. `CompactStr::new("s")`; `CompactStr::new_const` stores up to `MAX_INLINE_LEN` (16) bytes inline at compile time. Convert from arena strings via `Ident::to_compact_str` / `into_compact_str`.

### ContentEq

Import from `oxc_span::ContentEq` (umbrella: `oxc::span::ContentEq`).

```rust
use oxc_span::ContentEq;
expr1.content_eq(&expr2)  // Structural equality; ignores spans and node IDs
```

`f64` comparison is bit-pattern equality: `f64::NAN.content_eq(f64::NAN) == true` (identical bits), but NaNs produced by arithmetic are not guaranteed to match. `0.0.content_eq(-0.0) == false`.

### SourceType

```rust
pub struct SourceType { language: Language, module_kind: ModuleKind, variant: LanguageVariant }
```

| Constructor                  | Result                              |
| ---------------------------- | ----------------------------------- |
| `SourceType::from_path(p)`   | `Result` — auto-detect by extension |
| `SourceType::from_extension` | `Result` — from extension string    |
| `SourceType::unambiguous()`  | Parser detects module kind          |
| `SourceType::mjs()`          | JavaScript + Module                 |
| `SourceType::cjs()`          | JavaScript + CommonJS               |
| `SourceType::script()`       | JavaScript + Script                 |
| `SourceType::jsx()`          | JavaScript + Module + JSX           |
| `SourceType::ts()`           | TypeScript + Module                 |
| `SourceType::tsx()`          | TypeScript + Module + JSX           |
| `SourceType::d_ts()`         | TypeScript declaration file         |

Queries: `is_javascript()`, `is_typescript()`, `is_module()`, `is_commonjs()`, `is_jsx()`, `is_strict()`. Modules are always strict mode.

---

## Layer 2 — AST

Depends on: Layer 0 (arena), Layer 1 (`Span`, `Ident`, `Str`).

### NodeId

Every AST node's first field is `pub node_id: Cell<NodeId>`. `NodeId::DUMMY` (= 0 = `NodeId::ROOT`, the `Program` node) until `SemanticBuilder` assigns real IDs. `AstBuilder` always creates nodes with `DUMMY`.

### Expression (43 variants, `#[repr(C, u8)]`)

All variants boxed: `Expression::StringLiteral(Box<'a, StringLiteral<'a>>)`. Discriminants 0-39 run sequentially; member expressions keep 48-50.

| Range | Category   | Variants                                                                                                                                                                                         |
| ----- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0-6   | Literals   | Boolean, Null, Numeric, BigInt, RegExp, String, Template                                                                                                                                         |
| 7     | Identifier | `IdentifierReference`                                                                                                                                                                            |
| 8-9   | Meta       | MetaProperty, Super                                                                                                                                                                              |
| 10-31 | Operations | Array, Arrow, Assignment, Await, Binary, Call, Chain, Class, Conditional, Function, Import, Logical, New, Object, Parenthesized, Sequence, TaggedTemplate, This, Unary, Update, Yield, PrivateIn |
| 32-33 | JSX        | JSXElement, JSXFragment                                                                                                                                                                          |
| 34-38 | TypeScript | TSAs, TSSatisfies, TSTypeAssertion, TSNonNull, TSInstantiation                                                                                                                                   |
| 39    | Intrinsics | V8IntrinsicExpression                                                                                                                                                                            |
| 48-50 | Members    | Computed, Static, PrivateField (via `inherit_variants!`)                                                                                                                                         |

`inherit_variants!` flattens nested enums. `Expression::ComputedMemberExpression` exists directly. **`Expression::MemberExpression` does NOT exist** — use specific variants or `match_member_expression!()` / `match_expression!()`.

### Statement (~30 variants)

Key: `ExpressionStatement`, `ReturnStatement`, `IfStatement`, `SwitchStatement`, `WhileStatement`, `ForStatement`, `VariableDeclaration`, `FunctionDeclaration`, `BlockStatement`, `EmptyStatement`.

### Identifiers (4 types — MUST NOT mix)

| Type                  | Purpose               | Semantic ID Field                         |
| --------------------- | --------------------- | ----------------------------------------- |
| `IdentifierName`      | Property names        | None                                      |
| `IdentifierReference` | Variable reads/writes | `reference_id: Cell<Option<ReferenceId>>` |
| `BindingIdentifier`   | Declarations          | `symbol_id: Cell<Option<SymbolId>>`       |
| `LabelIdentifier`     | Loop/block labels     | None                                      |

All four also carry `node_id: Cell<NodeId>` like every node.

### Key Structs

```rust
pub struct VariableDeclaration<'a> {
    pub node_id: Cell<NodeId>,
    pub span: Span,
    pub kind: VariableDeclarationKind,  // Var, Let, Const, Using, AwaitUsing
    pub declarations: Vec<'a, VariableDeclarator<'a>>,
    pub declare: bool,
}

pub struct VariableDeclarator<'a> {
    pub node_id: Cell<NodeId>,
    pub span: Span,
    pub kind: VariableDeclarationKind,
    pub id: BindingPattern<'a>,
    pub type_annotation: Option<Box<'a, TSTypeAnnotation<'a>>>,
    pub init: Option<Expression<'a>>,
    pub definite: bool,
}

pub struct Function<'a> {
    pub node_id: Cell<NodeId>,
    pub span: Span,
    pub r#type: FunctionType,
    pub id: Option<BindingIdentifier<'a>>,
    pub generator: bool,
    pub r#async: bool,
    pub declare: bool,
    pub type_parameters: Option<Box<'a, TSTypeParameterDeclaration<'a>>>,
    pub this_param: Option<Box<'a, TSThisParameter<'a>>>,
    pub params: Box<'a, FormalParameters<'a>>,
    pub return_type: Option<Box<'a, TSTypeAnnotation<'a>>>,
    pub body: Option<Box<'a, FunctionBody<'a>>>,
    pub scope_id: Cell<Option<ScopeId>>,
    pub pure: bool,  // /* @__PURE__ */ annotation
    pub pife: bool,  // Possibly-invoked function expression
}
```

`BindingPattern` is an enum (`BindingIdentifier` | `ObjectPattern` | `ArrayPattern` | `AssignmentPattern`), no longer a struct — its old `type_annotation`/`optional` fields moved onto `VariableDeclarator` and parameter nodes.

### AstBuilder (Generated)

All node creation goes through `AstBuilder` (accessed via `ctx.ast` in traversal). Name parameters accept `impl Into<Ident<'a>>`, string values accept `impl Into<Str<'a>>` — pass `&str` directly.

```rust
// Literals
ctx.ast.expression_string_literal(SPAN, "value", None)   // raw: Option<Str>
ctx.ast.expression_numeric_literal(SPAN, 42.0, None, NumberBase::Decimal)
ctx.ast.expression_boolean_literal(SPAN, true)
ctx.ast.expression_null_literal(SPAN)

// Identifiers and operations
ctx.ast.expression_identifier(SPAN, "name")
ctx.ast.expression_binary(SPAN, left, BinaryOperator::Addition, right)
ctx.ast.expression_unary(SPAN, UnaryOperator::UnaryNegation, argument)

// Statements
ctx.ast.statement_expression(SPAN, expr)
ctx.ast.statement_return(SPAN, Some(expr))
ctx.ast.statement_block(SPAN, stmts)

// Variable declaration: `const x = init` (NONE = oxc_ast::NONE, fills type_annotation)
let id = ctx.ast.binding_pattern_binding_identifier(SPAN, "x");
let declarator = ctx.ast.variable_declarator(
    SPAN, VariableDeclarationKind::Const, id, NONE, Some(init), false,
);

// Collections and helpers
ctx.ast.vec()              // Empty arena vec
ctx.ast.vec1(item)         // Single-element vec
ctx.ast.vec_from_iter(it)  // From iterator
ctx.ast.alloc(node)        // Box<'a, T>
ctx.ast.ident("x")         // Ident<'a> in arena
ctx.ast.str("text")        // Str<'a> in arena
```

---

## Layer 3 — Parser

Depends on: Layer 0-2.

```rust
let ret = Parser::new(&allocator, source, source_type)
    .with_options(ParseOptions {
        preserve_parens: true,  // Emit ParenthesizedExpression nodes
        ..ParseOptions::default()
    })
    .parse();
```

| Field                       | Type               | Notes                             |
| --------------------------- | ------------------ | --------------------------------- |
| `ret.program`               | `Program<'a>`      | Empty if `panicked`               |
| `ret.diagnostics`           | `Diagnostics`      | Syntax errors and warnings        |
| `ret.panicked`              | `bool`             | `true` = unrecoverable, AST empty |
| `ret.module_record`         | `ModuleRecord<'a>` | ESM import/export metadata        |
| `ret.tokens`                | `Vec<'a, Token>`   | Only with `TokensParserConfig`    |
| `ret.irregular_whitespaces` | `Box<[Span]>`      | For oxlint                        |
| `ret.is_flow_language`      | `bool`             | Flow syntax detected              |

`Diagnostics` (newtype over `Vec<OxcDiagnostic>`): `has_errors()`, `has_warnings()`, `errors()`, `warnings()`, `into_vec()`.

`ParseOptions` fields: `parse_regular_expression`, `allow_return_outside_function`, `preserve_parens`.

Expression-only parsing: `Parser::new(...).parse_expression() -> Result<Expression, Diagnostics>`.

**MUST** check `diagnostics` and `panicked`. AST is structurally valid even with errors (recovery parser). Use `preserve_parens: true` when transforming. Parser-side checks are not comprehensive — run `SemanticBuilder::new().with_check_syntax_error(true)` for full syntax validation.

---

## Layer 4 — Semantics

Depends on: Layer 2 (AST). Produces symbol table, reference table, scope tree — and assigns `node_id`s.

```rust
let ret = SemanticBuilder::new().build(&program);
// ret.diagnostics: Diagnostics
// ret.semantic: Semantic<'a>

let scoping = ret.semantic.scoping();       // &Scoping (borrow)
let scoping = ret.semantic.into_scoping();  // Scoping (owned, consumes Semantic)
```

### Builder Options (all off by default)

| Option                          | Effect                                                               |
| ------------------------------- | -------------------------------------------------------------------- |
| `with_check_syntax_error(true)` | Full syntax error validation                                         |
| `with_build_nodes(true)`        | Populate `Semantic::nodes()` — required for `AstNodes` random access |
| `with_class_table(true)`        | Build the class table                                                |
| `with_enum_eval(true)`          | Evaluate TS enum member values (const enum inlining)                 |
| `with_cfg(true)`                | Build control flow graph (cargo feature `cfg`)                       |
| `with_stats(stats)`             | Pre-size allocations from AST stats (else computed by a pre-pass)    |

**`Semantic::nodes()` is empty unless `with_build_nodes(true)` is set.**

Accessors on `Semantic`: `scoping()`, `scoping_mut()`, `into_scoping()`, `nodes()`, `into_scoping_and_nodes()`, `comments()`, `cfg()`.

### Scoping (SoA layout)

Unified structure containing symbols, references, and scope tree. No lifetime parameter — owns its data in an internal arena. `Send + Sync`. Mutation needs exclusive access.

### IDs

| Type          | Represents            | Storage on AST node                |
| ------------- | --------------------- | ---------------------------------- |
| `SymbolId`    | Declaration (binding) | `Cell<Option<SymbolId>>`           |
| `ReferenceId` | Usage (reference)     | `Cell<Option<ReferenceId>>`        |
| `ScopeId`     | Lexical scope         | `Cell<Option<ScopeId>>`            |
| `NodeId`      | AST node identity     | `Cell<NodeId>` (DUMMY until built) |

### Flags

| Type             | Key Values                                                                                                                                                                                         | Key Queries                                                                     |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `SymbolFlags`    | `FunctionScopedVariable`, `BlockScopedVariable`, `ConstVariable`, `Class`, `CatchVariable`, `Function`, `Import`, `TypeAlias`, `Interface`, `RegularEnum`, `ConstEnum`, `TypeParameter`, `Ambient` | `is_const_variable()`, `is_function()`, `is_value()`                            |
| `ReferenceFlags` | `Read`, `Write`, `Type`, `ValueAsType`, `Namespace`, `MemberWriteTarget`                                                                                                                           | `is_read()`, `is_write()`, `is_read_only()`                                     |
| `ScopeFlags`     | `Top`, `Function`, `Arrow`, `StrictMode`, `ClassStaticBlock`, `TsModuleBlock`, `Constructor`, `GetAccessor`, `SetAccessor`, `CatchClause`, `DirectEval`, `TsConditional`                           | `is_var()` (hoisting boundary = Top, Function, ClassStaticBlock, TsModuleBlock) |

### Safe ID Resolution

Generated accessors `.reference_id()` / `.symbol_id()` / `.scope_id()` **panic** if unset — use them only on post-semantic ASTs. Use `.get()` when unsure; `set_*` counterparts exist.

```rust
// IdentifierReference -> SymbolId
let ref_id = ident.reference_id.get()?;
let symbol_id = scoping.get_reference(ref_id).symbol_id()?;  // None = global/unresolved

// BindingIdentifier -> SymbolId
let symbol_id = binding.symbol_id.get()?;

// VariableDeclarator -> SymbolId
decl.id.get_binding_identifier().and_then(|b| b.symbol_id.get())

// Function -> SymbolId
func.id.as_ref().and_then(|id| id.symbol_id.get())
```

### Queries

```rust
scoping.symbol_name(id)                              // &str
scoping.symbol_flags(id)                             // SymbolFlags
scoping.symbol_scope_id(id)                          // ScopeId
scoping.get_resolved_references(id)                  // Iterator<&Reference>
scoping.get_reference(ref_id)                        // &Reference: symbol_id(), node_id(), flags()
scoping.root_unresolved_references()                 // Globals: &ArenaIdentHashMap<ArenaVec<ReferenceId>>
scoping.get_binding(scope_id, Ident::from("name"))   // This scope only
scoping.find_binding(scope_id, Ident::from("name"))  // Walks parent scopes
scoping.rename_symbol(symbol_id, scope_id, new_name) // Rename a binding
```

`get_binding` / `find_binding` take `Ident`, not `&str` — wrap with `Ident::from("x")` or `static_ident!("x")`.

### Reference Counting

```rust
// Read count
scoping.get_resolved_references(id).filter(|r| r.flags().is_read()).count()

// Write count
scoping.get_resolved_references(id).filter(|r| r.flags().is_write()).count()
```

### Scoping Rebuild Rules

| Situation                  | Action  | Reason                      |
| -------------------------- | ------- | --------------------------- |
| Read-only pass             | Share   | IDs still valid             |
| Collect + apply (no decls) | Share   | References unchanged        |
| Add/remove declarations    | Rebuild | IDs may invalidate          |
| Rename identifiers         | Rebuild | Binding lookup changes      |
| Before unused detection    | Rebuild | Need fresh reference counts |

Rebuild: `scoping = SemanticBuilder::new().build(&program).semantic.into_scoping();`

---

## Layer 5 — Traversal

Depends on: Layer 0 (allocator), Layer 2 (AST), Layer 4 (semantics). Crate: `oxc_traverse`.

### Traverse Trait

```rust
impl<'a> Traverse<'a, ()> for MyPass {
    fn enter_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {}
    // Vec = oxc_allocator::Vec
    fn enter_statements(&mut self, stmts: &mut Vec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
    fn exit_statements(&mut self, stmts: &mut Vec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {}
    // ... hundreds of enter_*/exit_* hooks
}
```

The second type parameter is user state, exposed as `ctx.state`. Use `()` when unneeded.

**Rule**: Collect in `enter_*` (top-down). Transform in `exit_*` (bottom-up, children processed).

Key hooks: `expression`, `statements`, `function`, `variable_declarator`, `call_expression`, `identifier_reference`, `binding_identifier`.

### TraverseCtx

| Access                | Purpose                     |
| --------------------- | --------------------------- |
| `ctx.state`           | User state (`State` param)  |
| `ctx.ast`             | `AstBuilder` — create nodes |
| `ctx.ast.allocator`   | Get allocator               |
| `ctx.alloc(node)`     | Box a node in the arena     |
| `ctx.scoping()`       | Immutable scoping           |
| `ctx.scoping_mut()`   | Mutable scoping (rare)      |
| `ctx.parent()`        | Parent ancestor             |
| `ctx.ancestor(level)` | Nth ancestor (0 = parent)   |
| `ctx.ancestors()`     | Iterator parent -> root     |

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

| Feature       | `Traverse` (`oxc_traverse`) | `VisitMut` (`oxc_ast_visit`) |
| ------------- | --------------------------- | ---------------------------- |
| Context       | Full `TraverseCtx`          | None                         |
| Ancestors     | Yes                         | No                           |
| Symbol access | Yes                         | No                           |
| Use for       | **Any semantic-aware pass** | Simple subtree transforms    |

Use `Traverse` for any pass that needs symbol resolution, ancestor context, or scope tracking. Use `VisitMut` only for syntactic subtree operations (e.g., parameter substitution in a cloned body).

### Running a Traverse Pass

```rust
let scoping = traverse_mut(&mut my_pass, &allocator, &mut program, scoping, state);
```

For many files with the same pass, build a `ReusableTraverseCtx::new(state, scoping, allocator)` once and call `traverse_mut_with_ctx`.

---

## Layer 6 — Analysis

Depends on: Layer 2 (AST), Layer 4 (semantics). Crate: `oxc_ecmascript`.

Every analysis query takes a context. `GlobalContext` supplies global-identifier knowledge; pass `WithoutGlobalReferenceInformation` when none is available.

### ECMAScript Type Coercion

| Trait        | Method              | Returns                              |
| ------------ | ------------------- | ------------------------------------ |
| `ToBoolean`  | `to_boolean(ctx)`   | `Option<bool>` — `None` undetermined |
| `ToNumber`   | `to_number(ctx)`    | `Option<f64>`                        |
| `ToJsString` | `to_js_string(ctx)` | `Option<Cow<str>>`                   |

Also exported: `ToBigInt`, `ToInt32`, `ToUint32`, `ToPrimitive`, `StringToNumber`, `ArrayJoin`, `PropName`, `BoundNames`.

### Constant Evaluation

```rust
expr.evaluate_value(ctx)              // Option<ConstantValue> — ignores side effects
expr.evaluate_value_to_boolean(ctx)   // Option<bool>; also _to_number, _to_bigint, _to_string
expr.get_side_free_number_value(ctx)  // None if expression has side effects
```

`get_side_free_{number,bigint,boolean,string}_value` — use these before folding. `ctx` implements `ConstantEvaluationCtx` (= `MayHaveSideEffectsContext` + `ast()`).

### Side-Effect Analysis

```rust
expr.may_have_side_effects(ctx) -> bool  // ctx: &impl MayHaveSideEffectsContext
```

`MayHaveSideEffectsContext` (supertrait `GlobalContext`) configures the analysis:

| Method                          | Controls                           |
| ------------------------------- | ---------------------------------- |
| `annotations()`                 | Respect `/* @__PURE__ */` comments |
| `manual_pure_functions(callee)` | Treat listed callees as pure       |
| `property_read_side_effects()`  | `PropertyReadSideEffects` policy   |
| `property_write_side_effects()` | Property write policy              |
| `unknown_global_side_effects()` | Unknown global access policy       |

No side effects: literals, resolved identifiers, pure unary/binary/logical, function/arrow expressions. Side effects: calls, assignments, `delete`, `new`, update expressions, property access (per policy).

### Value Type

```rust
expr.value_type(ctx) -> ValueType  // DetermineValueType; ctx: &impl GlobalContext
// Undefined, Null, Number, BigInt, String, Boolean, Object, Undetermined
```

### Operators

| Category | Enum              | Key Methods                                                                                     |
| -------- | ----------------- | ----------------------------------------------------------------------------------------------- |
| Binary   | `BinaryOperator`  | `is_equality()`, `is_arithmetic()`, `compare_inverse_operator()`, `equality_inverse_operator()` |
| Unary    | `UnaryOperator`   | `is_keyword()` (typeof, void, delete)                                                           |
| Logical  | `LogicalOperator` | `Or`, `And`, `Coalesce`, `to_assignment_operator()`                                             |
| Update   | `UpdateOperator`  | `Increment`, `Decrement`                                                                        |

Precedence: `expr.precedence()` or `op.precedence()` (`GetPrecedence`) — used by codegen for parenthesization.

### Control Flow Graph

```rust
let ret = SemanticBuilder::new().with_cfg(true).build(&program);  // cargo feature "cfg"
let cfg = ret.semantic.cfg().expect("built with with_cfg(true)");

cfg.basic_block(block_id).is_unreachable()
cfg.is_reachable(from, to)
cfg.is_cyclic(node)
```

Edge types: `Jump` (conditional branch), `Normal` (sequential), `Backedge` (loop), `NewFunction` (nested function entry), `Finalize` (into `finally`), `Error` (exception path), `Unreachable` (dead code), `Join` (post-finalizer convergence).

---

## Layer 7 — Codegen

Depends on: Layer 2 (AST), optionally `oxc_mangler` output for identifier renaming.

```rust
Codegen::new().build(&program).code                                          // Pretty
Codegen::new().with_options(CodegenOptions::minify()).build(&program).code   // Minified

// Mangled identifiers (oxc_mangler)
let mangled = Mangler::new().build(&program);  // ManglerReturn
Codegen::new()
    .with_scoping(Some(mangled.scoping))
    .with_private_member_mappings(Some(mangled.class_private_mappings))
    .build(&program)
    .code
```

`CodegenReturn`:

| Field            | Type                     | Notes                                           |
| ---------------- | ------------------------ | ----------------------------------------------- |
| `code`           | `String`                 | Generated source                                |
| `map`            | `Option<OwnedSourceMap>` | Feature `sourcemap` + `options.source_map_path` |
| `legal_comments` | `Vec<Comment>`           | From `LegalComment::Linked` / `External`        |

---

## Common Patterns

### Expression Replacement (in exit\_\*)

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
fn exit_statements(&mut self, stmts: &mut Vec<'a, Statement<'a>>, ctx: &mut TraverseCtx<'a, ()>) {
    let mut new_stmts = ctx.ast.vec();
    for stmt in stmts.take_in(ctx.ast) {
        if should_keep(&stmt) { new_stmts.push(stmt); }
    }
    *stmts = new_stmts;
}
```

### Store and Reuse an Expression

```rust
// Reusing while the current Scoping stays valid — keep IDs
self.stored.insert(sym_id, expr.clone_in_with_semantic_ids(ctx.ast.allocator));

// Rebuilding semantics afterwards anyway — plain copy, IDs reset
self.stored.insert(sym_id, expr.clone_in(ctx.ast.allocator));
```

### Value Extraction (handles ParenthesizedExpression)

```rust
fn extract_string<'a>(expr: &'a Expression) -> Option<&'a str> {
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

| #   | Mistake                                   | Fix                                                               |
| --- | ----------------------------------------- | ----------------------------------------------------------------- |
| 1   | `.reference_id()` / `.symbol_id()` early  | Use `.get()?` — direct accessors panic on pre-semantic ASTs       |
| 2   | Stale scoping after structural changes    | Rebuild: `SemanticBuilder::new().build()`                         |
| 3   | Expecting `clone_in` to keep semantic IDs | Use `clone_in_with_semantic_ids` (`.clone()` does not compile)    |
| 4   | Storing `&Expression` references          | Clone into `HashMap<SymbolId, Expression>`                        |
| 5   | Replace in `enter_*`                      | Use `exit_*` (children not yet processed)                         |
| 6   | Manual recursion over ~50 variants        | Use `VisitMut` + `walk_mut::*`                                    |
| 7   | Skip `ParenthesizedExpression`            | Always recurse into parens                                        |
| 8   | Match `Expression::MemberExpression`      | Does not exist — use specific variants                            |
| 9   | `VisitMut` for semantic transforms        | Use `Traverse` (need symbols/ancestors)                           |
| 10  | Reading `semantic.nodes()` by default     | Empty unless `with_build_nodes(true)`                             |
| 11  | `find_binding(scope_id, "name")`          | Takes `Ident` — wrap with `Ident::from` / `static_ident!`         |
| 12  | Leaving `take_in` dummies in the AST      | Replace or remove (`NullLiteral` / `DebuggerStatement` artifacts) |

---

## Safety Summary

**MUST**:

| Rule                                     | Reason                            |
| ---------------------------------------- | --------------------------------- |
| `SPAN` for generated nodes               | Identifies synthetic nodes        |
| Create nodes via `ctx.ast`               | Arena allocation                  |
| Pick the right clone: plain vs semantic  | `clone_in` resets IDs             |
| Dummy cleanup after `take_in`            | Prevents invalid AST              |
| Recycle allocator via `reset()`          | Performance, no unbounded growth  |
| `Traverse` for semantic passes           | Need symbols/ancestors            |
| Transform in `exit_*`                    | Children processed first          |
| `.get()` for ID resolution               | Direct accessors panic            |
| `ContentEq` for AST comparison           | Ignores spans and node IDs        |
| Rebuild scoping after structural changes | Stale IDs corrupt results         |
| Check `diagnostics` + `panicked`         | Recovery parser emits partial AST |

**MUST NOT**:

| Rule                                  | Reason                              |
| ------------------------------------- | ----------------------------------- |
| `Drop` types in arena                 | Compile-time forbidden              |
| Match `MemberExpression` directly     | Does not exist as variant           |
| `VisitMut` for semantic transforms    | No symbol/ancestor access           |
| Hold ancestors across calls           | Stack invalidation                  |
| Assume `Vec` addresses stable         | Unstable after mutation             |
| Read `nodes()` without enabling it    | `with_build_nodes` off by default   |
| Pass `&str` where `Ident` is required | Binding lookups take hashed `Ident` |
