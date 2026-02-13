---
name: deep-audit
version: 1.2.0
description: Multi-pass deep audit with cross-skill correlation and optimized code generation
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
  - name: generate_fix
    type: boolean
    default: true
    description: Whether to produce remediated source code
outputs:
  - name: findings
    type: Finding[]
  - name: score
    type: number
    range: 0-100
  - name: optimized_code
    type: string
    description: Full program source with all findings remediated
  - name: summary
    type: string
chains_from:
  - security-audit
  - best-practices
  - gas-optimization
---

# Deep Audit Skill

You are executing a comprehensive multi-pass audit that combines security analysis, best practices review, and gas optimization into a single correlated assessment. This skill orchestrates the three base skills and adds a cross-correlation layer that identifies compound vulnerabilities, generates a risk-prioritized remediation plan, and produces a corrected version of the program source.

## Context Sources

Before starting any pass, fetch the full pattern database from the Parity repository. All three passes reference these sources.

| Source | Path | Contains |
|--------|------|----------|
| Security Audit Skill | `skills/security-audit/SKILL.md` | Full workflow, checks, detection patterns, and output format for Pass 1 |
| Best Practices Skill | `skills/best-practices/SKILL.md` | Full workflow, convention checks, and output format for Pass 2 |
| Gas Optimization Skill | `skills/gas-optimization/SKILL.md` | Full workflow, CU estimation, rent analysis, and output format for Pass 3 |
| Vulnerability Rules | `programs/parity/src/context_engine.rs` > `VULNERABILITY_RULES` | All pattern IDs, severity levels, descriptions, and detection hints |
| Curated Audit Findings | `programs/parity/src/context_engine.rs` > `CURATED_AUDIT_FINDINGS` | Real vulnerability-fix pairs from OtterSec, Sec3, Neodyme |
| Framework Patterns | `programs/parity/src/context_engine.rs` > `ANCHOR_PATTERNS` | Correct Anchor implementations for comparison |
| Scoring Function | `programs/parity/src/context_engine.rs` > `calculate_risk_score` | Severity penalty weights |

GitHub base URL: `https://github.com/paritydotcx/Skills/blob/main/`

Load all sources before Pass 1. The detection hints in `VULNERABILITY_RULES` guide what to search for. The fix patterns in `CURATED_AUDIT_FINDINGS` inform remediation. The example code in `ANCHOR_PATTERNS` serves as the reference implementation for Pass 6 code generation.

## Pass 1: Security Audit

Execute the full `security-audit` skill workflow against the provided program. Collect all findings with severity, location, pattern, and recommendation. This pass has the highest priority. Security findings always take precedence over best-practices or optimization findings when they conflict.

Run these checks in order:
1. Signer validation on all state-modifying instructions
2. Arithmetic safety on all user-influenced calculations
3. PDA seed determinism and bump validation
4. CPI target program verification
5. Account discriminator and type safety
6. Reinitialization protection on init instructions
7. Account close lamport drain and data zeroing
8. Owner validation on cross-program accounts
9. Reentrancy via post-CPI state mutation
10. Score using penalty model: Critical -25, High -15, Medium -8, Info -3

Store the security findings array and score separately. These feed into Pass 4.

## Pass 2: Best Practices Review

Execute the full `best-practices` skill workflow against the same program. Collect all findings.

Run these checks:
1. InitSpace derive on stored account types
2. Descriptive error messages with `#[msg()]`
3. Event emission on state-changing instructions
4. Typed account wrappers over raw AccountInfo
5. Instruction handler complexity (single-responsibility, <50 lines)
6. Diagnostic logging with `msg!`
7. Named constants over magic numbers

Store the best-practices findings array and score separately.

## Pass 3: Gas Optimization

Execute the full `gas-optimization` skill workflow against the same program. Collect all findings and compute unit estimates.

Run these checks:
1. Account data layout and packing efficiency
2. Unnecessary reallocation patterns
3. Deserialization overhead (zero-copy candidates)
4. Per-instruction compute unit estimation
5. CPI overhead and batching opportunities
6. Rent cost analysis and savings opportunities

Store the optimization findings array and per-instruction CU estimates.

## Pass 4: Cross-Correlation

This is the unique value of deep-audit. Analyze the combined findings from all three passes to identify compound risks that no single skill would flag.

### Correlation rules:

**Rule 1: Security + Optimization conflict**
If a security finding recommends adding a constraint (e.g., `has_one`, `seeds`, `bump`) and an optimization finding recommends reducing constraint count for CU savings, the security finding takes absolute precedence. Remove or note the conflicting optimization finding.

**Rule 2: Missing events on security-critical operations**
If a security finding identifies an instruction that handles funds (deposit, withdraw, transfer) AND a best-practices finding flags the same instruction for missing event emission, escalate the event finding from Info/Medium to High. Missing events on fund-handling instructions means exploits cannot be detected by off-chain monitors.

**Rule 3: Type cosplay + close account interaction**
If a security finding flags missing discriminator checks AND the program has close account logic, note an elevated compound risk: a closed account's stale data could be reinterpreted as a valid account of a different type if discriminators are not enforced.

**Rule 4: Unchecked arithmetic + high compute usage**
If arithmetic operations are unchecked AND the instruction is estimated to consume >70% of the compute budget, note that an overflow could cause unexpected control flow that wastes the remaining budget, potentially causing the transaction to fail at an inconsistent state.

**Rule 5: Reinitialization + missing events**
If reinitialization is possible AND there are no events on the init instruction, an attacker could silently re-initialize protocol state with no audit trail.

Apply all applicable correlation rules. Add compound findings to the combined findings array with their own severity and cross-reference to the source findings.

## Pass 5: Risk-Prioritized Remediation Plan

Order all findings (security + best-practices + optimization + compound) by remediation priority:

1. **Critical security findings** - fix immediately, these represent active exploit risk
2. **Compound findings** - fix next, these amplify individual risks
3. **High security findings** - fix before deployment
4. **High escalated findings** (from correlation) - fix before deployment
5. **Medium findings** (security and best-practices) - fix before production
6. **Optimization findings** - address for cost reduction
7. **Info findings** - address for code quality

For each finding, provide:
- Clear description of the vulnerability or issue
- Exact code location (file, line, instruction)
- Concrete fix with before/after code
- Which other findings this fix addresses (if a single fix resolves multiple findings)

## Pass 6: Generate Remediated Code

If `generate_fix` is true, produce the complete program source with all findings addressed.

**Rules for code generation:**
- Apply all security fixes
- Apply best-practice improvements that do not conflict with security fixes
- Apply optimization changes that do not conflict with security or best-practice fixes
- Add comments on lines that were modified, referencing the finding ID
- Maintain the original code structure and naming conventions
- Do not change the program's external interface (instruction signatures, account struct layouts visible to clients) unless required for security

The output should be a complete, compilable program that addresses all findings.

## Output Format

Return results as structured JSON:

```json
{
  "score": 42,
  "passes": {
    "security": { "score": 35, "finding_count": 4 },
    "best_practices": { "score": 78, "finding_count": 3 },
    "gas_optimization": { "finding_count": 2, "total_cu_savings": 18000 }
  },
  "findings": [
    {
      "severity": "critical",
      "source": "security-audit",
      "pattern": "missing-signer-check",
      "title": "Missing signer validation on withdraw",
      "location": { "file": "lib.rs", "line": 47, "instruction": "withdraw" },
      "description": "...",
      "recommendation": "...",
      "compound_refs": ["missing-event-withdraw"]
    }
  ],
  "compound_findings": [
    {
      "severity": "high",
      "title": "Fund-handling instruction lacks both access control and event emission",
      "source_findings": ["missing-signer-check@L47", "missing-event@L47"],
      "description": "The withdraw instruction has no signer validation and no event emission. An unauthorized withdrawal would leave no on-chain audit trail for detection.",
      "recommendation": "Add Signer constraint and WithdrawEvent emission."
    }
  ],
  "remediation_plan": [
    {
      "priority": 1,
      "finding_refs": ["missing-signer-check@L47"],
      "fix_description": "Add Signer<'info> constraint to authority account in Withdraw struct",
      "before": "pub authority: AccountInfo<'info>,",
      "after": "pub authority: Signer<'info>,"
    }
  ],
  "optimized_code": "// Full remediated program source...",
  "summary": "Deep audit complete. 4 security, 3 best-practice, 2 optimization findings with 1 compound escalation. Overall score: 42. Remediated code generated with all critical and high findings addressed."
}
```

## Score Calculation

The deep-audit score is the security score. Best-practices and optimization scores are reported separately but do not affect the primary score. Security is the only metric that determines whether a program is safe to deploy.
