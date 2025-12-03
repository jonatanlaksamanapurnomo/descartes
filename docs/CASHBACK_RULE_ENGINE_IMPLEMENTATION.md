# Cashback Rule Engine Implementation Guide

## Overview

This document provides a high-level implementation guide for building a multi-dimensional, priority-based cashback rule engine using Descartes.

### Goals
- Replace flat cashback config with priority-based rules
- Support dimensions: **country**, **platform**, **category**, **user_segment**
- Achieve **<200ms** evaluation latency
- Enable **auditability** (log which rule was applied)
- Maintain **backward compatibility**

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Transaction Request                            │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Extract Context                                                      │
│  - country (from user/IP)                                            │
│  - platform (ios/android/web)                                        │
│  - category (merchant category)                                       │
│  - user_segment (standard/premium/vip)                               │
│  - amount (transaction value)                                        │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Descartes Rule Engine                                               │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  evaluator.group.first_match                                    │ │
│  │  ┌──────────────────────────────────────────────────────────┐  │ │
│  │  │ Rule 1: country=ID + platform=ios + category=electronics │  │ │
│  │  │         + user_segment=premium                           │  │ │
│  │  │         → cashback: 10%, max: 50000                      │  │ │
│  │  ├──────────────────────────────────────────────────────────┤  │ │
│  │  │ Rule 2: country=ID + platform=ios + category=electronics │  │ │
│  │  │         → cashback: 7%, max: 30000                       │  │ │
│  │  ├──────────────────────────────────────────────────────────┤  │ │
│  │  │ Rule 3: country=ID + category=electronics                │  │ │
│  │  │         → cashback: 5%, max: 20000                       │  │ │
│  │  ├──────────────────────────────────────────────────────────┤  │ │
│  │  │ Rule N: Default                                          │  │ │
│  │  │         → cashback: 2%, max: 10000                       │  │ │
│  │  └──────────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Output: { cashback_percent, max_granted, applied_rule_id }          │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Apply Cashback + Log to coins_cashback table                        │
│  Record: transaction_id, rule_id, cashback_value                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Law Structure (cashback_v1)

```json
{
  "slug": "cashback_v1",
  "evaluator": {
    "type": "evaluator.group.first_match",
    "evaluators": [
      {
        "type": "evaluator",
        "rule": {
          "type": "rules.conditional.and",
          "rules": [
            { "type": "rules.string.equal", "field": "country", "value": "ID" },
            { "type": "rules.string.equal", "field": "platform", "value": "ios" },
            { "type": "rules.string.equal", "field": "category", "value": "electronics" },
            { "type": "rules.string.equal", "field": "user_segment", "value": "premium" }
          ]
        },
        "action": {
          "type": "actions.group",
          "actions": [
            {
              "cashback_percent": 10,
              "max_granted": 50000,
              "applied_rule_id": "rule_premium_id_electronics_ios"
            }
          ]
        }
      },
      {
        "type": "evaluator",
        "rule": {
          "type": "rules.conditional.and",
          "rules": [
            { "type": "rules.string.equal", "field": "country", "value": "ID" },
            { "type": "rules.string.equal", "field": "platform", "value": "ios" },
            { "type": "rules.string.equal", "field": "category", "value": "electronics" }
          ]
        },
        "action": {
          "cashback_percent": 7,
          "max_granted": 30000,
          "applied_rule_id": "rule_id_electronics_ios"
        }
      },
      {
        "type": "evaluator",
        "rule": {
          "type": "rules.conditional.and",
          "rules": [
            { "type": "rules.string.equal", "field": "country", "value": "ID" },
            { "type": "rules.string.equal", "field": "category", "value": "electronics" }
          ]
        },
        "action": {
          "cashback_percent": 5,
          "max_granted": 20000,
          "applied_rule_id": "rule_id_electronics"
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
          "cashback_percent": 3,
          "max_granted": 15000,
          "applied_rule_id": "rule_id_default"
        }
      },
      {
        "type": "evaluator",
        "rule": { "type": "rules.default" },
        "action": {
          "cashback_percent": 2,
          "max_granted": 10000,
          "applied_rule_id": "rule_global_default"
        }
      }
    ]
  },
  "cache": "cache.map"
}
```

---

## Fact Format (Transaction Input)

```json
{
  "slug": "cashback_v1",
  "param": {
    "country": "ID",
    "platform": "ios",
    "category": "electronics",
    "user_segment": "premium",
    "amount": 500000,
    "transaction_id": "txn_123456"
  }
}
```

---

## Go Implementation

### 1. Initialize Rule Engine (on application startup)

```go
package cashback

import (
    "github.com/ananrafs/descartes/core"
    "github.com/ananrafs/descartes/law"
)

var cashbackLaw *law.Law

func InitCashbackEngine() error {
    // Initialize Descartes with defaults
    core.InitFactory(core.WithDefaults())

    // Load law from config/database
    lawJSON := loadCashbackLawFromConfig() // returns JSON string

    var err error
    cashbackLaw, err = law.CreateLaw(lawJSON)
    if err != nil {
        return fmt.Errorf("failed to create cashback law: %w", err)
    }

    core.Register(cashbackLaw)
    return nil
}
```

### 2. Cashback Service

```go
package cashback

import (
    "context"
    "time"

    "github.com/ananrafs/descartes/core"
    "github.com/ananrafs/descartes/law"
)

type CashbackRequest struct {
    TransactionID string  `json:"transaction_id"`
    Country       string  `json:"country"`
    Platform      string  `json:"platform"`
    Category      string  `json:"category"`
    UserSegment   string  `json:"user_segment"`
    Amount        float64 `json:"amount"`
}

type CashbackResult struct {
    CashbackPercent float64 `json:"cashback_percent"`
    MaxGranted      float64 `json:"max_granted"`
    AppliedRuleID   string  `json:"applied_rule_id"`
    CashbackValue   float64 `json:"cashback_value"`
    EvalLatencyMs   int64   `json:"eval_latency_ms"`
}

func EvaluateCashback(ctx context.Context, req CashbackRequest) (*CashbackResult, error) {
    start := time.Now()

    // Build fact using FactMaker
    fact := law.MakeFact(
        map[string]interface{}{
            "country":        req.Country,
            "platform":       req.Platform,
            "category":       req.Category,
            "user_segment":   req.UserSegment,
            "amount":         req.Amount,
            "transaction_id": req.TransactionID,
        },
    ).Generate("cashback_v1")

    // Evaluate against rule engine
    result, err := core.Eval(fact)
    if err != nil {
        return nil, fmt.Errorf("rule evaluation failed: %w", err)
    }

    latency := time.Since(start).Milliseconds()

    // Parse result
    resultMap := result.(map[string]interface{})

    cashbackPercent := toFloat64(resultMap["cashback_percent"])
    maxGranted := toFloat64(resultMap["max_granted"])
    ruleID := toString(resultMap["applied_rule_id"])

    // Calculate actual cashback
    rawCashback := req.Amount * (cashbackPercent / 100)
    cashbackValue := min(rawCashback, maxGranted)

    return &CashbackResult{
        CashbackPercent: cashbackPercent,
        MaxGranted:      maxGranted,
        AppliedRuleID:   ruleID,
        CashbackValue:   cashbackValue,
        EvalLatencyMs:   latency,
    }, nil
}
```

### 3. HTTP Handler

```go
package handler

import (
    "encoding/json"
    "net/http"

    "yourapp/cashback"
)

func HandleCashbackEvaluation(w http.ResponseWriter, r *http.Request) {
    var req cashback.CashbackRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    result, err := cashback.EvaluateCashback(r.Context(), req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(result)
}
```

### 4. Audit Logging

```go
package cashback

import (
    "context"
    "time"
)

type CashbackAuditLog struct {
    TransactionID   string    `db:"transaction_id"`
    AppliedRuleID   string    `db:"applied_rule_id"`
    CashbackPercent float64   `db:"cashback_percent"`
    CashbackValue   float64   `db:"cashback_value"`
    MaxGranted      float64   `db:"max_granted"`
    EvalLatencyMs   int64     `db:"eval_latency_ms"`
    EvaluatedAt     time.Time `db:"evaluated_at"`

    // Context fields for debugging
    Country     string `db:"country"`
    Platform    string `db:"platform"`
    Category    string `db:"category"`
    UserSegment string `db:"user_segment"`
    Amount      float64 `db:"amount"`
}

func LogCashbackResult(ctx context.Context, req CashbackRequest, result *CashbackResult) error {
    log := CashbackAuditLog{
        TransactionID:   req.TransactionID,
        AppliedRuleID:   result.AppliedRuleID,
        CashbackPercent: result.CashbackPercent,
        CashbackValue:   result.CashbackValue,
        MaxGranted:      result.MaxGranted,
        EvalLatencyMs:   result.EvalLatencyMs,
        EvaluatedAt:     time.Now(),
        Country:         req.Country,
        Platform:        req.Platform,
        Category:        req.Category,
        UserSegment:     req.UserSegment,
        Amount:          req.Amount,
    }

    // Insert to coins_cashback table
    return db.InsertCashbackLog(ctx, log)
}
```

---

## Database Schema (coins_cashback)

```sql
CREATE TABLE coins_cashback (
    id              BIGSERIAL PRIMARY KEY,
    transaction_id  VARCHAR(64) NOT NULL,
    applied_rule_id VARCHAR(128) NOT NULL,
    cashback_percent DECIMAL(5,2) NOT NULL,
    cashback_value  DECIMAL(15,2) NOT NULL,
    max_granted     DECIMAL(15,2) NOT NULL,
    eval_latency_ms INTEGER NOT NULL,
    evaluated_at    TIMESTAMP NOT NULL DEFAULT NOW(),

    -- Context for debugging/analytics
    country         VARCHAR(8),
    platform        VARCHAR(16),
    category        VARCHAR(64),
    user_segment    VARCHAR(32),
    amount          DECIMAL(15,2),

    -- Indexes
    INDEX idx_transaction_id (transaction_id),
    INDEX idx_applied_rule_id (applied_rule_id),
    INDEX idx_evaluated_at (evaluated_at)
);
```

---

## Metrics & Observability

### Datadog Integration

```go
package metrics

import (
    "github.com/DataDog/datadog-go/statsd"
)

var ddClient *statsd.Client

func RecordCashbackMetrics(result *CashbackResult) {
    // Latency histogram
    ddClient.Histogram(
        "rule_engine.eval.latency",
        float64(result.EvalLatencyMs),
        nil,
        1,
    )

    // Evaluation count
    ddClient.Incr(
        "rule_engine.eval.count",
        []string{
            "rule_id:" + result.AppliedRuleID,
        },
        1,
    )

    // Cashback value distribution
    ddClient.Histogram(
        "rule_engine.cashback.value",
        result.CashbackValue,
        []string{
            "rule_id:" + result.AppliedRuleID,
        },
        1,
    )
}
```

### Key Metrics

| Metric | Type | Tags | Description |
|--------|------|------|-------------|
| `rule_engine.eval.latency` | Histogram | - | Evaluation latency in ms |
| `rule_engine.eval.count` | Counter | `rule_id` | Number of evaluations |
| `rule_engine.matched_rule_id` | Set | - | Distribution of matched rules |
| `rule_engine.fallback_rate` | Gauge | - | % of default rule matches |

---

## Rule Priority Strategy

Rules are ordered from **most specific** to **least specific**:

```
Priority 1: country + platform + category + user_segment (4 dimensions)
Priority 2: country + platform + category (3 dimensions)
Priority 3: country + platform (2 dimensions)
Priority 4: country + category (2 dimensions)
Priority 5: country only (1 dimension)
Priority 6: Default (catch-all)
```

### Rule Naming Convention

```
rule_{segment}_{country}_{category}_{platform}

Examples:
- rule_premium_id_electronics_ios
- rule_standard_id_electronics
- rule_id_default
- rule_global_default
```

---

## Adding New Rules (No Code Changes)

### Step 1: Update Law JSON

Add a new evaluator block to the `evaluators` array at the appropriate priority position:

```json
{
  "type": "evaluator",
  "rule": {
    "type": "rules.conditional.and",
    "rules": [
      { "type": "rules.string.equal", "field": "country", "value": "MY" },
      { "type": "rules.string.equal", "field": "platform", "value": "android" },
      { "type": "rules.string.equal", "field": "category", "value": "food" }
    ]
  },
  "action": {
    "cashback_percent": 8,
    "max_granted": 25000,
    "applied_rule_id": "rule_my_food_android"
  }
}
```

### Step 2: Reload Law

```go
func ReloadCashbackLaw(newLawJSON string) error {
    newLaw, err := law.CreateLaw(newLawJSON)
    if err != nil {
        return err
    }

    // Atomic swap
    core.Register(newLaw)
    return nil
}
```

---

## Performance Considerations

### Expected Latency

| Scenario | Rules | Expected Latency |
|----------|-------|------------------|
| Simple (5 rules) | 5 | <5ms |
| Moderate (50 rules) | 50 | <20ms |
| Complex (200 rules) | 200 | <50ms |

### Optimization Tips

1. **Rule Order**: Place frequently matched rules early
2. **Enable Caching**: Use `"cache": "cache.map"` for repeated inputs
3. **Avoid Deep Nesting**: Keep AND/OR nesting to 2-3 levels max
4. **Batch Evaluations**: If processing multiple transactions, parallelize

---

## Backward Compatibility

### Migration Strategy

```go
// Feature flag for gradual rollout
func GetCashback(ctx context.Context, req CashbackRequest) (*CashbackResult, error) {
    if featureFlags.IsEnabled("use_descartes_cashback") {
        return EvaluateCashback(ctx, req) // New Descartes engine
    }
    return getLegacyCashback(ctx, req) // Old flat config
}
```

### Fallback Rule Matches Legacy Config

Ensure default rule returns same values as current flat config:

```json
{
  "type": "evaluator",
  "rule": { "type": "rules.default" },
  "action": {
    "cashback_percent": 2,  // Current default
    "max_granted": 10000,   // Current max
    "applied_rule_id": "rule_legacy_fallback"
  }
}
```

---

## Testing

### Unit Test Example

```go
func TestCashbackEvaluation(t *testing.T) {
    // Initialize engine
    InitCashbackEngine()

    tests := []struct {
        name     string
        request  CashbackRequest
        expected string // expected rule_id
    }{
        {
            name: "premium user, ID, ios, electronics",
            request: CashbackRequest{
                Country:     "ID",
                Platform:    "ios",
                Category:    "electronics",
                UserSegment: "premium",
                Amount:      100000,
            },
            expected: "rule_premium_id_electronics_ios",
        },
        {
            name: "standard user, unknown category",
            request: CashbackRequest{
                Country:     "ID",
                Platform:    "web",
                Category:    "unknown",
                UserSegment: "standard",
                Amount:      50000,
            },
            expected: "rule_id_default",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := EvaluateCashback(context.Background(), tt.request)
            require.NoError(t, err)
            assert.Equal(t, tt.expected, result.AppliedRuleID)
        })
    }
}
```

### Load Test (k6)

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 25 },  // Normal load
        { duration: '1m', target: 50 },   // Peak load
        { duration: '30s', target: 0 },   // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<200'], // 95% under 200ms
    },
};

export default function () {
    const payload = JSON.stringify({
        transaction_id: `txn_${Date.now()}`,
        country: 'ID',
        platform: 'ios',
        category: 'electronics',
        user_segment: 'premium',
        amount: 150000,
    });

    const res = http.post('http://localhost:8080/api/cashback/evaluate', payload, {
        headers: { 'Content-Type': 'application/json' },
    });

    check(res, {
        'status is 200': (r) => r.status === 200,
        'has applied_rule_id': (r) => JSON.parse(r.body).applied_rule_id !== '',
    });

    sleep(0.1);
}
```

---

## Summary

| Component | Implementation |
|-----------|----------------|
| **Rule Engine** | Descartes with `evaluator.group.first_match` |
| **Priority** | Rule order in JSON array |
| **Dimensions** | country, platform, category, user_segment |
| **Auditability** | `applied_rule_id` in every response |
| **Persistence** | `coins_cashback` table |
| **Metrics** | Datadog: latency, count, rule distribution |
| **Latency Target** | <200ms (achievable with in-memory evaluation) |
| **Extensibility** | Add rules via JSON config, no code changes |

---

## Next Steps

1. **Define all rule combinations** based on business requirements
2. **Create initial law JSON** with priority ordering
3. **Implement service layer** with audit logging
4. **Add metrics collection** for observability
5. **Load test** to validate <200ms latency
6. **Gradual rollout** with feature flags
