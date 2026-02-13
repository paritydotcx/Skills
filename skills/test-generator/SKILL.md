---
name: test-generator
version: 1.1.0
description: Automatically generate comprehensive test suites for Anchor programs with Bankrun and proper account setup
author: parity-team
license: MIT
tags:
  - testing
  - anchor
  - tooling
inputs:
  - name: program
    type: file
    required: true
    description: Path to the Solana program source file (.rs)
  - name: framework
    type: string
    default: anchor
  - name: test_framework
    type: string
    default: bankrun
    description: Test framework to use (bankrun | anchor-test | both)
  - name: include_fuzz
    type: boolean
    default: false
    description: Whether to generate fuzz test scaffolds
outputs:
  - name: unit_tests
    type: string
    description: Complete unit test file
  - name: integration_tests
    type: string
    description: Complete integration test file
  - name: fuzz_tests
    type: string
    description: Fuzz test scaffold (if include_fuzz is true)
  - name: test_coverage
    type: object
    description: Coverage map showing which instructions and code paths are tested
---

# Solana Test Generator Skill

You are generating a comprehensive test suite for a Solana Anchor program. The tests must cover every instruction handler with positive cases, negative cases (expected failures), edge cases, and access control tests. The goal is not just code coverage but security coverage: every vulnerability pattern from the security-audit skill should have a corresponding test that verifies the program is protected.

## Context Sources

| Source | Path | Contains |
|--------|------|----------|
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | Patterns to generate negative tests for |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct patterns to verify in positive tests |
| Security Audit Skill | `skills/security-audit/SKILL.md` | Security checks to verify via tests |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

For each vulnerability pattern in `VULNERABILITY_RULES`, generate a test that proves the program either has the vulnerability (test should fail with expected error) or is protected against it.

## Step 1: Analyze Program Interface

Parse the program and extract:

1. All instruction handlers with their parameters and types
2. All `#[derive(Accounts)]` structs with their constraints
3. All account types with their fields
4. All PDA seeds and derivation patterns
5. All error codes
6. All events

Build a test plan that maps each instruction to:
- 1 positive test (happy path)
- 1 negative test per `require!` or constraint
- 1 access control test (wrong signer)
- 1 edge case test (boundary values, zero amounts, max values)

## Step 2: Test Setup and Fixtures

Generate a shared test setup module:

```typescript
import { startAnchor } from "solana-bankrun";
import { BankrunProvider } from "anchor-bankrun";
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Keypair, PublicKey, SystemProgram } from "@solana/web3.js";

async function setupTest() {
  const context = await startAnchor("", [], []);
  const provider = new BankrunProvider(context);
  anchor.setProvider(provider);
  
  const program = anchor.workspace.ProgramName as Program<ProgramName>;
  const authority = provider.wallet;
  const unauthorizedUser = Keypair.generate();
  
  // Airdrop to unauthorized user for negative tests
  // ... bankrun account funding
  
  return { context, provider, program, authority, unauthorizedUser };
}
```

Generate PDA derivation helpers that mirror the program's seed patterns:

```typescript
function findVaultPda(authority: PublicKey, programId: PublicKey): [PublicKey, number] {
  return PublicKey.findProgramAddressSync(
    [Buffer.from("vault"), authority.toBuffer()],
    programId
  );
}
```

## Step 3: Positive Tests (Happy Path)

For each instruction, generate a test that:

1. Sets up all required accounts in the correct state
2. Calls the instruction with valid parameters
3. Fetches the modified accounts after the transaction
4. Asserts every field matches the expected state

```typescript
it("deposits tokens successfully", async () => {
  const { program, authority } = await setupTest();
  const [vaultPda] = findVaultPda(authority.publicKey, program.programId);
  
  // Initialize first
  await program.methods.initialize().accounts({ vault: vaultPda }).rpc();
  
  // Deposit
  const depositAmount = new anchor.BN(1_000_000);
  await program.methods
    .deposit(depositAmount)
    .accounts({ vault: vaultPda })
    .rpc();
  
  // Verify state
  const vault = await program.account.vault.fetch(vaultPda);
  expect(vault.balance.toNumber()).to.equal(1_000_000);
});
```

## Step 4: Access Control Tests

For every instruction that requires a signer, generate a test where the wrong signer calls the instruction:

```typescript
it("rejects deposit from unauthorized user", async () => {
  const { program, unauthorizedUser } = await setupTest();
  const [vaultPda] = findVaultPda(unauthorizedUser.publicKey, program.programId);
  
  try {
    await program.methods
      .deposit(new anchor.BN(1_000_000))
      .accounts({ vault: vaultPda, authority: unauthorizedUser.publicKey })
      .signers([unauthorizedUser])
      .rpc();
    expect.fail("Should have thrown");
  } catch (err) {
    expect(err.error.errorCode.code).to.equal("ConstraintHasOne");
  }
});
```

Generate one access control test per instruction that has a `Signer` or `has_one` constraint.

## Step 5: Input Validation Tests

For every `require!` check in the program, generate a test that triggers it:

```typescript
it("rejects zero deposit amount", async () => {
  const { program } = await setupTest();
  const [vaultPda] = findVaultPda(/*...*/);
  
  try {
    await program.methods
      .deposit(new anchor.BN(0))
      .accounts({ vault: vaultPda })
      .rpc();
    expect.fail("Should have thrown");
  } catch (err) {
    expect(err.error.errorCode.code).to.equal("InvalidAmount");
  }
});
```

Tests to generate per instruction:
- Zero amount (if amount parameter exists)
- Maximum u64 value (overflow test)
- Insufficient balance (if withdrawal)
- Invalid account state (if status flags exist)

## Step 6: PDA and Account Tests

Generate tests that verify PDA derivation:

```typescript
it("derives vault PDA correctly", async () => {
  const { program, authority } = await setupTest();
  const [expectedPda, bump] = findVaultPda(authority.publicKey, program.programId);
  
  await program.methods.initialize().accounts({ vault: expectedPda }).rpc();
  
  const vault = await program.account.vault.fetch(expectedPda);
  expect(vault.bump).to.equal(bump);
});
```

Generate tests that verify account reinitialization is blocked:

```typescript
it("prevents vault reinitialization", async () => {
  const { program } = await setupTest();
  const [vaultPda] = findVaultPda(/*...*/);
  
  await program.methods.initialize().accounts({ vault: vaultPda }).rpc();
  
  try {
    await program.methods.initialize().accounts({ vault: vaultPda }).rpc();
    expect.fail("Should have thrown");
  } catch (err) {
    // Account already initialized
    expect(err).to.exist;
  }
});
```

## Step 7: Event Tests

If the program emits events, generate tests that capture and verify them:

```typescript
it("emits DepositEvent on deposit", async () => {
  const { program } = await setupTest();
  const [vaultPda] = findVaultPda(/*...*/);
  
  const listener = program.addEventListener("DepositEvent", (event) => {
    expect(event.amount.toNumber()).to.equal(1_000_000);
    expect(event.newBalance.toNumber()).to.equal(1_000_000);
  });
  
  await program.methods
    .deposit(new anchor.BN(1_000_000))
    .accounts({ vault: vaultPda })
    .rpc();
  
  program.removeEventListener(listener);
});
```

## Step 8: Fuzz Test Scaffolds (Optional)

If `include_fuzz` is true, generate fuzz test scaffolds using `trident` or equivalent:

```rust
// Fuzz test scaffold for deposit instruction
fuzz_target!(|data: FuzzData| {
    let amount = data.amount;
    let authority = data.authority;
    
    // Setup: initialize program state
    // Execute: call deposit with fuzzed amount
    // Verify: balance invariants hold
    //   - balance >= 0
    //   - balance <= MAX_SUPPLY
    //   - balance_after == balance_before + amount (no overflow)
});
```

Generate one fuzz target per instruction with:
- Randomized input parameters
- Invariant checks after execution
- Boundary value focus (0, 1, u64::MAX -1, u64::MAX)

## Output Format

```json
{
  "unit_tests": "// Complete test file with all unit tests...",
  "integration_tests": "// Integration test file with multi-instruction flows...",
  "fuzz_tests": "// Fuzz test scaffolds (if requested)...",
  "test_coverage": {
    "instructions": {
      "initialize": { "positive": 1, "negative": 2, "access_control": 1, "edge_case": 1 },
      "deposit": { "positive": 1, "negative": 3, "access_control": 1, "edge_case": 2 },
      "withdraw": { "positive": 1, "negative": 2, "access_control": 1, "edge_case": 2 }
    },
    "total_tests": 18,
    "vulnerability_patterns_covered": ["missing-signer-check", "unchecked-arithmetic", "reinitialization-attack"]
  }
}
```
