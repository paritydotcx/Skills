---
name: gas-optimization
version: 1.2.0
description: Compute unit and rent cost optimization analysis for Solana programs
author: parity
license: MIT
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
    description: Program framework (anchor | native)
  - name: compute_budget
    type: number
    default: 200000
    description: Target compute unit budget per instruction
outputs:
  - name: findings
    type: Finding[]
  - name: estimated_compute_units
    type: object
    description: Per-instruction compute unit estimates
  - name: summary
    type: string
---

# Gas Optimization Skill

You are analyzing a Solana smart contract for compute unit efficiency and rent cost optimization. Solana charges per compute unit consumed and per byte of on-chain storage. This skill identifies wasteful patterns that increase transaction costs, reduce throughput, or cause unnecessary rent burden.

## Context

Solana programs have a default compute budget of 200,000 compute units per instruction. Programs can request up to 1,400,000 CUs via `ComputeBudgetInstruction::set_compute_unit_limit`, but higher CU consumption means higher priority fees and reduced inclusion probability during congestion. Anchor adds overhead for account deserialization and constraint validation. Minimizing CU usage per instruction directly reduces user costs and improves scalability.

Rent is charged at ~6.96 SOL per MB per year (rent-exempt threshold). Every byte of on-chain account data has a permanent cost. Over-allocating account space wastes rent deposits.

## Context Sources

Before starting analysis, fetch the following from the Parity repository:

| Source | Path | Contains |
|--------|------|----------|
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct Anchor patterns including data layout, CPI invocation, and account initialization examples |
| Skill Definition | `programs/parity/src/skills.rs` > `BUILTIN_SKILLS` (gas-optimization entry) | Canonical step list and output spec |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

Use `ANCHOR_PATTERNS` to compare the target program's account structures and CPI calls against known efficient implementations.

## Step 1: Account Data Layout

Analyze all `#[account]` struct definitions for data packing efficiency.

**What to look for:**
- Field ordering that causes padding bytes (Rust struct layout follows field declaration order without automatic packing in Borsh serialization, but alignment still matters for cache efficiency)
- Use of `String` where fixed `[u8; N]` would suffice (String stores length prefix + heap data, more expensive to serialize/deserialize)
- `Vec<T>` fields where a fixed array `[T; N]` could work (Vec requires length prefix and dynamic allocation)
- `Option<Pubkey>` (33 bytes: 1 discriminator + 32 key) where a zero-key sentinel could save space
- Unused or deprecated fields that still consume space

**Wasteful pattern:**
```rust
#[account]
pub struct UserProfile {
    pub name: String,           // 4 + N bytes, variable
    pub bio: String,            // 4 + N bytes, variable
    pub avatar_url: String,     // 4 + N bytes, variable
    pub authority: Pubkey,      // 32
    pub created_at: i64,        // 8
}
```

**Optimized pattern:**
```rust
#[account]
pub struct UserProfile {
    pub authority: Pubkey,       // 32
    pub created_at: i64,         // 8
    pub name: [u8; 32],          // 32, fixed
    pub bio: [u8; 128],          // 128, fixed
    pub avatar_hash: [u8; 32],   // 32, store hash, not full URL
}
```

**Finding if applicable:**
- Severity: Medium
- Title: "Variable-length field increases account size and deserialization cost"
- Description: "String or Vec fields require dynamic length handling during serialization. If the maximum length is known, use fixed-size arrays to reduce compute overhead and make space calculation deterministic."

## Step 2: Account Reallocation

Check for unnecessary `realloc` operations or patterns that create accounts and immediately resize them.

**What to look for:**
- `realloc` constraints on accounts that could have been initialized with the correct size
- Repeated reallocation in hot-path instructions
- Growing Vec fields that trigger reallocation on append

**Finding if applicable:**
- Severity: Medium
- Title: "Avoidable account reallocation"
- Description: "Account is reallocated during a high-frequency instruction. Pre-allocating sufficient space at initialization eliminates the realloc compute overhead (~5,000 CU per realloc)."

## Step 3: Deserialization Overhead

Identify accounts that are deserialized unnecessarily.

**What to look for:**
- Large accounts deserialized in instructions that only read one or two fields
- Accounts declared as `Account<'info, T>` (full deserialization) where `AccountLoader<'info, T>` (zero-copy) would be appropriate for accounts over 1KB
- Multiple instructions that fully deserialize the same large account type

**Optimized pattern:**
```rust
// For large accounts (>1KB), use zero-copy to avoid full deserialization
#[account(zero_copy)]
#[repr(C)]
pub struct LargeState {
    pub data: [u64; 128],  // 1KB, accessed field-by-field without full deser
}

// In accounts struct:
pub large_state: AccountLoader<'info, LargeState>,
```

**Finding if applicable:**
- Severity: Medium
- Title: "Large account uses full deserialization"
- Description: "Account data exceeds 1KB and is fully deserialized on every instruction call. Using AccountLoader with zero_copy allows field-level access without deserializing the entire account, saving ~10 CU per byte."

## Step 4: Compute Unit Estimation

Estimate the compute unit consumption of each instruction handler.

**Base costs to reference:**
- Account deserialization (Borsh): ~2,000-5,000 CU per account depending on size
- Account constraint validation: ~500 CU per constraint
- PDA derivation (`find_program_address`): ~1,500 CU per derivation
- CPI call: ~25,000 CU base + target program cost
- SHA256 hash: ~100 CU per 64 bytes
- `msg!` logging: ~100 CU per call
- Arithmetic operations: negligible (~1-10 CU)

For each instruction, sum the estimated costs and compare against the `compute_budget` input parameter (default 200,000). Flag instructions that exceed 70% of budget.

**Finding if applicable:**
- Severity: High
- Title: "Instruction may exceed compute budget"
- Description: "Estimated compute consumption exceeds 70% of the default 200,000 CU budget. Under network congestion, this instruction may fail or require explicit compute budget increase."

## Step 5: CPI Overhead

Evaluate cross-program invocations for batching opportunities.

**What to look for:**
- Multiple CPI calls to the same program in a single instruction (could batch via instruction data)
- CPI calls inside loops
- CPI calls that could be replaced with direct logic (e.g., manual lamport transfer instead of System Program CPI for simple SOL transfers within owned accounts)

**Finding if applicable:**
- Severity: Medium
- Title: "Repeated CPI to same program in single instruction"
- Description: "Multiple CPI calls to the same program within one instruction. Each CPI call costs ~25,000 CU base overhead. Consider batching operations or restructuring to minimize cross-program call count."

## Step 6: Rent Optimization

Calculate total rent cost for all account types and identify savings opportunities.

**Calculation:**
- Rent-exempt minimum = (account_size + 128) * 2 * rent_per_byte_year / 365.25 / 24 / 3600 * 2_years
- Roughly: 0.00089088 SOL per byte (at current rates)

For each `#[account]` type, compute:
1. Current space (including 8-byte discriminator)
2. Minimum achievable space with field optimization
3. Rent delta in SOL and lamports

**Finding if applicable:**
- Severity: Info
- Title: "Account space can be reduced by N bytes"
- Description: "Field layout optimization could reduce account size by N bytes, saving approximately X lamports in rent-exempt deposit per account instance."

## Output Format

Return results as structured JSON:

```json
{
  "estimated_compute_units": {
    "initialize": 45000,
    "deposit": 72000,
    "withdraw": 95000,
    "close": 38000
  },
  "findings": [
    {
      "severity": "medium",
      "title": "Variable-length String field in UserProfile",
      "location": {
        "file": "lib.rs",
        "line": 12,
        "instruction": null
      },
      "description": "UserProfile.name uses String type. Maximum name length is 32 characters. Using [u8; 32] saves ~4 bytes per account and eliminates length-prefix deserialization.",
      "recommendation": "Replace `pub name: String` with `pub name: [u8; 32]` and update serialization logic."
    }
  ],
  "summary": "Total estimated rent across all account types: 0.0089 SOL. 2 optimization opportunities identified saving ~12,000 CU on the deposit instruction."
}
```
