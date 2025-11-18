# Document 3: Pattern Matching

## Overview

Exhaustive pattern matching with compile-time coverage analysis and real-time metrics.

## Exhaustive Matching

Alkeyword enforces **exhaustive pattern matching** at compile time.

### Basic Pattern Matching

```elixir
alkeyform Result do
  phases do
    essence Success, value :: any()
    essence Error, message :: String.t(), code :: integer()
  end
end

# ✅ Exhaustive
message = distill Result, result do
  essence Success, value: v -> "Got: #{inspect(v)}"
  essence Error, message: m, code: c -> "Error #{c}: #{m}"
end

# ❌ Compile warning: non-exhaustive
message = distill Result, result do
  essence Success, value: v -> "Got: #{inspect(v)}"
  # Missing Error case!
end
```

### Guard Clauses

```elixir
distill Result, result do
  essence Success, value: v when is_integer(v) and v > 0 ->
    "Positive: #{v}"
  essence Success, value: v when is_integer(v) ->
    "Non-positive: #{v}"
  essence Success, value: v ->
    "Not an integer: #{inspect(v)}"
  essence Error, message: m, code: c ->
    "Error #{c}: #{m}"
end
```

### Nested Patterns

```elixir
alkeyform ApiResponse do
  phases do
    essence Success, data :: map(), status :: integer()
    essence Failure, error :: alkeyform(ErrorDetail)
  end
end

alkeyform ErrorDetail do
  phases do
    essence NetworkError, reason :: String.t()
    essence ValidationError, fields :: list()
    essence ServerError, code :: integer()
  end
end

# Nested matching
distill ApiResponse, response do
  essence Success, data: d, status: s when s in 200..299 ->
    {:ok, d}
    
  essence Failure, error: error_detail ->
    distill ErrorDetail, error_detail do
      essence NetworkError, reason: r -> {:error, :network, r}
      essence ValidationError, fields: f -> {:error, :validation, f}
      essence ServerError, code: c -> {:error, :server, c}
    end
end
```

## Pattern Match Coverage Analysis

Alkeyword tracks pattern match coverage in real-time:

```elixir
# Dashboard shows:
# - Total distill calls: 1,247
# - Exhaustive matches: 1,228 (98.4%)
# - Non-exhaustive warnings: 19 (1.6%)
# - Average match time: 0.003ms
```

### Coverage Metrics

```elixir
Alkeyword.Observatory.get_pattern_coverage("Result")
#=> %{
#=>   type: "Result",
#=>   total_patterns: 2,
#=>   matched_patterns: 2,
#=>   coverage: 100.0,
#=>   exhaustive: true,
#=>   match_distribution: %{
#=>     "Success" => 847,  # 68%
#=>     "Error" => 400     # 32%
#=>   }
#=> }
```

## Performance Optimization

Pattern matching is optimized at compile time:

### Clause Ordering

```elixir
# ✅ Most common first (faster)
distill Result, result do
  essence Success, value: v -> handle_success(v)  # 68% of cases
  essence Error, message: m, code: c -> handle_error(m, c)  # 32%
end

# ❌ Rare case first (slower)
distill Result, result do
  essence Error, message: m, code: c -> handle_error(m, c)  # 32%
  essence Success, value: v -> handle_success(v)  # 68%
end
```

### Match Caching

Alkeyword caches compiled patterns:

```elixir
# First call: compiled and cached
distill Result, result1 do
  essence Success, value: v -> v
  essence Error, message: m, code: c -> nil
end

# Subsequent calls: use cached pattern (10x faster)
distill Result, result2 do
  essence Success, value: v -> v
  essence Error, message: m, code: c -> nil
end
```

---

