# Sovereign Agent Fleet: Exhaustive Codebase Explanation

This document provides an exhaustive, in-depth explanation of the **Sovereign Agent Fleet** project. This project implements a sovereign cognitive control plane for governing probabilistic agents and their actions. Its central thesis is that **cognition is probabilistic, but authority must be deterministic.** It enforces this by separating AI model inference from authorization, execution, and verification.

This explanation covers the architecture, the implementation details, the cryptographic structures, domain adapters, security invariants, and the test suite.

---

## 1. The Core Thesis: Authority Non-Equivalence Principle

The entire architecture is built around one foundational principle: **The Authority Non-Equivalence Principle (Invariant 1).**

> `ModelOutput ≠ Authorization`

A language model's output—regardless of its confidence, logic, or semantic plausibility—never constitutes permission to execute a consequential action. An AI agent does not govern itself. Instead, the model acts as an untrusted **epistemic** component (it forms beliefs and makes proposals). The actual authority to act is decided by an entirely separate, deterministic **governance protocol**.

The pipeline follows an explicitly ordered **trust-transition sequence**:

```text
knowledge → compilation → retrieval → cognition → proposal
  → authorization → execution → verification → evidence
```

This sequence represents the crossing of two hard architectural boundaries:
1. **The Authority Boundary:** Where an untrusted proposal becomes an authorized action.
2. **The Execution Boundary:** Where an authorized action happens and is independently verified.

---

## 2. The Three Trust Domains

The system separates concerns into three distinct domains. This guarantees that an answer to one question is never mistaken for an answer to another.

### 2.1 The Epistemic Domain (Cognition)
*   **Question:** What does the system believe? Is the belief correct?
*   **Components:** RAG/Retrieval, Knowledge Compilers (like A10), Language Models, Quantitative analysis layers (`exchange/quant/`).
*   **Output:** Observations, Evidence, Beliefs, and untrusted **Proposals**.
*   **Rule:** This domain has zero authority. If the model is poisoned, hallucinating, or hijacked (Prompt Injection), the worst it can do is make a bad proposal.

### 2.2 The Authority Domain (Governance)
*   **Question:** What is the system permitted to do? Was the action legitimately authorized?
*   **Components:** Identity Certificates (`fleet/crypto/foundation.py`), Capability Scopes (`fleet/epistemic/scope.py`), Deterministic Policy (`fleet/epistemic/decision.py`).
*   **Output:** A signed `AuthorizationDecision` and (if required) a human `ApprovalRecord`.
*   **Rule:** This domain consults *no epistemic output* (no probability, no confidence score). It only looks at the cryptographic grant, the requested scope, and deterministic policy rules.

### 2.3 The Execution Domain (Verification & Audit)
*   **Question:** What actually happened? Did the authorized action happen as recorded?
*   **Components:** Executor, Independent Verifier, Signed Hash-Chain Audit Ledger (`fleet/crypto/chriscrypt/ledger.py`).
*   **Output:** Cryptographic evidence of the state transition.
*   **Rule:** The executor is not trusted to self-report success. A verifier recomputes the expected state from signed inputs. All actions are written to a tamper-evident ledger.

---

## 3. The `fleet/` Substrate: Deep Dive

The `fleet/` directory contains the general-purpose, domain-agnostic governance substrate. It provides the cryptographic foundation, policy enforcement, and audit ledger.

### 3.1 Cryptographic Identity & Envelope (`fleet/crypto/`)
The system does not rely on third-party cloud identity (like AWS IAM) for agent authority; it uses a sovereign, local-first root of trust.

*   **Root of Trust (`foundation.py`):** Derived using **Argon2id** (memory-hard KDF). The Master Secret generates a root **Ed25519** signing key.
*   **Agent Certificates (`AgentCert`):** Every agent (e.g., Researcher, Analyst, Operator) holds a root-signed `AgentCert` containing its `agent_id`, `role`, `capabilities`, and expiration. Because the agent doesn't hold the root key, it cannot forge or escalate its own capabilities.
*   **Key Rotation (K1):** The root key can be rotated. Live agent certs are re-issued under the new root, but the epoch increments. Verifiers hold `known_root_pubs` to maintain chain continuity for historical records.
*   **Confidentiality (`chriscrypt/envelope.py`):** Uses **XChaCha20-Poly1305** (or AES-GCM) with an HKDF-derived per-record subkey. This ensures that secrecy (encryption) and integrity (signatures) are handled correctly.

### 3.2 The Epistemic/Governance Boundary (`fleet/epistemic/`)
This is the most critical code in the project. It defines the exact types that cross the boundary and the function that decides them.

*   **Scopes (`scope.py`):** Distinguishes between what an agent can *know* (`EpistemicScope`), *propose* (`ProposalScope`), and *actually do* (`AuthorizationScope`). They are never merged into one "authority blob."
*   **The Request (`authorization.py`):** `AuthorizationRequest` is the final epistemic object. It asks for permission. It grants nothing.
*   **The Grant (`authority.py`):** `AuthorityGrant` is a root-signed capability. It is externally issued and bound to a specific agent and epoch.
*   **The Frozen `decide()` Function (`decision.py`):**
    This is the beating heart of the Sovereign Agent Fleet. It is a **pure, deterministic function** that takes an `AgentIdentity`, an `AuthorityGrant`, an `AuthorizationScope`, and deterministic `GovernanceConstraints`.
    *   **It specifically excludes:** probability, confidence, model score, belief, or calibration values.
    *   It verifies the grant signature against a pinned `trusted_issuer_pubkey_pem` (not the key embedded in the grant, preventing self-signed forgery).
    *   It checks epoch currency, identity binding, and capability scoping.
    *   It returns one of three verdicts: `AUTO`, `HUMAN`, or `BLOCKED`.

### 3.3 The Tamper-Evident Ledger (`fleet/crypto/chriscrypt/ledger.py`)
All consequential state transitions are written to an append-only, Ed25519-signed hash chain.
*   Each entry commits to the hash of the previous entry.
*   A signed `checkpoint` records the chain head and sequence length. This makes **tail truncation** (deleting the last few records) detectable.
*   Replay attacks, mutations, and truncations are immediately flagged by `Ledger.verify_chain()`.

---

## 4. The M0 Domain Generality (Consolidation)

A key claim of the research is that the governance substrate is **domain-general**. The exact same frozen `decide()` function is used to govern AI agents in Finance, Incident Response, Supply Chain, Scientific Research, and Energy Grids.

*   **`domain_registry/`**: This module proves the M0 meta-invariant. It contains `REGISTERED_CAPABILITIES` (e.g., `exchange/finance` -> `CAP_TRADE_EXECUTE`, `incident/security` -> `CAP_INCIDENT_REMEDIATE`).
*   **Cross-Domain Suite (`test_registry_cross_domain_generality.py`):**
    *   Proves that the semantic domain is irrelevant to `decide()`; only the literal capability and policy matter.
    *   Proves that flipping a policy (e.g., `require_human_approval=True`) changes all registered domains from `AUTO` to `HUMAN` simultaneously.
    *   Proves that a scoped grant in one domain cannot authorize a universal capability in another.
    *   Enforces an **Import Wall**: `fleet.epistemic` never imports any domain adapters.

---

## 5. Domain Implementations

While the substrate is domain-agnostic, the repo contains specific implementations to stress-test the architecture.

### 5.1 Financial Workload (`exchange/`)
This is the flagship exemplar. Finance is used because it perfectly exposes probabilistic reasoning (quant), risk, consequential actions, and verification.
*   Contains a venue matching engine, order books, and settlement.
*   `exchange/quant/`: The quantitative cognition layer. It uses Kelly sizing, Bayesian updates, and edge estimation. Crucially, this output is *Evidence*, not *Authority*.
*   **ZK Attestation (D24):** Contains `exchange/quant/zk.py`, a genuine Σ-protocol (Zero-Knowledge Proof) that proves a learned prior lies within a public risk range *without revealing the actual prior value*.

### 5.2 Incident Response Workload (`incident/`)
Proves that the architecture works for non-financial domains.
*   Uses a `SimEnv` to model workloads (`web-edge`, `revenue-svc`).
*   An AI agent acts as a Researcher/Analyst to produce `QualifiedIntel`.
*   An Operator agent requests to remediate (e.g., "isolate revenue-svc").
*   If the policy demands it, the transition halts until a cryptographically bound `ApprovalRecord` is signed by a human. (e.g., `test_pathB_high_requires_human`).
*   **Second-line defense:** Even if an agent bypasses the pipeline and calls `SimEnv.transition` directly, protected assets refuse containment.

---

## 6. Security and Threat Model (A1-A6)

The project defines six adversaries and maps them to architectural defenses. The test suite heavily fuzzes these boundaries.

1.  **A1 (Rogue Model):** A model hallucinating or acting maliciously is stopped by **Invariant 1** (`ModelOutput ≠ Authorization`).
2.  **A2 (Prompt Injection):** Stopped by epistemic isolation and the policy boundary.
3.  **A3 (Privilege Escalation / Forged Identity):** Stopped by cryptographic identity (Ed25519 cert chain) and capability scoping. (e.g., `test_forged_grant_signature_rejected`).
4.  **A4 (Executor Deception):** The executor falsely claims success. Stopped by independent verification recomputing the state hash (e.g., `test_verifier_critical_on_tamper`).
5.  **A5 (Audit Tampering):** Stopped by the signed hash-chain ledger.
6.  **A6 (Knowledge Poisoning):** This is the **Open Problem**. If the source knowledge is poisoned, the model reasons correctly from bad data, policy approves it, and execution succeeds. The architecture guarantees the *process* was valid, but cannot guarantee the *truth* of the underlying knowledge.

### 6.1 The Blind Adversary Harness
Unlike standard conformance tests, the suite includes `test_adversarial_blind_harness.py`. This is a threat-model-agnostic fuzzer that throws 5,000 randomized malformed, forged, and out-of-scope requests at `decide()`. The result: **0 false authorizations.**

---

## 7. Execution State Machine

Consequential actions do not happen in one step. They traverse a strict state machine:
`REQUEST → INTENT → PLAN → ACTION → TOOL → OBSERVATION → EVIDENCE → VERIFICATION → ARTIFACT → APPROVAL → FINAL → AUDIT`

This sequence is enforced by the `Runtime` layer (`fleet/layers/runtime.py`). It prevents a model from collapsing the pipeline into a single "Observe->Act" loop. Every transition is validated, human approval (if needed) acts as a hard gate before `FINAL`, and every step is persisted to the audit ledger.

---

## Summary

The Sovereign Agent Fleet is not a multi-agent orchestration framework (like LangChain or AutoGen). It is a **Governance Substrate**. It treats AI brains as dangerous, untrusted heuristic engines. By forcing all AI proposals through a frozen, deterministic cryptographic bottleneck (`fleet/epistemic/decide()`), it guarantees that no matter how capable, confident, or compromised an AI model becomes, it can never unilaterally escalate its own authority to perform consequential actions.
