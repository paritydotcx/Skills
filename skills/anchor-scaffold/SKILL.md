---
name: anchor-scaffold
version: 2.0.1
description: Generate production-ready Anchor program boilerplate with best practices baked in
author: parity-team
license: MIT
tags:
  - anchor
  - tooling
  - boilerplate
inputs:
  - name: program_name
    type: string
    required: true
    description: Name of the Anchor program to generate (snake_case)
  - name: instructions
    type: string[]
    required: true
    description: List of instruction names to scaffold (e.g. ["initialize", "deposit", "withdraw"])
  - name: accounts
    type: string[]
    required: false
    description: List of custom account type names (e.g. ["Vault", "UserProfile", "Config"])
  - name: include_tests
    type: boolean
    default: true
    description: Whether to generate test scaffolding alongside the program
outputs:
  - name: program_source
    type: string
    description: Complete lib.rs with all instructions, accounts, errors, and events
  - name: test_source
    type: string
    description: TypeScript test file with Bankrun setup and instruction test stubs
  - name: idl_hint
    type: object
    description: Predicted IDL structure for frontend integration
---

# Anchor Program Scaffold Skill

You are generating a production-ready Anchor program from a set of instruction and account specifications. The output must follow every Anchor best practice documented in the Parity context engine. This is not a minimal starter template. The generated code should be deployable after the developer fills in their business logic.

## Context Sources

Before generating code, fetch the following from the Parity repository to ensure generated patterns are correct:

| Source | Path | Contains |
|--------|------|----------|
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct patterns for account init, PDA derivation, CPI, access control, error handling, close, events, and checked math |
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | Common vulnerabilities to proactively avoid in generated code |
| Security Audit Skill | `skills/security-audit/SKILL.md` | The checks the generated code will be evaluated against |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

The generated code must pass all checks in the security-audit skill with a score of 95 or higher.

## Step 1: Program Module Structure

Generate the program module with proper Anchor attributes:

```rust
use anchor_lang::prelude::*;

declare_id!("YOUR_PROGRAM_ID_HERE");

#[program]
pub mod program_name {
    use super::*;
    
    // instruction handlers go here
}
```

Every generated file must include:
- `declare_id!` with a placeholder
- `use anchor_lang::prelude::*;`
- Additional imports as needed (`anchor_spl::token`, `anchor_spl::associated_token`)

## Step 2: Account Definitions

For each account type specified in the `accounts` input, generate a struct with:

1. `#[account]` attribute
2. `#[derive(InitSpace)]` for automatic space calculation
3. All fields with explicit type annotations
4. A `bump` field (`u8`) if the account is PDA-derived
5. An `authority` field (`Pubkey`) if the account is owned/controlled

```rust
#[account]
#[derive(InitSpace)]
pub struct Vault {
    pub authority: Pubkey,    // 32
    pub balance: u64,         // 8
    pub is_active: bool,      // 1
    pub bump: u8,             // 1
}
```

Rules:
- Use fixed-size types. No `String` fields. Use `[u8; N]` for text.
- No `Vec<T>` fields. Use fixed arrays `[T; N]` with a `count` field.
- Order fields largest-to-smallest for alignment efficiency.

## Step 3: Instruction Accounts Structs

For each instruction, generate a `#[derive(Accounts)]` struct with:

1. Proper signer constraint on authority accounts (`Signer<'info>`)
2. PDA seeds and bump on derived accounts
3. `init` constraint with `payer` and `space = 8 + T::INIT_SPACE` for initialization instructions
4. `has_one = authority` on accounts that require ownership validation
5. `mut` only on accounts that are actually modified
6. `Program<'info, System>` and `Program<'info, Token>` for program accounts (never raw `AccountInfo`)
7. `close = destination` on close instructions

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Vault::INIT_SPACE,
        seeds = [b"vault", authority.key().as_ref()],
        bump,
    )]
    pub vault: Account<'info, Vault>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    
    pub system_program: Program<'info, System>,
}
```

For every account in the struct, verify:
- If it holds funds or state: `has_one = authority` or equivalent
- If it is a PDA: `seeds` and `bump` constraints present
- If it is a token account: use `Account<'info, TokenAccount>` not `AccountInfo`

## Step 4: Instruction Handlers

For each instruction, generate a handler function with:

1. Input validation using `require!` macro
2. State updates using checked arithmetic (`checked_add`, `checked_sub`)
3. Event emission at the end of every state-changing handler
4. `msg!` logging at function entry with key parameters
5. Explicit `Ok(())` return

```rust
pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
    msg!("deposit: amount={}", amount);
    require!(amount > 0, ErrorCode::InvalidAmount);
    
    let vault = &mut ctx.accounts.vault;
    vault.balance = vault.balance
        .checked_add(amount)
        .ok_or(ErrorCode::Overflow)?;
    
    emit!(DepositEvent {
        authority: ctx.accounts.authority.key(),
        vault: vault.key(),
        amount,
        new_balance: vault.balance,
    });
    
    Ok(())
}
```

## Step 5: Error Definitions

Generate a custom error enum with descriptive messages for every validation:

```rust
#[error_code]
pub enum ErrorCode {
    #[msg("Amount must be greater than zero")]
    InvalidAmount,
    #[msg("Arithmetic overflow in balance calculation")]
    Overflow,
    #[msg("Insufficient balance for withdrawal")]
    InsufficientBalance,
    #[msg("Account is not active")]
    AccountInactive,
}
```

Rules:
- One error variant per `require!` check
- Messages must describe the condition, not just name the error
- Include the field or value that failed validation when possible

## Step 6: Event Definitions

Generate an event struct for every state-changing instruction:

```rust
#[event]
pub struct DepositEvent {
    pub authority: Pubkey,
    pub vault: Pubkey,
    pub amount: u64,
    pub new_balance: u64,
}
```

Events must include:
- The signer/authority who initiated the action
- The account(s) affected
- The values that changed
- The resulting state (new balance, new status)

## Step 7: Test Scaffolding

If `include_tests` is true, generate a TypeScript test file with:

1. Bankrun test setup (`solana-bankrun` or `anchor` test framework)
2. Account keypair generation
3. PDA derivation matching the program's seeds
4. One test per instruction with:
   - Account setup
   - Instruction call
   - State assertion (fetch account, verify fields)
5. One negative test per instruction (expected failure case)

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { ProgramName } from "../target/types/program_name";
import { expect } from "chai";

describe("program_name", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
  const program = anchor.workspace.ProgramName as Program<ProgramName>;

  it("initializes vault", async () => {
    const [vaultPda] = anchor.web3.PublicKey.findProgramAddressSync(
      [Buffer.from("vault"), provider.wallet.publicKey.toBuffer()],
      program.programId
    );
    
    await program.methods
      .initialize()
      .accounts({ vault: vaultPda })
      .rpc();
    
    const vault = await program.account.vault.fetch(vaultPda);
    expect(vault.authority.toString()).to.equal(provider.wallet.publicKey.toString());
    expect(vault.balance.toNumber()).to.equal(0);
  });
});
```

## Output Format

Return the generated code as structured JSON:

```json
{
  "program_source": "// Full lib.rs content...",
  "test_source": "// Full test file content...",
  "idl_hint": {
    "instructions": ["initialize", "deposit", "withdraw"],
    "accounts": ["Vault"],
    "events": ["DepositEvent", "WithdrawEvent"],
    "errors": ["InvalidAmount", "Overflow", "InsufficientBalance"]
  }
}
```
