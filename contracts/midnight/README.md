# Midnight Compact Contract: PayrollVault

A privacy-preserving payroll smart contract written in **Compact** for the **Midnight** blockchain.

## Overview

This contract demonstrates how Midnight's zero-knowledge proof capabilities enable confidential salary computation while maintaining verifiability. It serves as the **Privacy Layer** in our dual-chain architecture.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Midnight Privacy Vault                  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │  Employee   │  │   Salary    │  │   ZK Proof      │ │
│  │  Data Store │─▶│   Rules     │─▶│   Generator     │ │
│  │ (Encrypted) │  │  Engine     │  │                 │ │
│  └─────────────┘  └─────────────┘  └────────┬────────┘ │
│                                              │          │
│                                              ▼          │
│                                    ┌─────────────────┐ │
│                                    │  Proof Hash     │ │
│                                    │  (to Cardano)   │ │
│                                    └─────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## Key Features

### 1. Private State Management
- Employee data stored encrypted using Midnight's privacy model
- Salary rules remain confidential to the organization
- Only commitment hashes are publicly visible

### 2. ZK Proof Generation
- `computeSalaryProof()` generates proof without revealing:
  - Employee identity
  - Compensation level
  - Exact salary amount
- Proof can be verified on Cardano

### 3. Selective Disclosure
- View Keys enable authorized access for auditors
- `getAuditSummary()` provides aggregate data without individual details

### 4. Security Controls
- Admin authorization via derived public keys
- Round counter for replay protection
- Pause/resume functionality

## Contract Interface

### State Variables
```compact
export ledger admin_authority: Bytes<32>;   // Admin public key
export ledger contract_state: ContractState; // ACTIVE/PAUSED
export ledger round: Counter;                // Payroll cycle counter
export ledger employee_count: Uint<64>;      // Total employees
export ledger total_budget: Uint<64>;        // Locked funds
export ledger total_disbursed: Uint<64>;     // Paid out
export ledger rules_hash: Bytes<32>;         // Salary rules hash
export ledger claims_root: Bytes<32>;        // Claims Merkle root
```

### Public Circuits
| Circuit | Description |
|---------|-------------|
| `registerEmployee(commitment)` | Register new employee (admin only) |
| `updateSalaryRules(hash)` | Update compensation rules (admin only) |
| `depositBudget(amount)` | Add funds to vault (admin only) |
| `computeSalaryProof()` | **Core**: Generate ZK proof for salary claim |
| `markSettled(hash, amount)` | Mark claim as paid (after Cardano confirms) |
| `completePayrollCycle()` | Increment round, prevent replays |
| `getAuditSummary()` | Get aggregate stats for auditors |
| `pauseContract()` / `resumeContract()` | Emergency controls |
| `transferAdmin(newPk)` | Transfer admin authority |

### Witness Functions (Private Inputs)
```compact
witness getAdminSecretKey(): Bytes<32>;
witness getEmployeeData(): Vector<5, Bytes<32>>;
witness getSalaryRules(): Vector<10, Uint<64>>;
witness getClaimNonce(): Bytes<32>;
```

## How It Works

### Salary Computation Flow

1. **HR triggers payroll** via DApp
2. **DApp calls** `computeSalaryProof()`
3. **Witness functions** provide private inputs:
   - Employee ID, level, base salary, bonus rate
4. **Contract computes** salary using private rules
5. **ZK Proof generated** as `proof_hash`
6. **Proof hash sent** to Cardano Settlement Registry
7. **Cardano verifies** Oracle attestation and releases ADA

### Privacy Guarantees

| Data | Visibility |
|------|------------|
| Employee ID | Private (only hash disclosed) |
| Salary Level | Private (never disclosed) |
| Exact Salary | Private (only in proof) |
| Total Budget | Public (on ledger) |
| Total Disbursed | Public (on ledger) |
| Employee Count | Public (aggregate only) |

## Development

### Prerequisites
- Midnight SDK (`compactc` compiler)
- Node.js 18+

### Compile
```bash
compactc compile contracts/midnight/PayrollVault.compact managed/payroll
```

### Test
```bash
npm run test:compact
```

## Integration with Cardano

The `proof_hash` output from `computeSalaryProof()` is:
1. Attested by the Relay Oracle
2. Submitted to Cardano's Settlement Registry
3. Verified on-chain
4. Triggers ADA release to employee wallet

See `../cardano/README.md` for settlement details.

## Security Considerations

1. **Admin Key Security**: The admin secret key should be stored securely in a hardware wallet or HSM
2. **Round Management**: Never reuse rounds; always call `completePayrollCycle()` after each cycle
3. **Budget Monitoring**: Monitor `total_disbursed` vs `total_budget` to prevent overdraw
4. **Oracle Trust**: The Relay Oracle is a trust assumption until native bridge is available

## References

- [Midnight Developer Docs](https://docs.midnight.network/)
- [Compact Language Reference](https://docs.midnight.network/develop/reference/compact)
- [OpenZeppelin Compact Contracts](https://github.com/openzeppelin/compact-contracts)

