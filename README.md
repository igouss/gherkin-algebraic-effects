# Gherkin with ZIO-Style Algebraic Effects

> **Executable Specifications for AI-Driven Development, Model-Based Testing (MBT), and Deterministic Simulation Testing (DST)**

[![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE-MIT)
[![Reference Implementation](https://img.shields.io/badge/compiler-SpecForge-orange.svg)](https://github.com/igouss/specforge)

---

## 1. Motivation: The Paradigm Shift

Traditional Behavior-Driven Development (BDD) was designed for human stakeholders: business analysts wrote prose scenarios, and engineers manually wired glue code using regex step bindings and mock libraries. While readable, traditional scenarios suffer from:
- **Ambiguous operational semantics**: Prose cannot unambiguously specify state machine transitions or side-effects.
- **Mock brittleness**: Testing side-effects requires mock frameworks (`mockall`, `mockito`), coupling tests to implementation details and leaking dirty I/O into unit tests.
- **Lack of generative capability**: Hand-crafted scenarios cannot be explored by Model-Based Testing (MBT) or fuzzed with Property-Based Testing (PBT).

### When AI Maintains the Specifications
When software is developed and maintained by **AI coding agents**, Gherkin ceases to be just an informal communication tool. It becomes a **formal, executable Intermediate Representation (IR)**:
1. **Unambiguous**: Carries explicit state machine transitions, exact state deltas, and typed effect descriptors.
2. **Pure & Mock-Free**: Operations are modeled as pure mathematical transition functions:
   $$\text{step}: (\text{State}, \text{Action}, \text{Input}) \to (\text{NextState}, \text{Mutations}, \text{Vec}\langle\text{EmittedEffect}\rangle)$$
3. **Universally Executable**: Valid Gherkin that parses cleanly in standard Cucumber tools (`cucumber-rs`), while enabling MBT path generation, PBT input fuzzing, and DST deterministic chaos testing.

---

## 2. Specification Schema

Any Gherkin step can declare an algebraic transition contract inside a JSON `DocString`:

```gherkin
Feature: Escrow Settlement
  Scenario: Customer submits payment
    When the customer submits payment of $50
      """json
      {
        "expected_pure_transition": {
          "next_state": "HoldingInEscrow",
          "state_mutations": {
            "escrow_balance": "+= 5000",
            "locked_amount": "+= 5000"
          },
          "emitted_effects": [
            {
              "kind": "AsyncRpcCall",
              "target": "LedgerService",
              "method": "LockAccountFunds",
              "timeout_ms": 1500,
              "is_interruptible": true
            },
            {
              "kind": "EmitKafkaEvent",
              "target": "escrow.events.v1",
              "payload": "{\"status\": \"FUNDS_LOCKED\"}"
            }
          ],
          "error_channel": {
            "on_rejection": "NotifyBuyerPaymentFailed",
            "error_type": "InsufficientFunds",
            "retryable": false
          }
        }
      }
      """
```

### Schema Definitions

| Field | Type | Description |
|---|---|---|
| `next_state` | `string` (optional) | The explicit target state in the behavioral model. |
| `state_mutations` | `map<string, string>` | Key-value mapping of state field changes or deltas (e.g. `+= 5000`, `= true`). |
| `emitted_effects` | `array<EmittedEffect>` | List of pure side-effect descriptors emitted by the transition. |
| `emitted_effects[].kind` | `string` | Kind of effect (e.g. `AsyncRpcCall`, `EmitKafkaEvent`, `ScheduleTimer`, `WriteDisk`). |
| `emitted_effects[].target` | `string` (optional) | Target service, topic, or actor. |
| `emitted_effects[].method` | `string` (optional) | Specific operation or RPC method. |
| `emitted_effects[].payload` | `string` (optional) | Serialized message body or parameters. |
| `emitted_effects[].timeout_ms` | `integer` (optional) | Maximum allowed execution time in virtual or wall time. |
| `emitted_effects[].is_interruptible` | `boolean` | Whether preemption or cancellation can abort this effect. |
| `error_channel` | `object` (optional) | Typed error channel specification for rejected transitions. |
| `error_channel.on_rejection` | `string` (optional) | Fallback action or notification emitted upon rejection. |
| `error_channel.error_type` | `string` (optional) | Typed domain error code. |
| `error_channel.retryable` | `boolean` | Whether the operation is safe to retry. |

---

## 3. The Three Levels of "Effect Matching"

Algebraic effects are inert data structures. This enables three powerful matching capabilities across the software lifecycle:

```
                      ┌────────────────────────────────────────┐
                      │    Algebraic Effect Specification      │
                      │  (EmittedEffect in Gherkin DocString)  │
                      └───────────────────┬────────────────────┘
                                          │
            ┌─────────────────────────────┼─────────────────────────────┐
            ▼                             ▼                             ▼
   1. Test Verification         2. Code Implementation        3. DST Simulation
   ────────────────────         ──────────────────────        ─────────────────
   Match Actual vs Expected     Exhaustive Pattern Matching   Match to Inject Chaos
   Effects as Pure Data         via Closed Rust Enums         (Drops, Partitions, Delays)
```

### Level 1: Test Verification Matching (No Mocks)
The System Under Test (SUT) does not perform dirty I/O during business transitions. Instead, it returns `(NextState, Vec<EmittedEffect>)`. The test harness asserts that the actual effects match the specification:

```rust
let (next_state, actual_effects) = sut.handle_step("submit_payment", &payload);

// Assert state transition:
assert_eq!(next_state, "HoldingInEscrow");

// Assert emitted effects match specification:
let rpc_emitted = actual_effects.iter().any(|e| {
    e.kind == "AsyncRpcCall" 
        && e.target.as_deref() == Some("LedgerService")
        && e.method.as_deref() == Some("LockAccountFunds")
});
assert!(rpc_emitted, "Expected AsyncRpcCall to LedgerService was not emitted");
```

### Level 2: Implementation Matching (Exhaustive Pattern Match)
Coding agents implement effects as closed Rust enums (Algebraic Data Types). Production interpreters use exhaustive `match` expressions without wildcard (`_ =>`) arms:

```rust
pub enum DomainEffect {
    AsyncRpcCall { target: String, method: String, timeout_ms: u64 },
    EmitKafkaEvent { target: String, payload: String },
    ScheduleTimer { delay_ms: u64 },
}

// In the production runner:
for effect in emitted_effects {
    match effect {
        DomainEffect::AsyncRpcCall { target, method, timeout_ms } => {
            rpc_client.call(&target, &method, timeout_ms).await?;
        }
        DomainEffect::EmitKafkaEvent { target, payload } => {
            kafka_producer.publish(&target, &payload).await?;
        }
        DomainEffect::ScheduleTimer { delay_ms } => {
            tokio::time::sleep(Duration::from_millis(delay_ms)).await;
        }
    }
}
```
*If an AI spec agent introduces a new effect kind in a `.feature` file, the Rust compiler fails until the coding agent implements a handler for it.*

### Level 3: Simulation Matching (DST Fault Injection)
In Deterministic Simulation Testing (DST) with tools like **Turmoil** or **Madsim**, the virtual runtime intercepts effects by matching on their descriptor:

```rust
// Inside a Turmoil simulated cluster:
match effect.kind.as_str() {
    "AsyncRpcCall" if sim.is_partitioned(&effect.target) => {
        // Matched! Simulated network partition drops this call
        sim.drop_call(effect);
    }
    "AsyncRpcCall" => {
        // Matched! Simulate 150ms network delay before delivering
        sim.advance_virtual_clock_and_deliver(effect, 150);
    }
    _ => sim.execute_default(effect),
}
```

---

## 4. Complete End-to-End Example

```gherkin
@SPEC:ESCROW-001 @VERSION:2.0.0
@MODEL:EscrowStatechart
@PROP:ConservationOfFunds
Feature: Distributed Escrow Settlement Engine

  Scenario: Buyer deposits funds under simulated network latency
    Given the escrow contract is in state "PendingDeposit"
    
    When the engine evaluates action "ProcessDeposit"
      """json
      {
        "expected_pure_transition": {
          "next_state": "HoldingInEscrow",
          "state_mutations": {
            "escrow_balance": "+= 10000",
            "locked_amount": "+= 10000"
          },
          "emitted_effects": [
            {
              "kind": "AsyncRpcCall",
              "target": "LedgerService",
              "method": "LockAccountFunds",
              "timeout_ms": 1500,
              "is_interruptible": true
            },
            {
              "kind": "EmitAuditLog",
              "target": "audit.escrow",
              "payload": "Deposit of 10000 cents held"
            }
          ],
          "error_channel": {
            "on_rejection": "NotifyBuyerPaymentFailed",
            "error_type": "InsufficientFunds",
            "retryable": false
          }
        }
      }
      """

    Then the escrow contract state is "HoldingInEscrow"
    And invariant "ConservationOfFunds" holds
```

---

## 5. Reference Rust Types

These types are implemented in [SpecForge](https://github.com/igouss/specforge) (`specforge::model`):

```rust
use serde::{Deserialize, Serialize};
use std::collections::BTreeMap;

#[derive(Clone, Debug, Eq, PartialEq, Hash, Serialize, Deserialize)]
pub struct EmittedEffect {
    pub kind: String,
    pub target: Option<String>,
    pub method: Option<String>,
    pub payload: Option<String>,
    pub timeout_ms: Option<u64>,
    pub is_interruptible: bool,
}

#[derive(Clone, Debug, Default, Eq, PartialEq, Hash, Serialize, Deserialize)]
pub struct ErrorChannelSpec {
    pub on_rejection: Option<String>,
    pub error_type: Option<String>,
    pub retryable: bool,
}

#[derive(Clone, Debug, Default, Eq, PartialEq, Hash, Serialize, Deserialize)]
pub struct AlgebraicEffectSpec {
    pub next_state: Option<String>,
    pub state_mutations: BTreeMap<String, String>,
    pub emitted_effects: Vec<EmittedEffect>,
    pub error_channel: Option<ErrorChannelSpec>,
}
```

---

## 6. Open-Source Ecosystem Integration

| Layer | Tools | Role with Algebraic Gherkin |
|---|---|---|
| **Specification Compiler** | [**SpecForge**](https://github.com/igouss/specforge) | Compiles Gherkin DocStrings into `SpecificationGraph` and extracts explorable `BehavioralModel`s. |
| **BDD Execution** | [**`cucumber-rs`**](https://github.com/cucumber-rs/cucumber) | Standard test runner executing scenarios and step regex bindings. |
| **Property-Based Testing** | [**`proptest`**](https://github.com/proptest-rs/proptest) | Fuzzes parameters across algebraic state transitions with deterministic shrinking. |
| **Deterministic Simulation** | [**`turmoil`**](https://github.com/tokio-rs/turmoil) / [**`madsim`**](https://github.com/madsim-rs/madsim) | Intercepts emitted effects to simulate network drops, delays, and crashes in a deterministic loop. |

---

## 7. License

Licensed under either of:
- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.
