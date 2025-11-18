# Document 15: Migration from Plain Elixir

## Overview

Step-by-step guide for migrating existing Elixir applications to use Alkeyword types. Includes conversion strategies, testing approaches, and gradual adoption patterns.


### Step-by-Step Migration

#### Step 1: Identify Candidates

Look for:
- Structs with multiple variants (→ alkeyform)
- Case statements with pattern matching (→ distill)
- Recursive data structures (→ catalyst/vial)

#### Step 2: Convert Structs to alkeymatter

```elixir
# Before
defmodule User do
  defstruct [:id, :name, :email, :age]
end

# After
defmodule User do
  use Alkeyword
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
      reagent age :: integer()
    end
  end
end
```

#### Step 3: Convert Tagged Tuples to alkeyform

```elixir
# Before
def process(input) do
  case input do
    {:ok, data} -> {:success, data}
    {:error, msg} -> {:failure, msg, 500}
  end
end

# After
alkeyform Result do
  phases do
    essence Success, value :: any()
    essence Error, message :: String.t(), code :: integer()
  end
end

def process(input) do
  synthesize Result, input do
    {:ok, data} -> essence Success, value: data
    {:error, msg} -> essence Error, message: msg, code: 500
  end
end
```

#### Step 4: Add Ash Integration

```elixir
# Add Ash
use Alkeyword.Ash

# Automatic resource generation
attributes do
  # Auto-generated from alkeyform
end

actions do
  defaults [:create, :read, :update, :destroy]
end
```

---

