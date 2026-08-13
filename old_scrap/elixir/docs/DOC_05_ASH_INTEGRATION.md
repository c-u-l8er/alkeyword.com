# Document 5: Ash Framework Integration

## Overview

Alkeyword types are first-class Ash resources. Every alkeyform and alkeymatter automatically becomes an Ash resource with full CRUD, GraphQL, REST, and authorization support.

## Overview

Alkeyword types are **first-class Ash resources**. Every `alkeyform` and `alkeymatter` automatically becomes an Ash resource with full CRUD, GraphQL, REST, and authorization support.

## Automatic Resource Generation

```elixir
defmodule MyApp.Types.User do
  use Alkeyword
  use Alkeyword.Ash
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
      reagent role :: String.t()
    end
  end
  
  # Automatically provides:
  # - Ash.Resource implementation
  # - Ecto schema
  # - CRUD actions
  # - GraphQL schema
  # - REST endpoints
  # - Authorization policies
end
```

### Generated Ash Code

Behind the scenes, Alkeyword generates:

```elixir
# Automatically generated (you don't write this)
attributes do
  uuid_primary_key :id
  attribute :name, :string, allow_nil?: false
  attribute :email, :string, allow_nil?: false
  attribute :role, :string, allow_nil?: false
  
  # Observable metadata
  attribute :__alkeyword_created_at__, :utc_datetime_usec
  attribute :__alkeyword_type_name__, :string
  attribute :__alkeyword_version__, :integer
end

actions do
  defaults [:create, :read, :update, :destroy]
  
  create :create_user do
    accept [:name, :email, :role]
  end
  
  read :by_email do
    argument :email, :string, allow_nil?: false
    filter expr(email == ^arg(:email))
  end
end

code_interface do
  define_for MyApp.Api
  define :create, args: [:name, :email, :role]
  define :by_email, args: [:email]
end
```

## Sum Types as GraphQL Unions

`alkeyform` types become GraphQL unions:

```elixir
alkeyform Result do
  phases do
    essence Success, value :: any()
    essence Error, message :: String.t(), code :: integer()
  end
end

# Automatically generates GraphQL:
# union Result = ResultSuccess | ResultError
# 
# type ResultSuccess {
#   value: JSON
# }
#
# type ResultError {
#   message: String!
#   code: Int!
# }
```

### Querying Union Types

```graphql
query GetResults {
  results {
    __typename
    ... on ResultSuccess {
      value
    }
    ... on ResultError {
      message
      code
    }
  }
}
```

## Multi-Tenant Isolation

Alkeyword + Ash provides compile-time multi-tenancy:

```elixir
alkeymatter User do
  elements do
    reagent id :: String.t()
    reagent name :: String.t()
    reagent tenant_id :: String.t()
  end
end

# Ash multi-tenancy
multitenancy do
  strategy :attribute
  attribute :tenant_id
  global? false
end

# Customers can't access each other's data (guaranteed)
MyApp.User
|> Ash.Query.for_read(:read, %{}, tenant: "tenant_123")
|> MyApp.Api.read!()
# Only returns users where tenant_id == "tenant_123"
```

## Authorization Policies

Declarative, type-safe authorization:

```elixir
alkeyform Document do
  phases do
    essence Draft, content :: String.t()
    essence Published, content :: String.t(), published_at :: DateTime.t()
    essence Archived, content :: String.t(), reason :: String.t()
  end
end

policies do
  policy action_type(:read) do
    authorize_if always()
  end
  
  policy action_type(:update) do
    authorize_if expr(__variant__ == :Draft)
    authorize_if actor_attribute_equals(:role, "admin")
  end
  
  policy action_type(:destroy) do
    authorize_if actor_attribute_equals(:role, "admin")
  end
end
```

## REST Endpoints

Automatic REST API generation:

```elixir
# config/config.exs
config :alkeyword_ash,
  enable_rest: true,
  rest_prefix: "/api"

# Automatically provides:
# GET    /api/users
# GET    /api/users/:id
# POST   /api/users
# PATCH  /api/users/:id
# DELETE /api/users/:id
```

### Custom Actions

```elixir
alkeymatter User do
  elements do
    reagent id :: String.t()
    reagent email :: String.t()
    reagent verified :: boolean()
  end
end

actions do
  update :verify_email do
    accept []
    argument :verification_token, :string
    
    change fn changeset, context ->
      if verify_token(context.arguments.verification_token) do
        Ash.Changeset.change_attribute(changeset, :verified, true)
      else
        Ash.Changeset.add_error(changeset, "Invalid token")
      end
    end
  end
end

# POST /api/users/:id/verify_email
# { "verification_token": "abc123" }
```

## Observable Ash Metrics

Track Ash resource performance:

```elixir
Alkeyword.Observatory.get_ash_metrics("User")
#=> %{
#=>   resource: "User",
#=>   total_queries: 12_847,
#=>   avg_query_time_ms: 3.2,
#=>   cache_hit_rate: 87.3,
#=>   graphql_requests: 8_421,
#=>   rest_requests: 4_426,
#=>   policy_checks: 12_847,
#=>   policy_denials: 23
#=> }
```

---

