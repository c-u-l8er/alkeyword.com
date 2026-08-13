# Alkeyword LLM Coding Prompt Guide

## System Prompt for Code Assistants

When coding with Alkeyword, an Elixir DSL for algebraic data types with alchemical terminology, use this comprehensive guide.

---

## Core Concepts

**Alkeyword** is an Elixir library that provides type-safe algebraic data types through an alchemical metaphor. It replaces traditional Elixir structs and case statements with expressive, compile-time checked constructs.

### Philosophy
- **alkeymatter** = Product types (structured data with multiple fields)
- **alkeyform** = Sum types (variants that can take different forms)
- **Alchemy** = Transformation and type-safe manipulation of data

---

## Complete Syntax Reference

### 1. Product Types: `alkeymatter`

Product types define structured data with named fields (like structs).

```elixir
alkeymatter TypeName do
  elements do
    reagent field_name :: type_spec()
    reagent another_field :: type_spec()
  end
end
```

**Example:**
```elixir
alkeymatter User do
  elements do
    reagent id :: String.t()
    reagent name :: String.t()
    reagent email :: String.t()
    reagent age :: integer()
    reagent active :: boolean()
  end
end

# Usage
user = User.new(%{
  id: "123",
  name: "Alice",
  email: "alice@example.com",
  age: 30,
  active: true
})
```

### 2. Sum Types: `alkeyform`

Sum types define variants (like tagged unions or enums with data).

```elixir
alkeyform TypeName do
  phases do
    essence VariantName
    essence VariantWithData, field :: type_spec()
    essence AnotherVariant, field1 :: type(), field2 :: type()
  end
end
```

**Example:**
```elixir
alkeyform Result do
  phases do
    essence Success, value :: any()
    essence Error, message :: String.t(), code :: integer()
  end
end

alkeyform PaymentStatus do
  phases do
    essence Pending
    essence Processing, transaction_id :: String.t()
    essence Completed, transaction_id :: String.t(), amount :: float()
    essence Failed, reason :: String.t()
  end
end
```

### 3. Constructing Variants: `essence`

Use `essence` to create instances of sum type variants.

```elixir
# Without fields
empty_status = essence(Pending)

# With fields
success = essence(Success, value: "data")
error = essence(Error, message: "Not found", code: 404)
```

### 4. Type-Safe Construction: `synthesize`

Use `synthesize` to build variants based on pattern matching input.

```elixir
synthesize TypeName, input_value do
  pattern1 -> essence VariantName, field: value
  pattern2 -> essence OtherVariant, field: value
  _ -> essence DefaultVariant
end
```

**Example:**
```elixir
result = synthesize Result, response do
  {:ok, data} -> essence Success, value: data
  {:error, reason} -> essence Error, message: reason, code: 500
end

status = synthesize PaymentStatus, transaction do
  %{state: "pending"} -> essence Pending
  %{state: "processing", id: id} -> essence Processing, transaction_id: id
  %{state: "completed", id: id, amt: amt} -> 
    essence Completed, transaction_id: id, amount: amt
  %{state: "failed", error: err} -> essence Failed, reason: err
end
```

### 5. Pattern Matching: `distill`

Use `distill` to exhaustively pattern match on sum types.

```elixir
distill TypeName, value do
  essence VariantName -> result
  essence VariantWithData, field: var -> result_using_var
  essence AnotherVariant, field1: x, field2: y -> result_using_x_and_y
end
```

**Example:**
```elixir
message = distill Result, result do
  essence Success, value: data -> 
    "Operation succeeded: #{inspect(data)}"
  essence Error, message: msg, code: code -> 
    "Error #{code}: #{msg}"
end

action = distill PaymentStatus, status do
  essence Pending -> 
    {:wait, "Transaction is pending"}
  essence Processing, transaction_id: id -> 
    {:track, id}
  essence Completed, transaction_id: id, amount: amt -> 
    {:receipt, id, amt}
  essence Failed, reason: reason -> 
    {:retry, reason}
end
```

### 6. Eager Recursion: `catalyst`

Use `catalyst` for eager evaluation of recursive structures (computed immediately).

```elixir
alkeyform BinaryTree do
  phases do
    essence Empty
    essence Leaf, value :: any()
    essence Node,
      left :: catalyst(BinaryTree),
      right :: catalyst(BinaryTree),
      data :: any()
  end
end

# Build tree - all nodes computed immediately
tree = essence(Node,
  left: catalyst(essence(Leaf, value: 1)),
  right: catalyst(essence(Node,
    left: catalyst(essence(Leaf, value: 2)),
    right: catalyst(essence(Leaf, value: 3)),
    data: "subtree"
  )),
  data: "root"
)

# Access without function calls
left_value = tree[:left][:value]  # => 1
```

**When to use catalysts:**
- ✅ Small, bounded data structures
- ✅ Frequently accessed data
- ✅ Predictable memory usage
- ✅ Performance-critical paths

### 7. Lazy Recursion: `vial`

Use `vial` for lazy evaluation (computed on demand).

```elixir
alkeyform LazyStream do
  phases do
    essence Empty
    essence Cons,
      head :: any(),
      tail :: vial(LazyStream)
  end
end

# Build lazy stream - tail computed on demand
stream = essence(Cons,
  head: 1,
  tail: vial(fn ->
    essence(Cons,
      head: 2,
      tail: vial(fn -> 
        essence(Cons,
          head: 3,
          tail: vial(fn -> essence(Empty) end)
        )
      end)
    )
  end)
)

# Must unseal to access
tail_stream = unseal(stream[:tail])
second_value = tail_stream[:head]  # => 2
```

**When to use vials:**
- ✅ Large or infinite data structures
- ✅ Expensive computations
- ✅ Infrequently accessed data
- ✅ Streaming scenarios

### 8. Force Evaluation: `unseal`

Use `unseal` to evaluate vials (lazy values).

```elixir
lazy_value = vial(fn -> expensive_computation() end)
actual_value = unseal(lazy_value)
```

---

## Common Patterns

### Pattern 1: API Response Handling

```elixir
alkeyform ApiResponse do
  phases do
    essence Success, data :: map(), status :: integer()
    essence Redirect, url :: String.t(), status :: integer()
    essence ClientError, message :: String.t(), status :: integer()
    essence ServerError, message :: String.t(), status :: integer()
    essence NetworkError, reason :: String.t()
  end
end

def handle_api_call(url, params) do
  case HTTPClient.get(url, params) do
    {:ok, %{status: status, body: body}} when status in 200..299 ->
      essence Success, data: body, status: status
      
    {:ok, %{status: status, headers: headers}} when status in 300..399 ->
      essence Redirect, url: headers["location"], status: status
      
    {:ok, %{status: status, body: body}} when status in 400..499 ->
      essence ClientError, message: body["error"], status: status
      
    {:ok, %{status: status, body: body}} when status >= 500 ->
      essence ServerError, message: body["error"], status: status
      
    {:error, reason} ->
      essence NetworkError, reason: inspect(reason)
  end
end

def process_response(response) do
  distill ApiResponse, response do
    essence Success, data: data, status: _ ->
      {:ok, transform_data(data)}
      
    essence Redirect, url: url, status: _ ->
      {:redirect, url}
      
    essence ClientError, message: msg, status: code ->
      {:error, "Client error #{code}: #{msg}"}
      
    essence ServerError, message: msg, status: code ->
      {:error, "Server error #{code}: #{msg}"}
      
    essence NetworkError, reason: reason ->
      {:error, "Network failed: #{reason}"}
  end
end
```

### Pattern 2: State Machines

```elixir
alkeyform OrderState do
  phases do
    essence Draft, items :: list(), customer :: String.t()
    essence Submitted, 
      order_id :: String.t(), 
      timestamp :: DateTime.t(),
      items :: list()
    essence Processing, order_id :: String.t()
    essence Shipped, 
      order_id :: String.t(), 
      tracking :: String.t(),
      carrier :: String.t()
    essence Delivered, 
      order_id :: String.t(), 
      signature :: String.t(),
      delivered_at :: DateTime.t()
    essence Cancelled, 
      order_id :: String.t(),
      reason :: String.t()
  end
end

def transition_order(state, event) do
  synthesize OrderState, {state, event} do
    {essence(Draft, items: items, customer: cust), :submit} ->
      order_id = generate_order_id()
      essence Submitted, 
        order_id: order_id,
        timestamp: DateTime.utc_now(),
        items: items
        
    {essence(Submitted, order_id: id, items: _), :process} ->
      essence Processing, order_id: id
      
    {essence(Processing, order_id: id), {:ship, tracking, carrier}} ->
      essence Shipped, 
        order_id: id, 
        tracking: tracking,
        carrier: carrier
        
    {essence(Shipped, order_id: id), {:deliver, signature}} ->
      essence Delivered,
        order_id: id,
        signature: signature,
        delivered_at: DateTime.utc_now()
        
    {essence(Draft, items: _, customer: _), {:cancel, reason}} ->
      essence Cancelled, order_id: "none", reason: reason
      
    {state, :cancel} ->
      order_id = extract_order_id(state)
      essence Cancelled, order_id: order_id, reason: "User cancelled"
  end
end
```

### Pattern 3: Recursive Data Structures

```elixir
# JSON-like structure
alkeyform JsonValue do
  phases do
    essence Null
    essence Bool, value :: boolean()
    essence Number, value :: number()
    essence String, value :: String.t()
    essence Array, elements :: list()
    essence Object, fields :: map()
  end
end

def parse_json(input) do
  synthesize JsonValue, input do
    nil -> essence Null
    bool when is_boolean(bool) -> essence Bool, value: bool
    num when is_number(num) -> essence Number, value: num
    str when is_binary(str) -> essence String, value: str
    list when is_list(list) -> 
      essence Array, elements: Enum.map(list, &parse_json/1)
    map when is_map(map) -> 
      essence Object, fields: Map.new(map, fn {k, v} -> 
        {k, parse_json(v)} 
      end)
  end
end

def stringify(json_value) do
  distill JsonValue, json_value do
    essence Null -> "null"
    essence Bool, value: true -> "true"
    essence Bool, value: false -> "false"
    essence Number, value: num -> to_string(num)
    essence String, value: str -> "\"#{str}\""
    essence Array, elements: elements ->
      "[#{Enum.map_join(elements, ", ", &stringify/1)}]"
    essence Object, fields: fields ->
      pairs = Enum.map_join(fields, ", ", fn {k, v} ->
        "\"#{k}\": #{stringify(v)}"
      end)
      "{#{pairs}}"
  end
end
```

### Pattern 4: Option/Maybe Type

```elixir
alkeyform Option do
  phases do
    essence Some, value :: any()
    essence None
  end
end

def safe_divide(a, b) do
  synthesize Option, b do
    0 -> essence None
    divisor -> essence Some, value: a / divisor
  end
end

def get_user_email(user_id) do
  case Database.find_user(user_id) do
    nil -> essence None
    user -> essence Some, value: user.email
  end
end

def unwrap_or(option, default) do
  distill Option, option do
    essence Some, value: v -> v
    essence None -> default
  end
end
```

### Pattern 5: Tree Structures

```elixir
# Eager binary tree
alkeyform BinaryTree do
  phases do
    essence Empty
    essence Leaf, value :: any()
    essence Node,
      left :: catalyst(BinaryTree),
      right :: catalyst(BinaryTree),
      data :: any()
  end
end

def tree_insert(tree, value) do
  distill BinaryTree, tree do
    essence Empty ->
      essence Leaf, value: value
      
    essence Leaf, value: v when value < v ->
      essence Node,
        left: catalyst(essence(Leaf, value: value)),
        right: catalyst(essence(Empty)),
        data: v
        
    essence Leaf, value: v ->
      essence Node,
        left: catalyst(essence(Empty)),
        right: catalyst(essence(Leaf, value: value)),
        data: v
        
    essence Node, left: l, right: r, data: d when value < d ->
      essence Node,
        left: catalyst(tree_insert(l, value)),
        right: catalyst(r),
        data: d
        
    essence Node, left: l, right: r, data: d ->
      essence Node,
        left: catalyst(l),
        right: catalyst(tree_insert(r, value)),
        data: d
  end
end

# Lazy infinite tree
alkeyform LazyTree do
  phases do
    essence Leaf, value :: any()
    essence Branch,
      value :: any(),
      children :: vial(list())
  end
end

def fibonacci_tree(a, b, depth) do
  if depth <= 0 do
    essence Leaf, value: a
  else
    essence Branch,
      value: a,
      children: vial(fn ->
        [
          fibonacci_tree(b, a + b, depth - 1),
          fibonacci_tree(a + b, a + b + b, depth - 1)
        ]
      end)
  end
end
```

---

## Module Organization

```elixir
defmodule MyApp.Types do
  use Alkeyword
  
  # Product types
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
    end
  end
  
  alkeymatter Post do
    elements do
      reagent id :: String.t()
      reagent title :: String.t()
      reagent author_id :: String.t()
      reagent content :: String.t()
    end
  end
  
  # Sum types
  alkeyform Result do
    phases do
      essence Success, value :: any()
      essence Error, message :: String.t()
    end
  end
  
  alkeyform Status do
    phases do
      essence Active
      essence Inactive
      essence Suspended, reason :: String.t()
    end
  end
end

defmodule MyApp.Logic do
  import MyApp.Types
  
  def process_data(input) do
    result = synthesize Result, input do
      {:ok, data} -> essence Success, value: data
      {:error, msg} -> essence Error, message: msg
    end
    
    distill Result, result do
      essence Success, value: v -> handle_success(v)
      essence Error, message: m -> handle_error(m)
    end
  end
end
```

---

## Best Practices

### 1. Naming Conventions
- Use PascalCase for type names: `Result`, `PaymentStatus`, `BinaryTree`
- Use snake_case for fields: `user_id`, `transaction_id`, `error_message`
- Use PascalCase for variant names: `Success`, `Processing`, `Empty`

### 2. Exhaustiveness
Always handle all variants in `distill` blocks. The compiler will warn about missing cases.

```elixir
# Good - all cases handled
distill Result, result do
  essence Success, value: v -> handle_success(v)
  essence Error, message: m -> handle_error(m)
end

# Bad - compiler warning if Result has more variants
distill Result, result do
  essence Success, value: v -> handle_success(v)
end
```

### 3. Performance Considerations
- Use `catalyst` for small, frequently-accessed structures
- Use `vial` for large, infrequently-accessed or infinite structures
- Profile before optimizing - the default (catalyst) is usually fine

### 4. Type Specifications
Always include proper type specs on fields:

```elixir
alkeyform MyType do
  phases do
    essence Variant, 
      field :: String.t(),              # String
      count :: integer(),               # Integer
      active :: boolean(),              # Boolean
      data :: map(),                    # Map
      items :: list(String.t()),        # List of strings
      metadata :: any()                 # Any type
  end
end
```

---

## Quick Reference Card

| Concept | Keyword | Example |
|---------|---------|---------|
| Product type | `alkeymatter` | `alkeymatter User do ... end` |
| Sum type | `alkeyform` | `alkeyform Result do ... end` |
| Field container | `elements` | `elements do reagent id :: String.t() end` |
| Variant container | `phases` | `phases do essence Success end` |
| Field | `reagent` | `reagent name :: String.t()` |
| Variant | `essence` | `essence Error, message :: String.t()` |
| Construct variant | `essence()` | `essence(Success, value: data)` |
| Build from pattern | `synthesize` | `synthesize Result, x do ... end` |
| Match pattern | `distill` | `distill Result, r do ... end` |
| Eager recursion | `catalyst()` | `left :: catalyst(Tree)` |
| Lazy recursion | `vial()` | `tail :: vial(Stream)` |
| Force evaluation | `unseal()` | `unseal(lazy_value)` |

---

## Example: Complete Feature Implementation

```elixir
defmodule ShoppingCart do
  use Alkeyword
  
  # Product types
  alkeymatter Item do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent price :: float()
      reagent quantity :: integer()
    end
  end
  
  alkeymatter Cart do
    elements do
      reagent id :: String.t()
      reagent items :: list()
      reagent user_id :: String.t()
    end
  end
  
  # Sum types
  alkeyform CartOperation do
    phases do
      essence AddItem, item :: map()
      essence RemoveItem, item_id :: String.t()
      essence UpdateQuantity, item_id :: String.t(), quantity :: integer()
      essence Clear
    end
  end
  
  alkeyform CartResult do
    phases do
      essence Success, cart :: map(), message :: String.t()
      essence InvalidQuantity, message :: String.t()
      essence ItemNotFound, item_id :: String.t()
      essence CartEmpty
    end
  end
  
  # Business logic
  def apply_operation(cart, operation) do
    result = synthesize CartResult, {cart, operation} do
      {cart, essence(AddItem, item: item)} ->
        new_cart = add_item_to_cart(cart, item)
        essence Success, cart: new_cart, message: "Item added"
        
      {cart, essence(RemoveItem, item_id: id)} ->
        case find_item(cart, id) do
          nil -> essence ItemNotFound, item_id: id
          _ ->
            new_cart = remove_item_from_cart(cart, id)
            essence Success, cart: new_cart, message: "Item removed"
        end
        
      {cart, essence(UpdateQuantity, item_id: id, quantity: qty)} when qty > 0 ->
        case find_item(cart, id) do
          nil -> essence ItemNotFound, item_id: id
          _ ->
            new_cart = update_quantity(cart, id, qty)
            essence Success, cart: new_cart, message: "Quantity updated"
        end
        
      {_cart, essence(UpdateQuantity, item_id: _, quantity: qty)} when qty <= 0 ->
        essence InvalidQuantity, message: "Quantity must be positive"
        
      {cart, essence(Clear)} ->
        new_cart = %{cart | items: []}
        essence Success, cart: new_cart, message: "Cart cleared"
    end
    
    distill CartResult, result do
      essence Success, cart: c, message: msg ->
        {:ok, c, msg}
        
      essence InvalidQuantity, message: msg ->
        {:error, :invalid_quantity, msg}
        
      essence ItemNotFound, item_id: id ->
        {:error, :not_found, "Item #{id} not found"}
        
      essence CartEmpty ->
        {:error, :empty, "Cart is empty"}
    end
  end
  
  defp add_item_to_cart(cart, item) do
    %{cart | items: [item | cart.items]}
  end
  
  defp remove_item_from_cart(cart, item_id) do
    %{cart | items: Enum.reject(cart.items, &(&1.id == item_id))}
  end
  
  defp update_quantity(cart, item_id, quantity) do
    items = Enum.map(cart.items, fn item ->
      if item.id == item_id do
        %{item | quantity: quantity}
      else
        item
      end
    end)
    %{cart | items: items}
  end
  
  defp find_item(cart, item_id) do
    Enum.find(cart.items, &(&1.id == item_id))
  end
end
```

---

## Testing Patterns

```elixir
defmodule MyApp.TypesTest do
  use ExUnit.Case
  use Alkeyword
  import MyApp.Types
  
  describe "Result type" do
    test "creates success variant" do
      result = essence(Success, value: "data")
      
      message = distill Result, result do
        essence Success, value: v -> "Success: #{v}"
        essence Error, message: _ -> "Error"
      end
      
      assert message == "Success: data"
    end
    
    test "synthesizes from tuple" do
      result = synthesize Result, {:ok, 42} do
        {:ok, val} -> essence Success, value: val
        {:error, msg} -> essence Error, message: msg
      end
      
      assert distill(Result, result, do: (
        essence(Success, value: v) -> v
        essence(Error, message: _) -> nil
      )) == 42
    end
  end
  
  describe "State transitions" do
    test "transitions through states correctly" do
      draft = essence(Draft, items: [], customer: "Alice")
      
      submitted = transition_order(draft, :submit)
      
      assert distill(OrderState, submitted, do: (
        essence(Submitted, order_id: id, timestamp: _, items: _) -> id
        _ -> nil
      )) != nil
    end
  end
end
```

---

## Migration from Traditional Elixir

### Before (Traditional Elixir)
```elixir
defmodule Result do
  defstruct [:type, :value, :error, :code]
end

def process(input) do
  case input do
    {:ok, data} -> %Result{type: :success, value: data}
    {:error, msg} -> %Result{type: :error, error: msg, code: 500}
  end
end

def handle(result) do
  case result.type do
    :success -> "Got: #{result.value}"
    :error -> "Error #{result.code}: #{result.error}"
  end
end
```

### After (Alkeyword)
```elixir
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

def handle(result) do
  distill Result, result do
    essence Success, value: v -> "Got: #{v}"
    essence Error, message: m, code: c -> "Error #{c}: #{m}"
  end
end
```

**Benefits:**
- Type safety at compile time
- Exhaustiveness checking
- Cannot access wrong fields
- Clearer intent
- Better tooling support

---

## Common Errors and Solutions

### Error 1: Missing essence handler
```elixir
# Error: Non-exhaustive pattern match
distill Result, result do
  essence Success, value: v -> v
  # Missing Error case!
end

# Solution: Handle all cases
distill Result, result do
  essence Success, value: v -> v
  essence Error, message: m -> handle_error(m)
end
```

### Error 2: Wrong field names
```elixir
# Error: Unknown field :val
essence Success, val: data

# Solution: Use correct field name
essence Success, value: data
```

### Error 3: Forgetting to unseal vials
```elixir
# Error: Trying to access sealed vial
stream[:tail][:head]  # Won't work!

# Solution: Unseal first
tail = unseal(stream[:tail])
tail[:head]  # Works!
```

---

## When to Use Alkeyword

✅ **Use Alkeyword when:**
- Building type-safe APIs
- Implementing state machines
- Creating domain models
- Handling complex control flow
- Need exhaustive pattern matching
- Working with recursive data structures

❌ **Don't use Alkeyword for:**
- Simple key-value maps
- Performance-critical hot paths (use plain Elixir)
- External API responses (use plain maps first, then convert)
- Temporary intermediate values

---

## Summary Checklist

When writing Alkeyword code, remember:

- [ ] Use `alkeymatter` for structured data (products)
- [ ] Use `alkeyform` for variant types (sums)
- [ ] Use `essence` to construct variants
- [ ] Use `synthesize` to build from patterns
- [ ] Use `distill` to destructure/match
- [ ] Use `catalyst` for eager, `vial` for lazy
- [ ] Use `unseal` to force vial evaluation
- [ ] Always handle all variants (exhaustiveness)
- [ ] Include proper type specifications
- [ ] Follow naming conventions

---

This guide should enable any LLM to write idiomatic Alkeyword code. Refer to specific sections as needed during development.
