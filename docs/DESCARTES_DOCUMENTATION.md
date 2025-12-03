# Descartes Rule Engine Documentation

## Overview

Descartes is a lightweight, JSON-driven rule engine written in Go. It enables complex decision-making logic through declarative rules, evaluators, and actions without requiring code changes for new rules.

### Key Features
- **JSON-based configuration** - Rules defined entirely in JSON
- **Priority-based evaluation** - First-match wins with `evaluator.group.first_match`
- **Template substitution** - Dynamic values with `{{ fieldName }}` syntax
- **In-memory evaluation** - All processing happens in memory for low latency
- **Extensible architecture** - Add custom rules, evaluators, and actions via factories

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                           LAW                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      EVALUATOR                            │   │
│  │  ┌─────────────────┐    ┌─────────────────────────────┐  │   │
│  │  │      RULE       │───>│         ACTION              │  │   │
│  │  │  (Condition)    │    │     (Side Effect)           │  │   │
│  │  └─────────────────┘    └─────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │      FACT       │
                    │ (Input Params)  │
                    └─────────────────┘
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Law** | The main evaluation unit containing slug, evaluator, and optional cache |
| **Fact** | Input parameters to evaluate against a law |
| **Rule** | Condition checker (returns true/false) |
| **Evaluator** | Orchestrates rule checking and action execution |
| **Action** | Performs operations and returns results when rules match |

---

## Directory Structure

```
descartes/
├── cache/                    # Caching implementations
│   ├── cache.go             # CacheItf interface
│   ├── cache_map.go         # In-memory map cache
│   └── cache_nop.go         # No-op cache
├── core/                     # Core engine
│   ├── eval.go              # Law evaluation entry point
│   └── initiator.go         # Factory initialization
├── engine/
│   ├── actions/             # Action implementations
│   │   ├── action/          # Basic actions (int, float, map ops)
│   │   └── group/           # Action grouping
│   ├── evaluators/          # Evaluator implementations
│   │   ├── evaluator/       # Basic evaluator, iterate
│   │   └── group/           # first_match, multi_match, etc.
│   ├── facts/               # Fact interface
│   └── rules/               # Rule implementations
│       ├── rule/            # Individual rules (bool, int, string, etc.)
│       └── group/           # Conditional AND, OR, NOT
├── law/                      # Law and Fact creation
│   ├── law.go               # Law struct and CreateLaw
│   └── fact.go              # Fact struct and FactMaker
└── errors/                   # Error definitions
```

---

## Quick Start

### 1. Initialize the Factory

```go
package main

import (
    "github.com/ananrafs/descartes/core"
    "github.com/ananrafs/descartes/law"
)

func main() {
    // Initialize with default implementations
    core.InitFactory(core.WithDefaults())
}
```

### 2. Create and Register a Law

```go
jsonLaw := `{
    "slug": "discount_rules",
    "evaluator": {
        "type": "evaluator.group.first_match",
        "evaluators": [
            {
                "type": "evaluator",
                "rule": {
                    "type": "rules.string.equal",
                    "field": "customer_type",
                    "value": "premium"
                },
                "action": {
                    "discount_percent": 20,
                    "rule_id": "premium_discount"
                }
            },
            {
                "type": "evaluator",
                "rule": { "type": "rules.default" },
                "action": {
                    "discount_percent": 5,
                    "rule_id": "default_discount"
                }
            }
        ]
    }
}`

lawInstance, err := law.CreateLaw(jsonLaw)
if err != nil {
    panic(err)
}

core.Register(lawInstance)
```

### 3. Create and Evaluate a Fact

```go
jsonFact := `{
    "slug": "discount_rules",
    "param": {
        "customer_type": "premium",
        "order_total": 100000
    }
}`

fact, err := law.CreateFact(jsonFact)
if err != nil {
    panic(err)
}

result, err := core.Eval(fact)
if err != nil {
    panic(err)
}

// result: {"discount_percent": 20, "rule_id": "premium_discount"}
```

---

## Rule Types Reference

### Boolean Rules

```json
{
    "type": "rules.bool",
    "field": "is_active",
    "value": true
}
```

### String Rules

| Type | Description | Example |
|------|-------------|---------|
| `rules.string.equal` | Case-sensitive equality | `"field": "country", "value": "ID"` |
| `rules.string.equal_fold` | Case-insensitive equality | `"field": "country", "value": "id"` |
| `rules.string.equal.dynamic` | Compare two fields | `"left": "{{ field1 }}", "right": "{{ field2 }}"` |

```json
{
    "type": "rules.string.equal",
    "field": "country",
    "value": "ID"
}
```

### Integer Rules

| Type | Description |
|------|-------------|
| `rules.int.equal` | field == value |
| `rules.int.greater` | field > value |
| `rules.int.greater_equal` | field >= value |
| `rules.int.lesser` | field < value |
| `rules.int.lesser_equal` | field <= value |
| `rules.int.between` | start <= field <= end |

**Static comparison:**
```json
{
    "type": "rules.int.greater_equal",
    "field": "amount",
    "value": 100000
}
```

**Dynamic comparison (compare two fields):**
```json
{
    "type": "rules.int.greater.dynamic",
    "left": "{{ current_balance }}",
    "right": "{{ required_minimum }}"
}
```

**Range check:**
```json
{
    "type": "rules.int.between",
    "field": "age",
    "start": 18,
    "end": 65
}
```

### Array Rules

```json
{
    "type": "rules.array.contains",
    "field": "allowed_countries",
    "value": "ID"
}
```

### Existence Check

```json
{
    "type": "rules.exist",
    "field": "promo_code"
}
```

### Default Rule (Catch-all)

```json
{
    "type": "rules.default"
}
```

### Conditional Rules (Logical Operators)

**AND - All rules must match:**
```json
{
    "type": "rules.conditional.and",
    "rules": [
        { "type": "rules.string.equal", "field": "country", "value": "ID" },
        { "type": "rules.string.equal", "field": "platform", "value": "ios" }
    ]
}
```

**OR - At least one must match:**
```json
{
    "type": "rules.conditional.or",
    "rules": [
        { "type": "rules.string.equal", "field": "tier", "value": "gold" },
        { "type": "rules.string.equal", "field": "tier", "value": "platinum" }
    ]
}
```

**NOT - Rule must not match:**
```json
{
    "type": "rules.conditional.not",
    "rule": {
        "type": "rules.string.equal",
        "field": "status",
        "value": "blocked"
    }
}
```

### Time Rules

```json
{
    "type": "rules.time.between",
    "left": {
        "type": "time_type.dynamic",
        "field": "transaction_time"
    },
    "start": {
        "type": "time_type.dynamic",
        "field": "promo_start"
    },
    "end": {
        "type": "time_type.dynamic",
        "field": "promo_end"
    }
}
```

---

## Evaluator Types Reference

### Basic Evaluator

Single rule-action pair:

```json
{
    "type": "evaluator",
    "rule": { ... },
    "action": { ... }
}
```

### First Match (Priority-based)

Returns result from first matching rule. **This is the key evaluator for priority-based rule engines.**

```json
{
    "type": "evaluator.group.first_match",
    "evaluators": [
        { "type": "evaluator", "rule": { ... }, "action": { ... } },
        { "type": "evaluator", "rule": { ... }, "action": { ... } },
        { "type": "evaluator", "rule": { "type": "rules.default" }, "action": { ... } }
    ]
}
```

**Priority is determined by order** - place more specific rules first, fallback rules last.

### Multi-Match

Find up to N matching rules:

```json
{
    "type": "evaluator.group.multi_match",
    "evaluators": [ ... ],
    "max": 3,
    "reentrance": false,
    "merging": true
}
```

### Iterate

Loop through an array field:

```json
{
    "type": "evaluator.iterate",
    "field": "item",
    "iterant": "cart_items",
    "evaluator": {
        "type": "evaluator",
        "rule": { ... },
        "action": { ... }
    }
}
```

---

## Action Types Reference

### Generic Action (Template Output)

```json
{
    "discount": "{{ calculated_discount }}",
    "rule_id": "rule_001",
    "customer": "{{ customer_id }}"
}
```

### Action Group

Execute multiple actions sequentially:

```json
{
    "type": "actions.group",
    "actions": [
        {
            "type": "actions.int.multiple",
            "field": "cashback",
            "factors": ["{{ amount }}", 0.1]
        },
        {
            "cashback_value": "{{ cashback }}",
            "rule_id": "cashback_10_percent"
        }
    ]
}
```

### Arithmetic Actions

**Integer operations:**
```json
{
    "type": "actions.int.sum",
    "field": "total",
    "factors": ["{{ subtotal }}", "{{ tax }}", "{{ shipping }}"]
}
```

| Type | Operation |
|------|-----------|
| `actions.int.sum` | a + b + c + ... |
| `actions.int.subtract` | a - b - c - ... |
| `actions.int.multiple` | a * b * c * ... |
| `actions.int.divide` | a / b / c / ... |
| `actions.int.mod` | a % b |

**Float operations:** Same as integer with `actions.float.*` prefix.

### Map Append

Merge objects:

```json
{
    "type": "actions.map.append",
    "field": "metadata",
    "object": {
        "source": "rule_engine",
        "version": "1.0"
    }
}
```

---

## Template Syntax

Use `{{ fieldName }}` to reference fact values:

```json
{
    "action": {
        "message": "Hello, {{ customer_name }}!",
        "amount": "{{ order_total }}"
    }
}
```

Templates work in:
- Rule comparisons (dynamic rules)
- Action outputs
- Field names

---

## Caching

Enable caching for repeated evaluations:

```json
{
    "slug": "my_law",
    "evaluator": { ... },
    "cache": "cache.map"
}
```

| Cache Type | Description |
|------------|-------------|
| `cache.map` | In-memory hash-based caching |
| `""` (empty) | No caching (default) |

---

## Fact Builder Pattern

Programmatic fact construction:

```go
import "github.com/ananrafs/descartes/law"

fact := law.MakeFact(
    map[string]interface{}{
        "country": "ID",
        "platform": "ios",
    },
).AddFields(
    map[string]interface{}{
        "amount": 150000,
        "user_segment": "premium",
    },
).Generate("cashback_rules")

result, err := core.Eval(fact)
```

---

## Best Practices

### 1. Rule Ordering
Place more specific rules first in `first_match` evaluators:

```json
{
    "type": "evaluator.group.first_match",
    "evaluators": [
        // Most specific: country + platform + category + segment
        { "rule": { "type": "rules.conditional.and", "rules": [...4 conditions...] }, ... },
        // Less specific: country + platform + category
        { "rule": { "type": "rules.conditional.and", "rules": [...3 conditions...] }, ... },
        // Generic: country only
        { "rule": { "type": "rules.string.equal", "field": "country", ... }, ... },
        // Fallback: default
        { "rule": { "type": "rules.default" }, ... }
    ]
}
```

### 2. Always Include a Default Rule
Prevent "no match" scenarios:

```json
{
    "type": "evaluator",
    "rule": { "type": "rules.default" },
    "action": {
        "result": "fallback",
        "rule_id": "default_rule"
    }
}
```

### 3. Include Rule IDs in Actions
For auditability, always return the matched rule identifier:

```json
{
    "action": {
        "cashback_percent": 10,
        "applied_rule_id": "rule_premium_id_electronics"
    }
}
```

### 4. Use Action Groups for Complex Calculations

```json
{
    "type": "actions.group",
    "actions": [
        { "type": "actions.int.multiple", "field": "raw_cashback", "factors": ["{{ amount }}", 0.10] },
        { "type": "actions.int.lesser", "field": "cashback", "factors": ["{{ raw_cashback }}", "{{ max_cashback }}"] },
        { "cashback": "{{ cashback }}", "rule_id": "calculated" }
    ]
}
```

---

## Performance Characteristics

- **In-memory evaluation**: No I/O during rule evaluation
- **First-match short-circuit**: Stops on first match in `first_match` evaluator
- **Parse once**: Laws are parsed at registration time
- **Suitable for**: Up to 50+ evaluations/second on modest hardware
- **Rule complexity**: Handles 100+ rules efficiently in `first_match`

---

## Error Handling

```go
result, err := core.Eval(fact)
if err != nil {
    switch err {
    case errors.ErrNoLawFound:
        // Law with slug not registered
    case errors.ErrEvaluation:
        // Rule evaluation error
    default:
        // Other errors
    }
}
```

---

## Complete Example

```json
{
    "slug": "shipping_cost",
    "evaluator": {
        "type": "evaluator.group.first_match",
        "evaluators": [
            {
                "type": "evaluator",
                "rule": {
                    "type": "rules.conditional.and",
                    "rules": [
                        { "type": "rules.string.equal", "field": "country", "value": "ID" },
                        { "type": "rules.int.greater_equal", "field": "order_total", "value": 500000 }
                    ]
                },
                "action": {
                    "shipping_cost": 0,
                    "shipping_type": "free",
                    "rule_id": "free_shipping_id_500k"
                }
            },
            {
                "type": "evaluator",
                "rule": {
                    "type": "rules.string.equal",
                    "field": "country",
                    "value": "ID"
                },
                "action": {
                    "shipping_cost": 15000,
                    "shipping_type": "standard",
                    "rule_id": "standard_shipping_id"
                }
            },
            {
                "type": "evaluator",
                "rule": { "type": "rules.default" },
                "action": {
                    "shipping_cost": 50000,
                    "shipping_type": "international",
                    "rule_id": "international_shipping"
                }
            }
        ]
    }
}
```

---

## Version

This documentation covers Descartes as of commit `a7a6630`.
