# Document 6: Phoenix LiveView Integration

## Overview

Alkeyword provides LiveView components for real-time type system observability. Track types, metrics, and pattern matches in your Phoenix dashboard.

## Overview

Alkeyword provides **LiveView components** for real-time type system observability. Track types, metrics, and pattern matches in your Phoenix dashboard.

## Dashboard Installation

```elixir
# router.ex
scope "/", MyAppWeb do
  pipe_through :browser
  
  # Alkeyword dashboard
  alkeyword_dashboard "/alkeyword"
end
```

Visit `http://localhost:4000/alkeyword` to see:
- Type browser
- Real-time metrics
- Pattern match coverage
- Compilation statistics
- Safety scores

## Live Components

### Live Metric Component

```elixir
defmodule MyAppWeb.TypeStatsLive do
  use Phoenix.LiveView
  use Alkeyword.Observable
  
  def render(assigns) do
    ~H"""
    <div class="stats-dashboard">
      <.live_metric 
        label="Types Defined" 
        value={@type_count}
        trend={@type_trend}
        status={:healthy} />
      
      <.live_metric 
        label="Pattern Coverage" 
        value="#{@pattern_coverage}%"
        trend={@coverage_trend}
        status={coverage_status(@pattern_coverage)} />
      
      <.live_metric 
        label="Avg Compilation" 
        value="#{@avg_compile_ms}ms"
        trend={@compile_trend}
        status={:healthy} />
    </div>
    """
  end
  
  def mount(_params, _session, socket) do
    if connected?(socket) do
      # Subscribe to real-time updates
      Alkeyword.Observable.subscribe()
      :timer.send_interval(1000, self(), :update_metrics)
    end
    
    {:ok, load_metrics(socket)}
  end
  
  def handle_info(:update_metrics, socket) do
    {:noreply, load_metrics(socket)}
  end
  
  def handle_info({:alkeyword_update, _event}, socket) do
    {:noreply, load_metrics(socket)}
  end
  
  defp load_metrics(socket) do
    metrics = Alkeyword.Observatory.get_global_metrics()
    
    assign(socket,
      type_count: metrics.type_count,
      type_trend: metrics.type_trend,
      pattern_coverage: metrics.pattern_coverage,
      coverage_trend: metrics.coverage_trend,
      avg_compile_ms: metrics.avg_compile_ms,
      compile_trend: metrics.compile_trend
    )
  end
  
  defp coverage_status(coverage) when coverage >= 95.0, do: :healthy
  defp coverage_status(coverage) when coverage >= 80.0, do: :warning
  defp coverage_status(_), do: :critical
end
```

### Live Type Browser

```elixir
<.live_table 
  id="types-table"
  rows={@types}
  row_click={&show_type_detail/1}>
  
  <:col :let={type} label="Name">
    <span class="type-name"><%= type.name %></span>
  </:col>
  
  <:col :let={type} label="Kind">
    <.badge color={kind_color(type.kind)}>
      <%= type.kind %>
    </.badge>
  </:col>
  
  <:col :let={type} label="Safety Score">
    <.progress_bar 
      value={type.safety_score} 
      max={100}
      color={score_color(type.safety_score)} />
    <%= type.safety_score %>%
  </:col>
  
  <:col :let={type} label="Usage">
    <%= format_number(type.usage_count) %>
  </:col>
  
  <:col :let={type} label="Status">
    <.live_indicator status={type.health_status} />
  </:col>
</.live_table>
```

### Live Pattern Match Heatmap

```elixir
<.live_heatmap 
  id="pattern-heatmap"
  data={@pattern_distribution}
  title="Pattern Match Distribution"
  x_axis="Type"
  y_axis="Variant"
  value_label="Matches" />
```

## Real-Time Updates

LiveView components update automatically:

```elixir
# When a type is compiled
Alkeyword.Observatory.broadcast(:type_compiled, %{
  type: "Result",
  compilation_time_ms: 12.3
})

# When a pattern match occurs
Alkeyword.Observatory.broadcast(:pattern_matched, %{
  type: "Result",
  variant: "Success",
  exhaustive: true
})

# All connected LiveView clients update instantly
```

## Custom Dashboards

Build your own dashboards:

```elixir
defmodule MyAppWeb.CustomDashboardLive do
  use Phoenix.LiveView
  use Alkeyword.Observable
  
  def render(assigns) do
    ~H"""
    <div class="custom-dashboard">
      <h1>My Application Type System</h1>
      
      <div class="metrics-row">
        <.metric_card 
          title="API Result Types" 
          value={@api_result_count}
          icon="🎯" />
        
        <.metric_card 
          title="State Machine Types" 
          value={@state_machine_count}
          icon="🔄" />
        
        <.metric_card 
          title="Error Handling Coverage" 
          value="#{@error_coverage}%"
          icon="⚠️" />
      </div>
      
      <.live_chart 
        type={:line}
        data={@compilation_history}
        title="Compilation Time Trend"
        x_label="Time"
        y_label="Milliseconds" />
      
      <.live_type_graph 
        types={@types}
        show_relationships={true} />
    </div>
    """
  end
end
```

## Observable Hooks

React to type system events:

```elixir
defmodule MyApp.TypeObserver do
  use Alkeyword.Observable.Hook
  
  def handle_event(:type_compiled, %{type: type, time_ms: time}) do
    if time > 50 do
      Logger.warning("Slow compilation: #{type} took #{time}ms")
    end
    :ok
  end
  
  def handle_event(:pattern_non_exhaustive, %{type: type, missing: variants}) do
    Logger.error("Non-exhaustive pattern match in #{type}: missing #{inspect(variants)}")
    :ok
  end
  
  def handle_event(:type_created, %{type: type}) do
    Logger.info("New type defined: #{type}")
    :ok
  end
end
```

---

