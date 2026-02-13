---
name: idl-docs
version: 1.3.0
description: Automatically generates comprehensive API documentation from Anchor IDL files with instruction signatures, account layouts, error codes, and usage examples
author: parity-team
license: MIT
tags:
  - tooling
  - anchor
  - documentation
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Anchor IDL JSON file or the program source (.rs)
  - name: format
    type: string
    default: markdown
    description: Output format (markdown | html | json)
  - name: include_examples
    type: boolean
    default: true
    description: Whether to generate TypeScript usage examples for each instruction
outputs:
  - name: documentation
    type: string
    description: Complete API documentation
  - name: type_definitions
    type: string
    description: TypeScript type definitions matching the IDL
---

# IDL Documentation Generator Skill

You are generating comprehensive API documentation from an Anchor program's IDL (Interface Description Language). The IDL is the canonical interface contract between the on-chain program and its clients. The documentation must be complete enough for a frontend developer to integrate with the program without reading the Rust source.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Anchor account patterns to explain in documentation context |
| Best Practices Skill | `skills/best-practices/SKILL.md` | Convention checks including error message quality and event documentation |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

## Step 1: IDL Parsing

If the input is a `.json` IDL file, parse it directly. If the input is a `.rs` source file, extract the interface by analyzing:

- `#[program]` module — instruction handler signatures
- `#[derive(Accounts)]` structs — account inputs per instruction
- `#[account]` structs — stored account layouts with field types and sizes
- `#[error_code]` enum — error codes with messages
- `#[event]` structs — emitted events with field types

Build a structured representation of the program interface.

## Step 2: Program Overview

Generate a header section:

```markdown
# Program Name

**Program ID:** `YOUR_PROGRAM_ID`
**Framework:** Anchor vX.X.X
**Instructions:** N
**Account Types:** N
**Events:** N
**Errors:** N
```

Include a table of contents linking to each instruction, account type, event, and error.

## Step 3: Instruction Documentation

For each instruction, generate:

### Instruction Name

**Description:** Inferred from handler logic and parameter names.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| amount | u64 | Amount of tokens to deposit |

**Accounts:**

| Name | Type | Mutable | Signer | Description |
|------|------|---------|--------|-------------|
| vault | Account\<Vault\> | Yes | No | PDA-derived vault account |
| authority | Signer | No | Yes | Transaction signer and vault owner |
| system_program | Program\<System\> | No | No | Solana System Program |

**PDA Seeds** (if any accounts are PDA-derived):
```
vault: ["vault", authority.key()]
```

**Errors that may be thrown:**
- `InvalidAmount` — Amount must be greater than zero
- `Overflow` — Arithmetic overflow in balance calculation

**Events emitted:**
- `DepositEvent` — Emitted on successful deposit with amount and new balance

**TypeScript Example** (if `include_examples` is true):
```typescript
await program.methods
  .deposit(new anchor.BN(1_000_000))
  .accounts({
    vault: vaultPda,
    authority: wallet.publicKey,
    systemProgram: SystemProgram.programId,
  })
  .rpc();
```

## Step 4: Account Type Documentation

For each `#[account]` type:

### Account Name

**Space:** 8 (discriminator) + N bytes

| Field | Type | Size | Description |
|-------|------|------|-------------|
| authority | Pubkey | 32 | Account owner |
| balance | u64 | 8 | Current token balance |
| bump | u8 | 1 | PDA canonical bump |

**Fetch Example:**
```typescript
const vault = await program.account.vault.fetch(vaultPda);
console.log("Balance:", vault.balance.toNumber());
```

**Derivation** (if PDA):
```typescript
const [vaultPda] = PublicKey.findProgramAddressSync(
  [Buffer.from("vault"), authority.toBuffer()],
  programId
);
```

## Step 5: Event Documentation

For each `#[event]`:

### Event Name

| Field | Type | Description |
|-------|------|-------------|
| authority | Pubkey | Signer who initiated the action |
| amount | u64 | Amount transferred |

**Listener Example:**
```typescript
const listener = program.addEventListener("DepositEvent", (event, slot) => {
  console.log("Deposit:", event.amount.toNumber(), "at slot", slot);
});
```

## Step 6: Error Code Documentation

Table of all error codes:

| Code | Name | Message |
|------|------|---------|
| 6000 | InvalidAmount | Amount must be greater than zero |
| 6001 | Overflow | Arithmetic overflow in balance calculation |
| 6002 | InsufficientBalance | Withdrawal exceeds available balance |

**Error Handling Example:**
```typescript
try {
  await program.methods.deposit(new anchor.BN(0)).accounts({...}).rpc();
} catch (err) {
  if (err.error.errorCode.code === "InvalidAmount") {
    console.error("Deposit amount must be positive");
  }
}
```

## Step 7: TypeScript Type Definitions

Generate TypeScript interfaces matching the IDL:

```typescript
export interface Vault {
  authority: PublicKey;
  balance: BN;
  isActive: boolean;
  bump: number;
}

export interface DepositEvent {
  authority: PublicKey;
  vault: PublicKey;
  amount: BN;
  newBalance: BN;
}
```

## Output Format

Return the complete documentation as a single markdown document (or HTML/JSON depending on `format` input). The document must be self-contained and include all examples, type definitions, and integration guidance.

```json
{
  "documentation": "# Program Name\n\n...",
  "type_definitions": "export interface Vault { ... }"
}
```
