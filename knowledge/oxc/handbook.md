# OXC Deobfuscation Handbook

> **Version:** 2.0.0 | **Status:** Active | **Updated:** 2026-04-10

Practical patterns for building a high-accuracy JavaScript deobfuscator with OXC in Rust. For API reference, see `reference.md`.

---

## Pipeline Architecture

### Flow

```text
Source Code
    |
    v
[Parser + SemanticBuilder] -> AST + Scoping
    |
    v
[Pre-Formatters] (run once)
    |
    v
[Deobfuscation Loop] (until 0 modifications or max passes)
    |
    v
[Post-Formatters] (run once)
    |
    v
[Codegen] -> Clean JavaScript
```

### Module Trait

```rust
pub trait Module: Send + Sync {
    fn name(&self) -> &'static str;
    fn changes_symbols(&self) -> bool { false }

    fn transform<'a>(
        &mut self,
        allocator: &'a Allocator,
        program: &mut Program<'a>,
        scoping: Scoping,
    ) -> Result<TransformResult>;
}

pub struct TransformResult {
    pub modifications: usize,
    pub scoping: Scoping,
}
```

### Convergence Loop

```rust
let mut passes = 0;
loop {
    let mut total_mods = 0;
    for module in &mut deobfuscators {
        let result = module.transform(allocator, program, scoping)?;
        total_mods += result.modifications;
        scoping = result.scoping;
        if module.changes_symbols() {
            scoping = SemanticBuilder::build(program);
        }
    }
    if total_mods == 0 { break; }
    passes += 1;
    if passes >= 50 { break; }
}
```

### Execution Order

#### Pre-Formatters (once)

| Order | Module            | Reason                                 |
| ----- | ----------------- | -------------------------------------- |
| 1     | IdentifierRenamer | Normalize names for consistent resolution |
| 2     | BraceWrapper      | Consistent block structure             |
| 3     | ForVarHoister     | Hoist loop vars for accurate scoping   |
| 4     | StatementSplitter | Split compound declarations            |
| 5     | MemberSimplifier  | Simplify member access                 |

#### Deobfuscation Loop (repeated until convergence)

| Order | Module                 | Reason                           |
| ----- | ---------------------- | -------------------------------- |
| 1     | StringArrayDecoder     | Earliest — unlocks encoded strings |
| 2     | ConstantPropagator     | Inline literals from decoding    |
| 3     | AliasInliner           | Remove temporary aliases         |
| 4     | MemberSimplifier       | Repeated simplification          |
| 5     | ObjectFlattener        | Inline object properties         |
| 6     | ProxyInliner           | Inline wrapper functions         |
| 7     | IifeOptimizer          | Unwrap IIFEs                     |
| 8     | EvalCallInliner        | Handle encoded eval              |
| 9     | SequenceSimplifier     | Flatten comma expressions        |
| 10    | DeadCodeEliminator     | Remove unused                    |
| 11    | StaticEvaluator        | Evaluate safe expressions        |
| 12    | ControlFlowDeflattener | Latest — linearize control flow  |

**Order rationale**: Decode -> propagate -> simplify -> cleanup -> control flow. Changing order reduces effectiveness.

#### Post-Formatters (once)

Same as pre + StringArrayPostCleaner.

### Scoping Share vs Rebuild

| Situation                    | Action    | Reason                     |
| ---------------------------- | --------- | -------------------------- |
| Read-only / collect-only     | Share     | Same SymbolIds valid       |
| Collect + apply (no decls)   | Share     | Reference counts unchanged |
| Add/remove declarations      | Rebuild   | SymbolIds may invalidate   |
| Rename identifiers           | Rebuild   | Binding lookup changes     |
| Before unused detection      | Rebuild   | Need fresh reference counts |

### changes_symbols() by Module

| Module                 | Value     | Reason                            |
| ---------------------- | --------- | --------------------------------- |
| ConstantPropagator     | `false`   | Read-only, replaces references    |
| StaticEvaluator        | `false`   | Replaces expressions with literals |
| ProxyInliner           | `false`   | Inlines calls, no decl changes    |
| DeadCodeEliminator     | **`true`** | Removes declarations             |
| IdentifierRenamer      | **`true`** | Renames bindings                 |
| StatementSplitter      | **`true`** | Splits declarations              |
| StringArrayDecoder     | **`true`** | Post-cleanup removes array       |
| ControlFlowDeflattener | **`true`** | Restructures control flow        |

---

## Symbol Resolution Patterns

### Safe Resolution (CRITICAL)

Direct `.reference_id()` / `.symbol_id()` **panic** if not set. Always use `.get()`:

```rust
// IdentifierReference -> SymbolId
fn get_reference_symbol(scoping: &Scoping, ident: &IdentifierReference) -> Option<SymbolId> {
    let ref_id = ident.reference_id.get()?;
    scoping.get_reference(ref_id).symbol_id()
}

// BindingIdentifier -> SymbolId
fn get_binding_symbol(binding: &BindingIdentifier) -> Option<SymbolId> {
    binding.symbol_id.get()
}

// VariableDeclarator -> SymbolId
decl.id.get_binding_identifier().and_then(|b| b.symbol_id.get())

// Function -> SymbolId
func.id.as_ref().and_then(|id| id.symbol_id.get())
```

### Reference Counting

```rust
fn count_reads(scoping: &Scoping, id: SymbolId) -> usize {
    scoping.get_resolved_references(id).filter(|r| r.flags().is_read()).count()
}

fn count_writes(scoping: &Scoping, id: SymbolId) -> usize {
    scoping.get_resolved_references(id).filter(|r| r.flags().is_write()).count()
}
```

### Inlining Safety Decision

```text
Want to inline identifier?
  -> Get SymbolId via .get()? -> None = global/unknown -> DO NOT inline
  -> Check writes -> any is_write -> NOT safe (reassigned)
  -> Inline value
  -> Check reads after -> zero reads -> safe to remove declaration
```

Global references: `scoping.root_unresolved_references()`. Do NOT assume constants for globals.

---

## Traversal Patterns

### Two-Pass: Collect then Apply

```rust
impl Module for ConstantPropagator {
    fn transform<'a>(&mut self, allocator: &'a Allocator, program: &mut Program<'a>, scoping: Scoping) -> Result<TransformResult> {
        let mut collector = Collector::default();
        let scoping = traverse_mut(&mut collector, allocator, program, scoping, ());

        if collector.constants.is_empty() {
            return Ok(TransformResult { modifications: 0, scoping });
        }

        let mut inliner = Inliner { constants: collector.constants, modifications: 0 };
        let scoping = traverse_mut(&mut inliner, allocator, program, scoping, ());
        Ok(TransformResult { modifications: inliner.modifications, scoping })
    }
}
```

### Three-Pass: Collect, Apply, Clean

After apply, rebuild scoping, then run cleanup pass to remove now-unused declarations.

### Expression Replacement (in exit_*)

```rust
fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {
    if let Expression::Identifier(ident) = expr {
        if let Some(symbol_id) = get_reference_symbol(ctx.scoping(), ident) {
            if let Some(value) = self.constants.get(&symbol_id) {
                *expr = value.clone_in(ctx.ast.allocator);
                self.modifications += 1;
            }
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
        if should_remove(&stmt) { self.modifications += 1; continue; }
        new_stmts.push(stmt);
    }
    *stmts = new_stmts;
}
```

### Subtree Substitution (VisitMut)

For parameter substitution inside a cloned expression body:

```rust
let mut visitor = ParamSubstituter { subs: &map, alloc: allocator };
visitor.visit_expression(&mut body);  // Auto-recursion via walk_mut
```

---

## Expression and Statement Manipulation

### Value Extraction

```rust
fn extract_string(expr: &Expression) -> Option<&str> {
    match expr {
        Expression::StringLiteral(lit) => Some(lit.value.as_str()),
        Expression::ParenthesizedExpression(p) => extract_string(&p.expression),
        _ => None,
    }
}

fn extract_numeric(expr: &Expression) -> Option<f64> {
    match expr {
        Expression::NumericLiteral(lit) => Some(lit.value),
        Expression::UnaryExpression(u) if u.operator == UnaryOperator::UnaryNegation => {
            extract_numeric(&u.argument).map(|v| -v)
        }
        Expression::ParenthesizedExpression(p) => extract_numeric(&p.expression),
        _ => None,
    }
}
```

### LiteralValue (Compact Storage)

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum LiteralValue {
    String(String),
    Number(f64),
    Boolean(bool),
    Null,
    Undefined,
}
```

Use for constant propagation, string array mapping, static evaluation caching. No lifetimes, cheap clone, hashable.

### CaseValue (Switch Dispatch)

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum CaseValue {
    Number(i64),   // Exact integers for switch dispatch
    String(String),
}
```

Use for control flow deflattening and state machine reconstruction.

### Value Type Decision

| Scenario                   | Use              | Reason                          |
| -------------------------- | ---------------- | ------------------------------- |
| Constant inlining          | `LiteralValue`   | Compact, hashable               |
| String array mapping       | `LiteralValue`   | String-only, fast lookup        |
| Switch case reconstruction | `CaseValue`      | Exact matching, hashable        |
| Proxy function body        | `Expression<'a>` | Full structure for substitution |

### Advanced Statement Patterns

Split multi-declarations:

```rust
for decl in var_decl.declarations.take_in(allocator) {
    new_stmts.push(ctx.ast.statement_variable_declaration(span, kind, ctx.ast.vec1(decl)));
}
```

Block wrapping:

```rust
if !matches!(stmt, Statement::BlockStatement(_)) {
    let inner = stmt.take_in(allocator);
    *stmt = ctx.ast.statement_block(inner.span(), ctx.ast.vec1(inner));
}
```

Unreachable trimming:

```rust
let mut terminated = false;
for stmt in stmts.take_in(allocator) {
    if terminated { self.modifications += 1; continue; }
    if is_terminator(&stmt) { terminated = true; }
    new_stmts.push(stmt);
}
```

---

## Dynamic Evaluation

### JsEvaluator

```rust
pub struct JsEvaluator {
    runtime: JsRuntime,  // deno_core or V8 isolate
    timeout_ms: u64,     // Default 1000ms
}
```

Methods: `execute(code)`, `eval(code) -> JSON`, `eval_to_string()`, `eval_to_number()`.

### Safety Lists

Only evaluate **deterministic, side-effect-free** expressions.

| Category | Safe | Unsafe |
| -------- | ---- | ------ |
| Global functions | `parseInt`, `parseFloat`, `isNaN`, `isFinite`, `encodeURI`, `decodeURI`, `atob`, `btoa`, `Number`, `String`, `Boolean` | `eval`, `Function`, `setTimeout`, `setInterval`, `fetch`, `require` |
| Global objects | `Math`, `JSON`, `Object`, `Array`, `String`, `Number` | `console`, `document`, `window`, `globalThis`, `process` |
| Math methods | All except `random` | `random` |
| String methods | `charAt`, `charCodeAt`, `concat`, `includes`, `indexOf`, `slice`, `split`, `substring`, `toLowerCase`, `toUpperCase`, `trim`, `replace`, `repeat`, `padStart`, `padEnd`, `startsWith`, `endsWith`, `at`, `normalize`, `search`, `matchAll` | — |
| Array methods | `at`, `concat`, `entries`, `every`, `filter`, `find`, `flat`, `flatMap`, `includes`, `indexOf`, `join`, `keys`, `map`, `reduce`, `slice`, `some`, `toString`, `values`, `length` | Mutating: `push`, `pop`, `shift`, `splice`, `sort`, `reverse` |

### Safety Check Structure

```rust
pub fn is_safe_expr(expr: &Expression) -> bool {
    match expr {
        // Literals: always safe
        Expression::StringLiteral(_) | Expression::NumericLiteral(_)
        | Expression::BooleanLiteral(_) | Expression::NullLiteral(_) => true,

        // Identifiers: only safe globals
        Expression::Identifier(id) => SAFE_GLOBALS.contains(&id.name.as_str()),

        // Unary/Binary/Logical: safe if operands safe
        Expression::UnaryExpression(u) => is_safe_unary(u) && is_safe_expr(&u.argument),
        Expression::BinaryExpression(b) => is_safe_expr(&b.left) && is_safe_expr(&b.right),
        Expression::LogicalExpression(l) => is_safe_expr(&l.left) && is_safe_expr(&l.right),

        // Calls: check callee is safe function + all args safe
        Expression::CallExpression(call) => is_safe_call(call),

        // Member access on safe objects
        Expression::StaticMemberExpression(m) => is_safe_member(&m.object, &m.property.name),

        // Recurse into parens, conditionals, sequences
        Expression::ParenthesizedExpression(p) => is_safe_expr(&p.expression),
        Expression::ConditionalExpression(c) => is_safe_expr(&c.test) && is_safe_expr(&c.consequent) && is_safe_expr(&c.alternate),

        _ => false,
    }
}
```

For `is_safe_call`: check callee is either a safe global function, or a method call on a safe object (string literal method, Math method, etc.). Check all arguments are safe.

### Static Evaluation Pattern

```rust
fn exit_expression(&mut self, expr: &mut Expression<'a>, ctx: &mut TraverseCtx<'a, ()>) {
    if expr.is_literal() { return; }
    if !is_safe_expr(expr) { return; }

    let code = expr_to_code(expr);
    let value = self.cache.entry(code.clone()).or_insert_with(|| self.evaluator.eval(&code).ok());

    if let Some(Some(v)) = value {
        if let Some(result) = value_to_expr(v.clone(), ctx.ast.allocator) {
            if !expr.content_eq(&result) {
                *expr = result;
                self.modifications += 1;
            }
        }
    }
}
```

### AST / Code Conversion

```rust
fn expr_to_code(expr: &Expression) -> String {
    Codegen::new().print_expression(expr).into_source_text()
}

fn value_to_expr<'a>(value: serde_json::Value, allocator: &'a Allocator) -> Option<Expression<'a>> {
    let ast = AstBuilder::new(allocator);
    match value {
        serde_json::Value::String(s) => Some(ast.expression_string_literal(SPAN, ast.atom(&s), None)),
        serde_json::Value::Number(n) => n.as_f64().map(|f| ast.expression_numeric_literal(SPAN, f, None, NumberBase::Decimal)),
        serde_json::Value::Bool(b) => Some(ast.expression_boolean_literal(SPAN, b)),
        serde_json::Value::Null => Some(ast.expression_null_literal(SPAN)),
        _ => None,
    }
}
```

---

## Truthiness and Side-Effect Analysis

```rust
fn get_truthiness(expr: &Expression) -> Option<bool> {
    match expr {
        Expression::BooleanLiteral(l) => Some(l.value),
        Expression::NumericLiteral(n) => Some(n.value != 0.0 && !n.value.is_nan()),
        Expression::StringLiteral(s) => Some(!s.value.is_empty()),
        Expression::NullLiteral(_) => Some(false),
        Expression::Identifier(i) if i.name == "undefined" || i.name == "NaN" => Some(false),
        Expression::ArrayExpression(_) | Expression::ObjectExpression(_) => Some(true),
        Expression::FunctionExpression(_) | Expression::ArrowFunctionExpression(_) => Some(true),
        Expression::UnaryExpression(u) if u.operator == UnaryOperator::LogicalNot => {
            get_truthiness(&u.argument).map(|v| !v)
        }
        Expression::UnaryExpression(u) if u.operator == UnaryOperator::Void => Some(false),
        Expression::ParenthesizedExpression(p) => get_truthiness(&p.expression),
        _ => None,
    }
}

fn is_side_effect_free(expr: &Expression) -> bool {
    match expr {
        Expression::NumericLiteral(_) | Expression::StringLiteral(_)
        | Expression::BooleanLiteral(_) | Expression::NullLiteral(_)
        | Expression::Identifier(_) | Expression::ThisExpression(_)
        | Expression::FunctionExpression(_) | Expression::ArrowFunctionExpression(_) => true,
        Expression::UnaryExpression(u) => u.operator != UnaryOperator::Delete && is_side_effect_free(&u.argument),
        Expression::BinaryExpression(b) => is_side_effect_free(&b.left) && is_side_effect_free(&b.right),
        Expression::LogicalExpression(l) => is_side_effect_free(&l.left) && is_side_effect_free(&l.right),
        Expression::ConditionalExpression(c) => is_side_effect_free(&c.test) && is_side_effect_free(&c.consequent) && is_side_effect_free(&c.alternate),
        Expression::ParenthesizedExpression(p) => is_side_effect_free(&p.expression),
        Expression::StaticMemberExpression(m) => is_side_effect_free(&m.object),
        _ => false,
    }
}
```

---

## Core Recipes

### Constant Propagation

Collect: find `const`/`let` with literal init and zero writes. Apply: replace identifier references with cloned literal. Two-pass, `changes_symbols: false`.

### Dead Code Elimination

Single-pass in `exit_statements` and `exit_expression`:

| Target | Action |
| ------ | ------ |
| Statements after terminator (`return`, `throw`, `break`) | Remove |
| Empty/debugger statements | Remove |
| Unused variable declarations (zero reads, no side-effect init) | Remove |
| Unused function declarations (zero reads) | Remove |
| `if(true) { body }` | Unwrap body |
| `if(false) { body } else { alt }` | Unwrap alt |
| `true ? a : b` | Replace with `a` |
| `true \|\| x` | Replace with `true` |
| `false && x` | Replace with `false` |

`changes_symbols: true`. Rebuild scoping before running.

### Proxy Function Inlining

Detect: single-return functions (`function f(a,b) { return expr(a,b); }`). Collect: map function SymbolId to param names + body expression. Apply: replace calls with cloned body, substitute params via VisitMut. `changes_symbols: false`.

### String Array Decoding

Detect: large string array + decoder function (`function _0x123(x) { return arr[x]; }`). Map index to string. Replace calls with literal strings. Post-cleanup removes array. Run **earliest** in loop.

### Object Property Flattening

Detect: object literal assigned to variable with no mutations. Collect properties. Replace member access with property values. Safety: object not reassigned, properties not computed.

### Control Flow Deflattening

Detect: `while(true) { switch(state) { case X: ... state = Y; break; } }`. Map case values to block sequences. Reconstruct linear flow. Replace dispatcher. Run **latest** in loop.

---

## Anti-Patterns

| #  | Mistake | Symptom | Fix |
| -- | ------- | ------- | --- |
| 1  | Direct `.reference_id()` / `.symbol_id()` | Panic | Use `.get()?` |
| 2  | Stale scoping after removing declarations | Wrong reference counts | Rebuild with `SemanticBuilder::build()` |
| 3  | Standard `.clone()` on AST nodes | Lifetime errors | `clone_in(ctx.ast.allocator)` |
| 4  | Storing `&Expression<'a>` references | Dangling pointers | Clone into `FxHashMap<SymbolId, Expression<'a>>` |
| 5  | Replacing in `enter_*` | Skips child processing | Replace in `exit_*` |
| 6  | Manual AST recursion (match all ~50 variants) | Incomplete, unmaintainable | Use `VisitMut` + `walk_mut::*` |
| 7  | Forgetting `ParenthesizedExpression` in extraction | Misses `(("string"))` | Always recurse into parens |
| 8  | Matching `Expression::MemberExpression` | Compile error | Use `StaticMemberExpression`, `ComputedMemberExpression` |
| 9  | Forgetting `self.modifications += 1` | Convergence never triggers | Always count changes |
| 10 | No-op modification counting | Infinite loop | Guard with `content_eq` before replacing |

---

## Modification Counting

**MUST** increment `modifications` on every actual change:

```rust
if !expr.content_eq(&new_expr) {
    *expr = new_expr;
    self.modifications += 1;
}
```

Enables convergence detection. Forgetting causes either no convergence (missed count) or infinite loop (counting no-ops).

---

## Recommended File Structure

```text
src/
├── main.rs
├── core/
│   ├── module.rs          # Module trait + TransformResult
│   ├── registry.rs        # ModuleRegistry
│   └── engine.rs          # Convergence loop
├── utils/
│   ├── ast.rs             # AstBuilder helpers
│   ├── symbols.rs         # Safe resolution functions
│   ├── literal.rs         # LiteralValue, extraction, conversion
│   └── eval.rs            # expr_to_code, value_to_expr
├── eval/
│   ├── runtime.rs         # JsEvaluator (V8/deno_core)
│   └── safety.rs          # is_safe_expr
├── modules/
│   ├── formatters/        # Pre/post formatters
│   └── transformers/      # Deobfuscation passes
└── visitors/
    └── param_substituter.rs  # VisitMut for proxy inlining
```
