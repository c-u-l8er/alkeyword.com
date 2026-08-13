# Document 7: Type Inference Engine

## Overview

Alkeyword's type inference engine validates types at compile-time, catching errors before runtime.

## Overview

Alkeyword's type inference engine validates types at compile-time, catching errors before runtime.

## Inference Rules

### Rule 1: Field Type Validation

```elixir
alkeymatter User do
  elements do
    reagent age :: integer()
  end
end

User.new(%{age: "not a number"})
# ❌ Compile error: expected integer(), got: String.t()
```

### Rule 2: Variant Exhaustiveness

```elixir
alkeyform Status do
  phases do
    essence Active
    essence Inactive
    essence Suspended
  end
end

distill Status, status do
  essence Active -> "active"
  essence Inactive -> "inactive"
  # ❌ Compile warning: missing Suspended case
end
```

### Rule 3: Recursive Type Consistency

```elixir
alkeyform Tree do
  phases do
    essence Leaf, value :: integer()
    essence Node,
      left :: catalyst(Tree),
      right :: catalyst(Tree)
  end
end

essence(Node,
  left: catalyst(essence(Leaf, value: "wrong type")),
  right: catalyst(essence(Leaf, value: 2))
)
# ❌ Compile error: Leaf value must be integer()
```

## Type System Internals

### Type AST

Alkeyword builds a type AST at compile-time:

```elixir
# alkeyform Result is compiled to:
%Alkeyword.TypeAST{
  name: "Result",
  kind: :sum,
  variants: [
    %{
      name: "Success",
      fields: [%{name: "value", type: :any}]
    },
    %{
      name: "Error",
      fields: [
        %{name: "message", type: :string},
        %{name: "code", type: :integer}
      ]
    }
  ],
  constraints: [],
  metadata: %{
    file: "lib/my_app/types.ex",
    line: 42
  }
}
```

### Type Checking

```elixir
# At compile-time, Alkeyword validates:
defmodule Alkeyword.TypeChecker do
  def validate_alkeymatter(ast) do
    # 1. All reagents have valid type specs
    # 2. No duplicate field names
    # 3. Type references are defined
    # 4. Circular dependencies are handled
  end
  
  def validate_alkeyform(ast) do
    # 1. All essences have unique names
    # 2. Field types are valid
    # 3. Recursive types use catalyst/vial
    # 4. Pattern matches are exhaustive
  end
end
```

## Observable Type Checking

Track type inference metrics:

```elixir
Alkeyword.Observatory.get_inference_metrics()
#=> %{
#=>   types_checked: 1_247,
#=>   errors_caught: 23,
#=>   warnings_issued: 47,
#=>   avg_check_time_ms: 0.8,
#=>   most_common_errors: [
#=>     "non_exhaustive_pattern",
#=>     "type_mismatch",
#=>     "undefined_type"
#=>   ]
#=> }
```

---

