---
name: cpi-validator
version: 1.0.1
description: Traces and validates Cross-Program Invocation chains across Solana programs, detecting unsafe CPI patterns, verifying program ownership at each hop, and flagging reentrancy vectors through nested invocations
author: parity-team
license: MIT
tags:
  - security
  - anchor
  - defi
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
  - name: max_depth
    type: number
    default: 3
    description: Maximum CPI chain depth to analyze
outputs:
  - name: findings
    type: Finding[]
  - name: cpi_map
    type: object
    description: Complete CPI call graph with program IDs, account flows, and signer propagation
  - name: score
    type: number
    range: 0-100
  - name: summary
    type: string
---

# CPI Chain Validator Skill

You are tracing and validating every Cross-Program Invocation chain in a Solana program. CPI is how Solana programs compose, but each hop introduces trust boundaries. An unverified program ID at any hop means an attacker can substitute a malicious program that mimics the expected interface. This skill builds a complete CPI call graph and validates safety at every edge.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | `insecure-cpi` pattern with detection hints |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real CPI vulnerabilities from OtterSec, Sec3, Neodyme including reentrancy-via-callback |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct CPI invocation pattern with typed Program accounts |
| Security Audit Skill | `skills/security-audit/SKILL.md` | CPI security checks (Step 5) and reentrancy checks (Step 10) |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: CPI Discovery

Scan the program for every CPI call site:

**Anchor-style CPI:**
- `CpiContext::new(program, accounts)` — unsigned CPI
- `CpiContext::new_with_signer(program, accounts, signer_seeds)` — PDA-signed CPI
- `token::transfer`, `token::mint_to`, `token::burn` and other `anchor_spl` helpers

**Native-style CPI:**
- `invoke(&instruction, &account_infos)` — unsigned
- `invoke_signed(&instruction, &account_infos, &signer_seeds)` — signed

For each CPI call, record:
- Source instruction (which handler makes the call)
- Target program (how the program account is typed/validated)
- Accounts passed (which accounts flow from the caller to the callee)
- Signer seeds (if PDA-signed)
- Line number and code context

## Step 2: Program ID Verification (Critical)

For every CPI target, verify the program ID is validated:

**Safe: Typed Program account**
```rust
// Anchor verifies program ID at deserialization
pub token_program: Program<'info, Token>,
pub system_program: Program<'info, System>,
```

**Unsafe: Raw AccountInfo as program**
```rust
// VULNERABLE: any program can be substituted
pub target_program: AccountInfo<'info>,
// then used in:
invoke(&ix, &[...target_program.to_account_info()])?;
```

**Unsafe: Unchecked UncheckedAccount**
```rust
// VULNERABLE: explicitly unchecked, no program ID validation
/// CHECK: trusted
pub external_program: UncheckedAccount<'info>,
```

For each unsafe CPI target, check whether there is a manual program ID check before the CPI call:
```rust
// Manual check (acceptable but less safe than typed)
require!(
    ctx.accounts.target_program.key() == expected_program::ID,
    ErrorCode::InvalidProgram
);
```

**Finding if violated:**
- Severity: Critical
- Pattern: `insecure-cpi`
- Title: "CPI target program not verified"

## Step 3: Account Flow Analysis (High)

Trace which accounts are passed from the calling program to the CPI target:

1. **Privilege escalation**: Does the CPI pass a signer account that the target program could misuse? If account A is a signer in the calling program, and it's passed to the CPI target, the target can use that signer authority for operations the caller didn't intend.

2. **Writable propagation**: Accounts marked `mut` in the caller's context that are passed to CPI can be modified by the target program. Verify the caller expects this.

3. **Missing accounts**: If the CPI instruction expects accounts that aren't passed, the CPI will fail. Check that the account list is complete.

4. **Account aliasing**: Check whether the same account is passed in multiple positions (e.g., source and destination in a transfer are the same account).

**Finding if applicable:**
- Severity: High
- Title: "Signer privilege escalation through CPI account passing"

## Step 4: Reentrancy via CPI Callback (Critical)

Check the checks-effects-interactions pattern:

1. Identify all state mutations (writing to accounts) in each instruction
2. Identify all CPI calls in each instruction
3. Verify that ALL state mutations happen BEFORE any CPI call

If state is mutated after a CPI call, the target program could potentially callback into the originating program (directly or through a chain), reading state that hasn't been updated yet.

```rust
// VULNERABLE: state updated after CPI
token::transfer(cpi_ctx, amount)?;
ctx.accounts.vault.balance -= amount; // read by reentering call shows old balance

// SECURE: checks-effects-interactions
ctx.accounts.vault.balance -= amount; // update first
token::transfer(cpi_ctx, amount)?;    // then CPI
```

Map every instruction to determine if it follows checks-effects-interactions.

**Finding if violated:**
- Severity: Critical
- Title: "State mutation after CPI call enables reentrancy"

## Step 5: CPI Depth Analysis (Medium)

If the program chains CPI calls (program A calls B which calls C):

1. Map the full call chain up to `max_depth`
2. Verify program ID validation at each hop
3. Check compute budget: each CPI hop costs ~25,000 CU base. Deep chains risk exceeding budget.
4. Flag circular chains (A -> B -> A) which indicate reentrancy risk

**Finding if applicable:**
- Severity: Medium
- Title: "CPI chain depth exceeds 2 hops"

## Step 6: PDA Signer Safety in CPI (High)

When using `CpiContext::new_with_signer` or `invoke_signed`:

1. The signer seeds match the PDA derivation exactly
2. The bump is the canonical bump from account data, not hardcoded
3. The PDA account referenced in the CPI matches the seeds

Reference `skills/pda-helper/SKILL.md` for detailed PDA signer validation patterns.

**Finding if violated:**
- Severity: High
- Title: "CPI signer seeds do not match PDA derivation"

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
  "score": 55,
  "cpi_map": {
    "deposit": {
      "calls": [
        {
          "target": "Token Program",
          "verified": true,
          "method": "typed_program",
          "operation": "transfer",
          "line": 67,
          "accounts_passed": ["user_token", "vault_token", "authority"],
          "signer_type": "user"
        }
      ],
      "state_before_cpi": true,
      "reentrancy_safe": true
    },
    "swap": {
      "calls": [
        {
          "target": "Unknown",
          "verified": false,
          "method": "raw_accountinfo",
          "line": 134
        }
      ],
      "state_before_cpi": false,
      "reentrancy_safe": false
    }
  },
  "findings": [...],
  "summary": "8 CPI calls across 4 instructions. 1 critical (unverified program ID in swap), 1 critical (state after CPI in swap). deposit and withdraw are safe."
}
```
