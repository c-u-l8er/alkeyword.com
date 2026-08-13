# Document 1: Getting Started (15 Minutes)

## Overview

Alkeyword brings type-safe algebraic data types to Elixir with an alchemical DSL. This guide gets you from zero to your first type definition in 15 minutes.

## Overview

Alkeyword brings type-safe algebraic data types to Elixir with an alchemical DSL. This guide gets you from zero to your first type definition in 15 minutes.

## Prerequisites

- Elixir 1.14+
- Phoenix 1.7+ (optional, for LiveView features)
- PostgreSQL 14+ (for Observable features)

## Installation

### Step 1: Add Dependencies

```elixir
# mix.exs
def deps do
  [
    {:alkeyword, "~> 1.0"},
    {:alkeyword_ash, "~> 1.0"},         # Ash Framework integration
    {:alkeyword_observable, "~> 1.0"},  # Real-time observability
    {:alkeyword_phoenix, "~> 1.0"}      # LiveView components
  ]
end
```

### Step 2: Install

```bash
mix deps.get
mix alkeyword.install
```

This creates:
- Database migrations for type metadata
- Configuration files
- Dashboard routes (if Phoenix is present)

### Step 3: Configure

```elixir
# config/config.exs
config :alkeyword,
  repo: MyApp.Repo,
  enable_observable: true,
  dashboard_path: "/alkeyword"

config :alkeyword_ash,
  enable_graphql: true,
  enable_rest: true
```

### Step 4: Run Migrations

```bash
mix ecto.migrate
```

## Your First Type

### Product Type (alkeymatter)

```elixir
defmodule MyApp.Types.User do
  use Alkeyword
  use Alkeyword.Ash
  use Alkeyword.Observable
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
      reagent created_at :: DateTime.t()
    end
  end
  
  # Ash resource automatically generated
  attributes do
    attribute :id, :string
    attribute :name, :string
    attribute :email, :string
    attribute :created_at, :datetime
  end
  
  actions do
    defaults [:create, :read, :update, :destroy]
  end
end
```

### Sum Type (alkeyform)

```elixir
defmodule MyApp.Types.Result do
  use Alkeyword
  use Alkeyword.Ash
  use Alkeyword.Observable
  
  alkeyform Result do
    phases do
      essence Success, value :: any()
      essence Error, message :: String.t(), code :: integer()
    end
  end
  
  # Automatic Ash resource with variant tracking
  attributes do
    attribute :__variant__, :atom
    attribute :value, :map
    attribute :message, :string
    attribute :code, :integer
  end
  
  # GraphQL union type auto-generated
  graphql do
    type :result_union
  end
end
```

## Using Your Types

```elixir
# Create instances
user = User.new(%{
  id: "u123",
  name: "Alice",
  email: "alice@example.com",
  created_at: DateTime.utc_now()
})

# Type-safe construction
result = synthesize Result, {:ok, user} do
  {:ok, data} -> essence Success, value: data
  {:error, msg} -> essence Error, message: msg, code: 500
end

# Pattern matching
message = distill Result, result do
  essence Success, value: user -> 
    "Welcome, #{user.name}!"
  essence Error, message: msg, code: code -> 
    "Error #{code}: #{msg}"
end
```

## View the Dashboard

```bash
mix phx.server
```

Navigate to `http://localhost:4000/alkeyword/dashboard`

You'll see:
- **Types Defined**: Real-time count
- **Pattern Match Coverage**: % exhaustive
- **Compilation Metrics**: Average time
- **Safety Scores**: Per-type analysis
- **Live Type Browser**: Explore all defined types

## Next Steps

- Read [Core Concepts](#2-core-concepts) to understand the philosophy
- Learn [Pattern Matching](#3-pattern-matching) for exhaustive matching
- Explore [Ash Integration](#5-ash-framework-integration) for APIs
- Set up [Monitoring](#13-monitoring--observability) for production

---

