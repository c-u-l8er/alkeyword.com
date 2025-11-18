# Document 9: GraphQL Integration

## Overview

Alkeyword types automatically generate GraphQL schemas through Ash Framework integration. Sum types (`alkeyform`) become GraphQL unions, product types (`alkeymatter`) become object types, and all CRUD operations are exposed via type-safe APIs.

## Automatic Schema Generation

### Product Types → GraphQL Objects

```elixir
defmodule MyApp.Types.User do
  use Alkeyword
  use Alkeyword.Ash
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
      reagent age :: integer()
      reagent created_at :: DateTime.t()
    end
  end
  
  # Automatically generates GraphQL:
  # type User {
  #   id: String!
  #   name: String!
  #   email: String!
  #   age: Int!
  #   createdAt: DateTime!
  # }
end
```

### Sum Types → GraphQL Unions

```elixir
defmodule MyApp.Types.Result do
  use Alkeyword
  use Alkeyword.Ash
  
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
end
```

### Complex Sum Types

```elixir
alkeyform PaymentStatus do
  phases do
    essence Pending
    essence Processing, transaction_id :: String.t(), started_at :: DateTime.t()
    essence Completed, 
      transaction_id :: String.t(), 
      amount :: float(), 
      completed_at :: DateTime.t()
    essence Failed, reason :: String.t(), error_code :: String.t()
    essence Refunded, 
      original_transaction_id :: String.t(),
      refund_amount :: float(),
      refunded_at :: DateTime.t()
  end
end

# Generates:
# union PaymentStatus = 
#   | PaymentStatusPending
#   | PaymentStatusProcessing
#   | PaymentStatusCompleted
#   | PaymentStatusFailed
#   | PaymentStatusRefunded
#
# type PaymentStatusPending {
#   _variant: String!
# }
#
# type PaymentStatusProcessing {
#   transactionId: String!
#   startedAt: DateTime!
# }
# 
# (etc for other variants)
```

## Queries

### Basic Queries

Alkeyword + Ash automatically generates queries:

```graphql
query {
  # List all users
  users {
    id
    name
    email
    age
  }
  
  # Get single user
  user(id: "user_123") {
    id
    name
    email
  }
  
  # Filter users
  users(filter: { age: { greaterThan: 21 } }) {
    id
    name
    age
  }
}
```

### Union Type Queries

```graphql
query {
  # Query returns union type
  processPayment(amount: 100.0) {
    __typename
    
    ... on PaymentStatusPending {
      _variant
    }
    
    ... on PaymentStatusProcessing {
      transactionId
      startedAt
    }
    
    ... on PaymentStatusCompleted {
      transactionId
      amount
      completedAt
    }
    
    ... on PaymentStatusFailed {
      reason
      errorCode
    }
  }
}
```

### Nested Queries

```elixir
alkeyform ApiResponse do
  phases do
    essence Success, 
      data :: alkeymatter(ResponseData),
      metadata :: alkeymatter(ResponseMeta)
    essence Error, error :: alkeyform(ErrorDetail)
  end
end

alkeyform ErrorDetail do
  phases do
    essence NetworkError, reason :: String.t()
    essence ValidationError, fields :: list()
    essence ServerError, code :: integer(), message :: String.t()
  end
end
```

```graphql
query {
  makeApiCall(endpoint: "/api/users") {
    __typename
    
    ... on ApiResponseSuccess {
      data {
        # Nested product type
        users {
          id
          name
        }
      }
      metadata {
        requestId
        duration
      }
    }
    
    ... on ApiResponseError {
      error {
        __typename
        
        # Nested sum type
        ... on ErrorDetailNetworkError {
          reason
        }
        
        ... on ErrorDetailValidationError {
          fields
        }
        
        ... on ErrorDetailServerError {
          code
          message
        }
      }
    }
  }
}
```

## Mutations

### Creating Records

```graphql
mutation {
  createUser(
    input: {
      name: "Alice"
      email: "alice@example.com"
      age: 30
    }
  ) {
    result {
      id
      name
      email
    }
    errors {
      message
      field
    }
  }
}
```

### Updating Records

```graphql
mutation {
  updateUser(
    id: "user_123"
    input: {
      name: "Alice Smith"
      age: 31
    }
  ) {
    result {
      id
      name
      age
    }
    errors {
      message
    }
  }
}
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

graphql do
  mutations do
    update :verify_email
  end
end
```

```graphql
mutation {
  verifyUserEmail(
    id: "user_123"
    verificationToken: "abc123xyz"
  ) {
    result {
      id
      email
      verified
    }
    errors {
      message
    }
  }
}
```

## Type-Safe Resolvers

### Automatic Resolvers

Alkeyword generates type-safe resolvers:

```elixir
# This is automatic - you don't write this
defmodule MyAppWeb.Schema.UserResolvers do
  def list_users(_parent, args, _resolution) do
    # Type-safe: args validated against alkeyform definition
    User
    |> Ash.Query.for_read(:read, args)
    |> MyApp.Api.read()
  end
  
  def get_user(_parent, %{id: id}, _resolution) do
    # Type-safe: id must be String.t() per alkeyform
    User
    |> Ash.Query.for_read(:by_id, %{id: id})
    |> MyApp.Api.read_one()
  end
end
```

### Custom Resolvers

```elixir
alkeyform SearchResult do
  phases do
    essence Found, 
      users :: list(),
      total :: integer(),
      page :: integer()
    essence NoResults, query :: String.t()
    essence Error, message :: String.t()
  end
end

# Custom resolver
defmodule MyAppWeb.Schema.SearchResolvers do
  use Alkeyword
  
  def search_users(_parent, %{query: query, page: page}, _resolution) do
    case MyApp.Search.search_users(query, page) do
      {:ok, users, total} when length(users) > 0 ->
        {:ok, essence(Found, users: users, total: total, page: page)}
        
      {:ok, [], _total} ->
        {:ok, essence(NoResults, query: query)}
        
      {:error, reason} ->
        {:ok, essence(Error, message: reason)}
    end
  end
end

# Schema
graphql do
  queries do
    field :search_users, :search_result do
      arg :query, non_null(:string)
      arg :page, :integer, default_value: 1
      resolve &MyAppWeb.Schema.SearchResolvers.search_users/3
    end
  end
end
```

## Subscriptions

### Real-Time Updates

```elixir
alkeyform UserEvent do
  phases do
    essence UserCreated, user :: alkeymatter(User)
    essence UserUpdated, user :: alkeymatter(User), changes :: map()
    essence UserDeleted, user_id :: String.t()
  end
end

# Subscription
graphql do
  subscriptions do
    subscribe :user_events do
      config fn _args, _context ->
        {:ok, topic: "user_events"}
      end
      
      trigger :create_user, 
        topic: fn user -> "user_events" end
      
      trigger :update_user,
        topic: fn user -> "user_events" end
      
      trigger :delete_user,
        topic: fn user -> "user_events" end
    end
  end
end
```

```graphql
subscription {
  userEvents {
    __typename
    
    ... on UserEventUserCreated {
      user {
        id
        name
        email
      }
    }
    
    ... on UserEventUserUpdated {
      user {
        id
        name
      }
      changes
    }
    
    ... on UserEventUserDeleted {
      userId
    }
  }
}
```

## Input Types

### Automatic Input Generation

```elixir
alkeymatter CreateUserInput do
  elements do
    reagent name :: String.t()
    reagent email :: String.t()
    reagent age :: integer()
    reagent role :: String.t()
  end
end

# Generates:
# input CreateUserInput {
#   name: String!
#   email: String!
#   age: Int!
#   role: String!
# }
```

### Validation

```elixir
alkeymatter CreateUserInput do
  elements do
    reagent name :: String.t()
    reagent email :: String.t()
    reagent age :: integer()
  end
end

# Add Ash validations
validations do
  validate string_length(:name, min: 2, max: 100)
  validate match(:email, ~r/@/)
  validate compare(:age, greater_than_or_equal_to: 18)
end

# GraphQL returns validation errors
# {
#   "errors": [
#     {
#       "message": "age must be greater than or equal to 18",
#       "field": "age"
#     }
#   ]
# }
```

## Dataloader Integration

Efficient N+1 query prevention:

```elixir
alkeymatter Post do
  elements do
    reagent id :: String.t()
    reagent title :: String.t()
    reagent author_id :: String.t()
  end
end

relationships do
  belongs_to :author, User do
    source_attribute :author_id
    destination_attribute :id
  end
  
  has_many :comments, Comment do
    destination_attribute :post_id
  end
end

# Automatic dataloader setup
graphql do
  queries do
    field :posts, list_of(:post) do
      # Dataloader automatically batches author queries
      resolve dataloader(MyApp.Repo)
    end
  end
end
```

```graphql
query {
  posts {
    id
    title
    # Single batch query for all authors
    author {
      id
      name
    }
    # Single batch query for all comments
    comments {
      id
      content
    }
  }
}

# Without Dataloader: 1 + N + M queries
# With Dataloader: 3 queries (posts, authors, comments)
```

## Error Handling

### Type-Safe Errors

```elixir
alkeyform MutationResult do
  phases do
    essence Success, data :: any()
    essence ValidationError, errors :: list()
    essence AuthorizationError, message :: String.t()
    essence NotFoundError, resource :: String.t(), id :: String.t()
  end
end

def create_user(args) do
  case User.create(args) do
    {:ok, user} ->
      {:ok, essence(Success, data: user)}
      
    {:error, %Ash.Error.Invalid{errors: errors}} ->
      {:ok, essence(ValidationError, errors: format_errors(errors))}
      
    {:error, %Ash.Error.Forbidden{}} ->
      {:ok, essence(AuthorizationError, message: "Unauthorized")}
  end
end
```

```graphql
mutation {
  createUser(input: { name: "", email: "invalid" }) {
    __typename
    
    ... on MutationResultSuccess {
      data {
        id
        name
      }
    }
    
    ... on MutationResultValidationError {
      errors {
        field
        message
      }
    }
    
    ... on MutationResultAuthorizationError {
      message
    }
  }
}
```

## Pagination

### Relay-Style Pagination

```elixir
alkeyform UserConnection do
  phases do
    essence Connection,
      edges :: list(),
      page_info :: alkeymatter(PageInfo),
      total_count :: integer()
  end
end

alkeymatter PageInfo do
  elements do
    reagent has_next_page :: boolean()
    reagent has_previous_page :: boolean()
    reagent start_cursor :: String.t()
    reagent end_cursor :: String.t()
  end
end

graphql do
  queries do
    field :users, :user_connection do
      arg :first, :integer
      arg :after, :string
      
      resolve &MyAppWeb.Resolvers.list_users_paginated/3
    end
  end
end
```

```graphql
query {
  users(first: 10, after: "cursor_123") {
    edges {
      node {
        id
        name
        email
      }
      cursor
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      startCursor
      endCursor
    }
    totalCount
  }
}
```

## Observable GraphQL Metrics

### Query Performance Tracking

```elixir
Alkeyword.Observatory.get_graphql_metrics()
#=> %{
#=>   total_queries: 12_847,
#=>   total_mutations: 3_421,
#=>   total_subscriptions: 847,
#=>   avg_query_time_ms: 23.4,
#=>   avg_mutation_time_ms: 45.7,
#=>   slowest_queries: [
#=>     %{query: "searchUsers", time_ms: 247.3, count: 12},
#=>     %{query: "listPosts", time_ms: 189.4, count: 47}
#=>   ],
#=>   most_common_errors: [
#=>     %{type: "ValidationError", count: 234},
#=>     %{type: "NotFoundError", count: 89}
#=>   ]
#=> }
```

### Type Usage Statistics

```elixir
Alkeyword.Observatory.get_graphql_type_usage("Result")
#=> %{
#=>   type: "Result",
#=>   query_count: 8_421,
#=>   variant_distribution: %{
#=>     "Success" => 7_847,  # 93.2%
#=>     "Error" => 574       # 6.8%
#=>   },
#=>   avg_response_time_ms: 12.3
#=> }
```

## Testing GraphQL APIs

### Query Testing

```elixir
defmodule MyAppWeb.Schema.UserQueriesTest do
  use MyAppWeb.ConnCase
  use Alkeyword
  
  test "list users returns correct type", %{conn: conn} do
    # Create test data
    {:ok, user1} = User.create(%{name: "Alice", email: "alice@example.com"})
    {:ok, user2} = User.create(%{name: "Bob", email: "bob@example.com"})
    
    # Query
    query = """
    query {
      users {
        id
        name
        email
      }
    }
    """
    
    result = conn
    |> post("/graphql", %{query: query})
    |> json_response(200)
    
    # Assert
    assert %{"data" => %{"users" => users}} = result
    assert length(users) == 2
    assert Enum.any?(users, &(&1["name"] == "Alice"))
  end
  
  test "search returns union type", %{conn: conn} do
    query = """
    query {
      searchUsers(query: "nonexistent") {
        __typename
        ... on SearchResultNoResults {
          query
        }
      }
    }
    """
    
    result = conn
    |> post("/graphql", %{query: query})
    |> json_response(200)
    
    assert %{
      "data" => %{
        "searchUsers" => %{
          "__typename" => "SearchResultNoResults",
          "query" => "nonexistent"
        }
      }
    } = result
  end
end
```

### Mutation Testing

```elixir
test "create user with validation error", %{conn: conn} do
  mutation = """
  mutation {
    createUser(input: { name: "", email: "invalid" }) {
      __typename
      ... on MutationResultSuccess {
        data {
          id
        }
      }
      ... on MutationResultValidationError {
        errors {
          field
          message
        }
      }
    }
  }
  """
  
  result = conn
  |> post("/graphql", %{query: mutation})
  |> json_response(200)
  
  assert %{
    "data" => %{
      "createUser" => %{
        "__typename" => "MutationResultValidationError",
        "errors" => errors
      }
    }
  } = result
  
  assert Enum.any?(errors, &(&1["field"] == "name"))
  assert Enum.any?(errors, &(&1["field"] == "email"))
end
```

## GraphQL Playground

Alkeyword provides a GraphQL playground:

```elixir
# router.ex
scope "/" do
  pipe_through :browser
  
  forward "/graphql", Absinthe.Plug.GraphiQL,
    schema: MyAppWeb.Schema,
    interface: :playground
end
```

Visit `http://localhost:4000/graphql` for interactive API exploration with:
- Auto-generated documentation
- Type introspection
- Query builder
- Real-time subscriptions testing

## Performance Best Practices

### 1. Use Dataloader

```elixir
# ❌ N+1 queries
query {
  posts {
    author { name }  # N queries
  }
}

# ✅ Batched queries
graphql do
  queries do
    field :posts, list_of(:post) do
      resolve dataloader(MyApp.Repo)
    end
  end
end
```

### 2. Limit Depth

```elixir
# config/config.exs
config :absinthe,
  max_complexity: 200,
  max_depth: 10
```

### 3. Cache Results

```elixir
graphql do
  queries do
    field :expensive_query, :result do
      # Cache for 60 seconds
      middleware MyApp.CacheMiddleware, ttl: 60
      resolve &MyAppWeb.Resolvers.expensive_query/3
    end
  end
end
```

## Summary

Alkeyword + Ash provides:
- ✅ Automatic GraphQL schema generation
- ✅ Type-safe union types from alkeyforms
- ✅ Efficient dataloader integration
- ✅ Real-time subscriptions
- ✅ Observable query metrics
- ✅ Built-in validation
- ✅ Production-ready error handling

**GraphQL API generation: Zero boilerplate, 100% type-safe.**

---

**Next:** [Document 10: Multi-tenant Patterns](#document-10-multi-tenant-patterns)
