---
name: pda-helper
version: 1.0.0
description: Assists with Program Derived Address patterns, validates seed construction, bump handling, canonical bump usage, and cross-program PDA access
author: parity-team
license: MIT
tags:
  - anchor
  - tooling
  - accounts
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
outputs:
  - name: findings
    type: Finding[]
  - name: pda_map
    type: object
  - name: score
    type: number
    range: 0-100
  - name: summary
    type: string
---

# PDA Derivation Helper Skill

You are analyzing all Program Derived Address patterns in a Solana program. PDAs are the primary mechanism for deterministic account addressing on Solana, and their misuse is a leading source of critical vulnerabilities. This skill produces a complete PDA map and validates every derivation against known-safe patterns.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | `unvalidated-pda` pattern with detection hints |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct PDA derivation and account init patterns |
| Security Audit Skill | `skills/security-audit/SKILL.md` | PDA validation checks (Step 4) |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: PDA Discovery

Scan the entire program and catalog every PDA reference:

**In `#[derive(Accounts)]` structs:**
- `seeds = [...]` constraints
- `bump` and `bump = account.field` constraints
- `init` with seeds (PDA creation)

**In instruction handlers:**
- `Pubkey::find_program_address` calls
- `Pubkey::create_program_address` calls
- `invoke_signed` with signer seeds

For each PDA, record: account name, seed components (type and source), bump handling, referencing instructions, and whether it is used as a CPI signer.

## Step 2: Seed Determinism Validation (Critical)

Verify seeds are fully deterministic and not attacker-controlled.

**Safe seeds:** string literals (`b"vault"`), account public keys (`.key().as_ref()`), numeric IDs (`.to_le_bytes()`), program-controlled state.

**Unsafe seeds:** user-supplied strings without length validation, unbounded byte arrays.

```rust
// VULNERABLE: unbounded user input
seeds = [b"profile", user_input.as_bytes()]

// SECURE: deterministic, known-length key
seeds = [b"profile", user.key().as_ref()]
```

**Finding if violated:** Severity Critical, Pattern `unvalidated-pda`.

## Step 3: Canonical Bump Handling (High)

Verify every PDA stores and reuses its canonical bump:

1. On `init`: bump is stored in the account data
2. On access: `bump = account.bump` reuses stored bump (avoids 1,500 CU re-derivation)
3. Consistency: stored bump must be canonical

```rust
// VULNERABLE: re-derives each time
seeds = [b"vault", authority.key().as_ref()], bump,

// SECURE: uses stored bump
seeds = [b"vault", authority.key().as_ref()], bump = vault.bump,
```

## Step 4: Seed Collision Analysis (Critical)

Check for potential PDA collisions:

1. **Prefix uniqueness**: every PDA type has a unique string prefix
2. **Length ambiguity**: `[b"ab", b"cd"]` == `[b"abcd"]` without separator
3. **Cross-instruction collision**: PDAs from instruction A passable to instruction B as different type

## Step 5: PDA as Signer (High)

If PDAs are used as CPI signers:

1. Signer seeds must exactly match derivation seeds
2. Bump must be canonical (from stored account data, not hardcoded)
3. PDA account must be validated with constraints

```rust
// VULNERABLE: hardcoded bump
let seeds = &[b"authority", &[255u8]];

// SECURE: stored bump
let seeds = &[b"authority", &[ctx.accounts.auth_pda.bump]];
```

## Step 6: Cross-Program PDA Access (Medium)

For PDAs owned by other programs: verify owner validation, correct external program ID in derivation, and matching seed patterns.

## Scoring

| Severity | Penalty |
|----------|---------|
| Critical | -25 |
| High | -15 |
| Medium | -8 |
| Info | -3 |

## Output Format

```json
{
  "score": 70,
  "pda_map": {
    "vault": {
      "seeds": ["\"vault\"", "authority.key()"],
      "bump_handling": "stored_canonical",
      "created_in": "initialize",
      "accessed_in": ["deposit", "withdraw"],
      "used_as_signer": false
    }
  },
  "findings": [...],
  "summary": "5 PDAs analyzed. 1 high (bump re-derivation). No collision risks."
}
```
