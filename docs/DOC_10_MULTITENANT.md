# Document 10: Multi-tenant Patterns

## Overview

Alkeyword + Ash provides compile-time guaranteed multi-tenant isolation. Every query is automatically scoped to the current tenant, with zero possibility of data leakage between customers.

## Attribute-Based Multi-tenancy

### Setup

```elixir
defmodule MyApp.Types.Organization do
  use Alkeyword
  use Alkeyword.Ash
  
  alkeymatter Organization do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent subdomain :: String.t()
      reagent created_at :: DateTime.t()
    end
  end
  
  # No tenant_id on Organization itself
  # It's the tenant root
end

defmodule MyApp.Types.User do
  use Alkeyword
  use Alkeyword.Ash
  use Alkeyword.Observable
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
      reagent role :: String.t()
      reagent tenant_id :: String.t()
    end
  end
  
  # Ash multi-tenancy configuration
  multitenancy do
    strategy :attribute
    attribute :tenant_id
    global? false  # Customers can't access each other's data - GUARANTEED
  end
  
  relationships do
    belongs_to :organization, Organization do
      source_attribute :tenant_id
      destination_attribute :id
    end
  end
end
```

### Usage

```elixir
# All queries automatically scoped to tenant
# No way to accidentally query across tenants

# ✅ Correct - tenant scoped
MyApp.User
|> Ash.Query.for_read(:read, %{}, tenant: "org_123")
|> MyApp.Api.read!()
# SQL: SELECT * FROM users WHERE tenant_id = 'org_123'

# ❌ Error - tenant required
MyApp.User
|> Ash.Query.for_read(:read)
|> MyApp.Api.read!()
# ** (Ash.Error.Invalid) tenant is required

# ✅ Create with tenant
MyApp.User
|> Ash.Changeset.for_create(:create, %{
  name: "Alice",
  email: "alice@example.com",
  role: "admin"
}, tenant: "org_123")
|> MyApp.Api.create!()
# Automatically sets tenant_id = 'org_123'
```

### Multi-tenant Authorization

```elixir
alkeymatter Document do
  elements do
    reagent id :: String.t()
    reagent title :: String.t()
    reagent content :: String.t()
    reagent owner_id :: String.t()
    reagent tenant_id :: String.t()
  end
end

multitenancy do
  strategy :attribute
  attribute :tenant_id
  global? false
end

# Authorization policies respect tenancy
policies do
  # Users can only read documents in their tenant
  policy action_type(:read) do
    authorize_if expr(tenant_id == ^actor(:tenant_id))
  end
  
  # Users can only update their own documents
  policy action_type(:update) do
    authorize_if expr(owner_id == ^actor(:id))
    authorize_if actor_attribute_equals(:role, "admin")
  end
  
  # Admins in the tenant can delete
  policy action_type(:destroy) do
    authorize_if actor_attribute_equals(:role, "admin")
  end
end
```

## Schema-Based Multi-tenancy

### PostgreSQL Schemas per Tenant

```elixir
defmodule MyApp.Types.User do
  use Alkeyword
  use Alkeyword.Ash
  
  alkeymatter User do
    elements do
      reagent id :: String.t()
      reagent name :: String.t()
      reagent email :: String.t()
    end
  end
  
  # PostgreSQL schema per tenant
  multitenancy do
    strategy :context
    parse_attribute {MyApp.Tenant, :get_schema, []}
  end
end

# Tenant module
defmodule MyApp.Tenant do
  def get_schema(%{tenant: tenant_id}) do
    "tenant_#{tenant_id}"
  end
end

# Usage
MyApp.User
|> Ash.Query.for_read(:read, %{}, tenant: "org_123")
|> MyApp.Api.read!()
# SQL: SELECT * FROM tenant_org_123.users

MyApp.User
|> Ash.Query.for_read(:read, %{}, tenant: "org_456")
|> MyApp.Api.read!()
# SQL: SELECT * FROM tenant_org_456.users
```

### Schema Creation

```elixir
defmodule MyApp.Tenants do
  def create_tenant(organization_id, name) do
    # Create PostgreSQL schema
    Ecto.Adapters.SQL.query!(
      MyApp.Repo,
      "CREATE SCHEMA tenant_#{organization_id}"
    )
    
    # Run migrations for this schema
    Ecto.Migrator.run(
      MyApp.Repo,
      migrations_path(),
      :up,
      prefix: "tenant_#{organization_id}"
    )
    
    # Create organization record
    Organization.create(%{
      id: organization_id,
      name: name
    })
  end
  
  def delete_tenant(organization_id) do
    # Drop schema (cascade deletes all tables)
    Ecto.Adapters.SQL.query!(
      MyApp.Repo,
      "DROP SCHEMA tenant_#{organization_id} CASCADE"
    )
    
    # Delete organization record
    Organization.destroy(organization_id)
  end
end
```

## Hybrid Multi-tenancy

Combine both strategies:

```elixir
# Global tables (all tenants)
alkeymatter GlobalSettings do
  elements do
    reagent key :: String.t()
    reagent value :: String.t()
  end
end

multitenancy do
  strategy :attribute
  attribute :tenant_id
  global? true  # Accessible to all tenants
end

# Tenant-specific tables
alkeymatter User do
  elements do
    reagent id :: String.t()
    reagent name :: String.t()
    reagent tenant_id :: String.t()
  end
end

multitenancy do
  strategy :attribute
  attribute :tenant_id
  global? false  # Tenant-scoped only
end
```

## Multi-tenant GraphQL

### Automatic Tenant Scoping

```elixir
defmodule MyAppWeb.Context do
  def build_context(conn) do
    # Extract tenant from subdomain or JWT
    tenant_id = extract_tenant(conn)
    
    %{
      tenant: tenant_id,
      actor: get_current_user(conn)
    }
  end
  
  defp extract_tenant(conn) do
    # From subdomain
    case conn.host do
      "acme.myapp.com" -> "org_acme"
      "widgets.myapp.com" -> "org_widgets"
      _ -> nil
    end
  end
end

# All GraphQL queries automatically scoped
query {
  users {  # Only returns users in current tenant
    id
    name
    email
  }
}
```

### Tenant-Aware Subscriptions

```elixir
defmodule MyAppWeb.Schema.Subscriptions do
  use Absinthe.Schema.Notation
  
  subscription do
    field :user_created, :user do
      config fn _args, %{context: %{tenant: tenant_id}} ->
        {:ok, topic: "user_created:#{tenant_id}"}
      end
      
      trigger :create_user,
        topic: fn user ->
          "user_created:#{user.tenant_id}"
        end
    end
  end
end

# Each tenant gets their own subscription channel
# tenant_org_123 can't receive updates from tenant_org_456
```

## Observable Tenant Metrics

### Real-time Tenant Statistics

```elixir
Alkeyword.Observatory.get_tenant_metrics()
#=> %{
#=>   total_tenants: 847,
#=>   active_tenants_24h: 623,
#=>   active_tenants_7d: 782,
#=>   types_per_tenant_avg: 47.2,
#=>   types_per_tenant_median: 34,
#=>   largest_tenant: %{
#=>     id: "org_456",
#=>     name: "Acme Corp",
#=>     types: 1_247,
#=>     users: 847,
#=>     storage_mb: 2_847.3
#=>   },
#=>   smallest_tenant: %{
#=>     id: "org_789",
#=>     name: "Startup Inc",
#=>     types: 12,
#=>     users: 3,
#=>     storage_mb: 4.2
#=>   }
#=> }
```

### Per-Tenant Performance

```elixir
Alkeyword.Observatory.get_tenant_performance("org_123")
#=> %{
#=>   tenant_id: "org_123",
#=>   queries_24h: 12_847,
#=>   avg_query_time_ms: 23.4,
#=>   p95_query_time_ms: 47.8,
#=>   p99_query_time_ms: 89.2,
#=>   error_rate: 0.12,
#=>   slow_queries: [
#=>     %{query: "list_documents", time_ms: 247.3, count: 12}
#=>   ],
#=>   type_usage: %{
#=>     "User" => 8_421,
#=>     "Document" => 3_247,
#=>     "Project" => 1_179
#=>   }
#=> }
```

### Tenant Health Dashboard

```elixir
defmodule MyAppWeb.TenantDashboardLive do
  use Phoenix.LiveView
  use Alkeyword.Observable
  
  def render(assigns) do
    ~H"""
    <div class="tenant-dashboard">
      <h1>Tenant Overview</h1>
      
      <div class="metrics-grid">
        <.live_metric
          label="Total Tenants"
          value={@total_tenants}
          trend={@tenant_trend} />
        
        <.live_metric
          label="Active (24h)"
          value={@active_tenants}
          status={:healthy} />
        
        <.live_metric
          label="Avg Types/Tenant"
          value={@avg_types_per_tenant}
          trend={@types_trend} />
      </div>
      
      <.live_table
        id="tenants-table"
        rows={@tenants}
        row_click={&show_tenant_detail/1}>
        <:col :let={tenant} label="Organization">
          <%= tenant.name %>
        </:col>
        <:col :let={tenant} label="Users">
          <%= tenant.user_count %>
        </:col>
        <:col :let={tenant} label="Types">
          <%= tenant.type_count %>
        </:col>
        <:col :let={tenant} label="Queries (24h)">
          <%= format_number(tenant.queries_24h) %>
        </:col>
        <:col :let={tenant} label="Health">
          <.live_indicator status={tenant.health_status} />
        </:col>
      </.live_table>
    </div>
    """
  end
  
  def mount(_params, _session, socket) do
    if connected?(socket) do
      Alkeyword.Observable.subscribe(:tenant_metrics)
      :timer.send_interval(5000, self(), :update_metrics)
    end
    
    {:ok, load_tenant_metrics(socket)}
  end
  
  def handle_info(:update_metrics, socket) do
    {:noreply, load_tenant_metrics(socket)}
  end
end
```

## Tenant Data Isolation Guarantees

### Compile-Time Checks

```elixir
# ❌ This won't compile - tenant required
def get_all_users do
  MyApp.User
  |> Ash.Query.for_read(:read)
  |> MyApp.Api.read!()
end
# ** (Ash.Error.Invalid) tenant is required for User

# ✅ This compiles - tenant provided
def get_tenant_users(tenant_id) do
  MyApp.User
  |> Ash.Query.for_read(:read, %{}, tenant: tenant_id)
  |> MyApp.Api.read!()
end
```

### Runtime Validation

```elixir
# Attempting to access wrong tenant fails
user = MyApp.User.get!("user_123", tenant: "org_123")

# Try to update with different tenant
MyApp.User
|> Ash.Changeset.for_update(:update, user, %{name: "Bob"}, tenant: "org_456")
|> MyApp.Api.update!()
# ** (Ash.Error.Forbidden) Cannot access resource from different tenant
```

## Migration Strategies

### From Single-Tenant to Multi-Tenant

```elixir
# Step 1: Add tenant_id column
defmodule MyApp.Repo.Migrations.AddTenantId do
  use Ecto.Migration
  
  def change do
    alter table(:users) do
      add :tenant_id, :string
    end
    
    create index(:users, [:tenant_id])
  end
end

# Step 2: Backfill existing data
defmodule MyApp.Repo.Migrations.BackfillTenantId do
  use Ecto.Migration
  
  def up do
    # Assign all existing users to default tenant
    execute """
    UPDATE users SET tenant_id = 'default_org'
    WHERE tenant_id IS NULL
    """
    
    # Make tenant_id required
    alter table(:users) do
      modify :tenant_id, :string, null: false
    end
  end
end

# Step 3: Update Alkeyword types
alkeymatter User do
  elements do
    reagent id :: String.t()
    reagent name :: String.t()
    reagent tenant_id :: String.t()  # Add this
  end
end

multitenancy do
  strategy :attribute
  attribute :tenant_id
  global? false
end

# Step 4: Update all queries
# Before
User.list()

# After
User.list(tenant: current_tenant_id)
```

## Testing Multi-tenancy

### Tenant Isolation Tests

```elixir
defmodule MyApp.MultiTenancyTest do
  use MyApp.DataCase
  use Alkeyword
  
  setup do
    # Create two tenants
    org1 = create_organization("org_1", "Company A")
    org2 = create_organization("org_2", "Company B")
    
    # Create users in each tenant
    user1 = create_user("Alice", "org_1")
    user2 = create_user("Bob", "org_2")
    
    %{org1: org1, org2: org2, user1: user1, user2: user2}
  end
  
  test "users cannot see other tenant's data", %{user1: user1, user2: user2} do
    # Query as org_1
    users_org1 = User.list(tenant: "org_1")
    assert length(users_org1) == 1
    assert Enum.any?(users_org1, &(&1.id == user1.id))
    refute Enum.any?(users_org1, &(&1.id == user2.id))
    
    # Query as org_2
    users_org2 = User.list(tenant: "org_2")
    assert length(users_org2) == 1
    assert Enum.any?(users_org2, &(&1.id == user2.id))
    refute Enum.any?(users_org2, &(&1.id == user1.id))
  end
  
  test "cannot access resource from wrong tenant", %{user1: user1} do
    # Try to access user1 from wrong tenant
    assert_raise Ash.Error.Forbidden, fn ->
      User.get!(user1.id, tenant: "org_2")
    end
  end
  
  test "cannot update resource from wrong tenant", %{user1: user1} do
    # Try to update user1 from wrong tenant
    assert_raise Ash.Error.Forbidden, fn ->
      User
      |> Ash.Changeset.for_update(:update, user1, %{name: "Eve"}, tenant: "org_2")
      |> MyApp.Api.update!()
    end
  end
end
```

## Performance Optimization

### Tenant-Specific Indexes

```elixir
# Migration
defmodule MyApp.Repo.Migrations.AddTenantIndexes do
  use Ecto.Migration
  
  def change do
    # Composite index for common queries
    create index(:users, [:tenant_id, :email])
    create index(:users, [:tenant_id, :created_at])
    
    # For large tenants, consider partitioning
    execute """
    CREATE INDEX CONCURRENTLY users_tenant_id_large_idx
    ON users(tenant_id)
    WHERE tenant_id IN ('org_large_1', 'org_large_2')
    """
  end
end
```

### Query Optimization

```elixir
# ❌ Inefficient - fetches all then filters
def get_active_users(tenant_id) do
  User
  |> Ash.Query.for_read(:read, %{}, tenant: tenant_id)
  |> Ash.Query.filter(active == true)
  |> MyApp.Api.read!()
end

# ✅ Efficient - filters at database level
def get_active_users(tenant_id) do
  User
  |> Ash.Query.for_read(:read, %{}, tenant: tenant_id)
  |> Ash.Query.filter(active == true and tenant_id == ^tenant_id)
  |> MyApp.Api.read!()
end
# SQL: SELECT * FROM users WHERE tenant_id = 'org_123' AND active = true
```

## Security Best Practices

### 1. Always Validate Tenant from Token

```elixir
defmodule MyAppWeb.Plugs.TenantAuth do
  import Plug.Conn
  
  def init(opts), do: opts
  
  def call(conn, _opts) do
    with {:ok, token} <- extract_token(conn),
         {:ok, claims} <- verify_token(token),
         {:ok, tenant_id} <- extract_tenant(claims),
         :ok <- validate_tenant_exists(tenant_id) do
      assign(conn, :tenant_id, tenant_id)
    else
      _ ->
        conn
        |> put_status(:unauthorized)
        |> Phoenix.Controller.json(%{error: "Invalid tenant"})
        |> halt()
    end
  end
end
```

### 2. Audit Tenant Access

```elixir
defmodule MyApp.TenantAudit do
  use Alkeyword.Observable
  
  def log_access(tenant_id, user_id, resource, action) do
    Alkeyword.Observable.track_event(:tenant_access, %{
      tenant_id: tenant_id,
      user_id: user_id,
      resource: resource,
      action: action,
      timestamp: DateTime.utc_now()
    })
    
    # Store in audit log
    AuditLog.create(%{
      tenant_id: tenant_id,
      user_id: user_id,
      resource: resource,
      action: action
    })
  end
end
```

### 3. Rate Limiting per Tenant

```elixir
defmodule MyAppWeb.Plugs.TenantRateLimit do
  use PlugRateLimit,
    name: :tenant_api,
    max_requests: 1000,
    interval_seconds: 60
  
  def rate_limit_key(conn) do
    tenant_id = conn.assigns[:tenant_id]
    "tenant:#{tenant_id}"
  end
end
```

## Summary

Alkeyword + Ash multi-tenancy provides:
- ✅ **Compile-time guarantees** - Can't query without tenant
- ✅ **Runtime validation** - Cross-tenant access blocked
- ✅ **Observable metrics** - Per-tenant performance tracking
- ✅ **Flexible strategies** - Attribute or schema-based
- ✅ **GraphQL support** - Automatic tenant scoping
- ✅ **Production-ready** - Battle-tested patterns

**Multi-tenancy is not an afterthought—it's built into the type system.**

---

**Next:** [Document 11: Observable Type System](#document-11-observable-type-system)
