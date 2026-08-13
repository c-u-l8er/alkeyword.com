# Document 2: Core Concepts

## Overview

Understanding the alchemical metaphor and fundamental concepts behind Alkeyword's type system.

## The Alchemical Metaphor

Alkeyword uses alchemy as a metaphor for type transformation:

| Concept | Alkeyword | Traditional FP | Purpose |
|---------|-----------|----------------|---------|
| **Matter** | `alkeymatter` | Product Type | Structured data |
| **Form** | `alkeyform` | Sum Type | Variant data |
| **Essence** | `essence` | Constructor | Create variants |
| **Synthesis** | `synthesize` | Build pattern | Construct from pattern |
| **Distillation** | `distill` | Match pattern | Extract via pattern |
| **Catalyst** | `catalyst()` | Eager eval | Immediate computation |
| **Vial** | `vial()` | Lazy eval | Deferred computation |
| **Unseal** | `unseal()` | Force eval | Evaluate lazy value |

## Product Types: alkeymatter

**Product types** are structured data with named fields (like structs, records, or objects).

```elixir
alkeymatter Person do
  elements do
    reagent name :: String.t()
    reagent age :: integer()
    reagent email :: String.t()
  end
end
```

**Properties:**
- All fields must be present
- Type-checked at compile time
- Immutable by default
- Observable (tracks creation, updates)

## Sum Types: alkeyform

**Sum types** are variants that can take different forms (like tagged unions or enums).

```elixir
alkeyform PaymentStatus do
  phases do
    essence Pending
    essence Processing, transaction_id :: String.t()
    essence Completed, amount :: float()
    essence Failed, reason :: String.t()
  end
end
```

**Properties:**
- Exactly one variant active at a time
- Exhaustive pattern matching enforced
- Type-safe construction
- Observable (tracks variant distribution)

## Type Safety Guarantees

### Compile-Time Checks

1. **Field Type Validation**
```elixir
user = User.new(%{name: 123})  # ❌ Compile error: expected String.t()
```

2. **Exhaustive Pattern Matching**
```elixir
distill Result, result do
  essence Success, value: v -> v
  # ❌ Compile warning: Missing Error variant
end
```

3. **Variant Field Access**
```elixir
case payment_status do
  essence(Pending) -> 
    # ❌ Compile error if accessing transaction_id
  essence(Processing, transaction_id: id) -> 
    # ✅ Can access id safely
end
```

## Observable Type System

Every type is tracked in real-time:

```elixir
# Query type metadata
Alkeyword.Observatory.get_type_metrics("Result")
#=> %{
#=>   name: "Result",
#=>   kind: :alkeyform,
#=>   variant_count: 2,
#=>   compilation_time_ms: 12.3,
#=>   pattern_coverage: 98.4,
#=>   safety_score: 99.2,
#=>   usage_count: 1247
#=> }
```

## Ash Framework Integration

Alkeyword types are first-class Ash resources:

```elixir
alkeyform Result do
  phases do
    essence Success, value :: any()
    essence Error, message :: String.t()
  end
end

# Automatically provides:
# - Ash.Resource implementation
# - CRUD actions
# - GraphQL schema (union types)
# - REST endpoints
# - Authorization policies
# - Multi-tenant isolation
```

## Philosophy: Transformation Over Mutation

Alkeyword emphasizes **transformation** rather than mutation:

```elixir
# ❌ Mutation (not Alkeyword style)
user.name = "Bob"

# ✅ Transformation (Alkeyword style)
updated_user = %{user | name: "Bob"}

# ✅ Type-safe transformation
new_status = synthesize PaymentStatus, :complete do
  :complete -> essence Completed, amount: 99.99
  :fail -> essence Failed, reason: "Insufficient funds"
end
```

This aligns with:
- Functional programming principles
- BEAM immutability
- Observable change tracking
- Type-safe guarantees

---

