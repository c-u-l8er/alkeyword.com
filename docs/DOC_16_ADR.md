# Document 16: Architecture Decision Records

## Overview

Key architectural decisions made during Alkeyword development, including rationale, alternatives considered, and consequences.


### ADR-001: Why Alchemy Metaphor?

**Context:** Need intuitive naming for algebraic data types

**Decision:** Use alchemy terminology

**Rationale:**
- Transformation matches type operations
- Memorable and distinctive
- Aligns with Elixir's "potion" theme
- Matter/Form duality is natural

**Consequences:**
- Unique branding
- Learning curve for new users
- Strong conceptual model

### ADR-002: Catalyst vs Vial

**Context:** Need two recursion strategies

**Decision:** 
- Catalyst = eager (chemical reaction)
- Vial = lazy (sealed container)

**Rationale:**
- Clear metaphor distinction
- Performance implications obvious
- Memorable names

### ADR-003: Observable by Default

**Context:** Production needs visibility

**Decision:** Built-in observability

**Rationale:**
- Following webhost.systems pattern
- Metrics crucial for type systems
- Zero-config monitoring

---

