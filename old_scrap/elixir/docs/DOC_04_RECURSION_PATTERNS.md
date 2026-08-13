# Document 4: Recursion Patterns

## Overview

Deep dive into catalyst (eager) and vial (lazy) recursion strategies with performance benchmarks.

## Catalyst vs Vial

Alkeyword provides two recursion strategies:

| Feature | Catalyst (Eager) | Vial (Lazy) |
|---------|------------------|-------------|
| **Evaluation** | Immediate | On-demand |
| **Memory** | Higher upfront | Lower initially |
| **Access Speed** | Fast (no function calls) | Slower (requires unseal) |
| **Use Case** | Small, frequent access | Large, rare access |
| **Infinite Structures** | ❌ Not supported | ✅ Supported |

## Catalyst: Eager Recursion

**Catalysts** compute recursively immediately:

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

# Build tree - all nodes computed now
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
left_value = tree[:left][:value]  #=> 1 (instant)
```

### When to Use Catalyst

✅ **Use catalyst when:**
- Tree depth < 15
- Frequently accessed data
- Bounded data structures
- Performance-critical paths
- Memory is available

❌ **Avoid catalyst when:**
- Deep nesting (>20 levels)
- Infrequent access
- Infinite structures
- Memory constrained

### Catalyst Performance

```elixir
# Benchmark: 1000 node tree
# Construction: 45ms
# Access (1000 reads): 3ms
# Memory: 12MB
# Pattern match: 0.002ms avg
```

## Vial: Lazy Recursion

**Vials** defer computation until unsealed:

```elixir
alkeyform LazyStream do
  phases do
    essence Empty
    essence Cons,
      head :: any(),
      tail :: vial(LazyStream)
  end
end

# Build lazy stream - tail not computed yet
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

# Access requires unsealing
tail_stream = unseal(stream[:tail])  # Computes now
second_value = tail_stream[:head]    #=> 2
```

### When to Use Vial

✅ **Use vial when:**
- Infinite or very large structures
- Infrequent access
- Streaming scenarios
- Expensive computations
- Memory constrained

❌ **Avoid vial when:**
- Frequent access (overhead adds up)
- Small structures
- Need predictable performance
- Real-time requirements

### Vial Performance

```elixir
# Benchmark: 1000 element stream
# Construction: 2ms (deferred)
# Access (100 reads): 15ms (100 unseals)
# Memory: 1MB (initial), grows as unsealed
# Pattern match: 0.005ms avg
```

## Infinite Structures

Vials enable infinite structures:

```elixir
defmodule FibonacciStream do
  use Alkeyword
  
  def generate(a \\ 0, b \\ 1) do
    essence(Cons,
      head: a,
      tail: vial(fn -> generate(b, a + b) end)
    )
  end
  
  def take(stream, 0), do: []
  def take(stream, n) do
    distill LazyStream, stream do
      essence Empty -> []
      essence Cons, head: h, tail: t ->
        [h | take(unseal(t), n - 1)]
    end
  end
end

# Generate infinite Fibonacci sequence
fibs = FibonacciStream.generate()

# Take first 10 (only computes 10)
first_10 = FibonacciStream.take(fibs, 10)
#=> [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

## Mixed Recursion

Combine catalyst and vial in same structure:

```elixir
alkeyform HybridTree do
  phases do
    essence Empty
    essence EagerNode,
      value :: any(),
      children :: list()  # Eager evaluation
    essence LazyNode,
      value :: any(),
      children :: vial(list())  # Lazy evaluation
  end
end

# Eager children (computed immediately)
eager = essence(EagerNode,
  value: "root",
  children: [
    catalyst(essence(EagerNode, value: "child1", children: [])),
    catalyst(essence(EagerNode, value: "child2", children: []))
  ]
)

# Lazy children (computed on demand)
lazy = essence(LazyNode,
  value: "root",
  children: vial(fn ->
    [essence(LazyNode, value: "child1", children: vial(fn -> [] end))]
  end)
)
```

## Observable Recursion Metrics

Track recursion performance in real-time:

```elixir
Alkeyword.Observatory.get_recursion_metrics()
#=> %{
#=>   catalyst_calls: 12_847,
#=>   vial_calls: 3_421,
#=>   unseal_calls: 3_421,
#=>   avg_catalyst_depth: 8.3,
#=>   avg_vial_depth: 142.7,
#=>   catalyst_memory_mb: 45.2,
#=>   vial_memory_mb: 12.8
#=> }
```

---

