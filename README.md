# On-chain Payroll

Privacy-preserving payroll system using a dual-chain architecture with **Midnight** and **Cardano**.

> **Note**: This repository serves as supporting documentation for the On-chain Payroll proposal.

## Overview

On-chain Payroll enables organizations to process salary payments on blockchain while keeping sensitive employee data private. The system leverages Midnight's zero-knowledge proof capabilities for confidential computation and Cardano for transparent fund settlement.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        On-chain Payroll System                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────┐         ┌──────────────────────────┐         │
│  │     Midnight Network     │         │    Cardano Network       │         │
│  │    (Privacy Layer)       │         │   (Settlement Layer)     │         │
│  │                          │         │                          │         │
│  │  • Private Employee Data │  ZK     │  • Proof Verification    │         │
│  │  • Salary Computation    │─Proof──▶│  • Fund Release          │         │
│  │  • ZK Proof Generation   │  Hash   │  • Multi-sig Support     │         │
│  │                          │         │                          │         │
│  └──────────────────────────┘         └──────────────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Smart Contracts

This project contains two smart contracts that work together:

### 1. Midnight: PayrollVault (Compact)

**Location**: [`contracts/midnight/PayrollVault.compact`](./contracts/midnight/PayrollVault.compact)

The privacy layer that handles confidential payroll computation:

- **Private Data Storage**: Employee IDs, salary levels, and compensation rules remain encrypted
- **ZK Proof Generation**: Computes salary claims without revealing sensitive information
- **Selective Disclosure**: Aggregate data available for auditors via View Keys

Key circuits:
- `computeSalaryProof()` - Generate ZK proof for salary claim
- `registerEmployee()` - Add employees (admin only)
- `getAuditSummary()` - Aggregate stats for compliance

📖 [Full Documentation](./contracts/midnight/README.md)

### 2. Cardano: Settlement Registry (Aiken)

**Location**: [`contracts/cardano/lib/settlement_registry.ak`](./contracts/cardano/lib/settlement_registry.ak)

The settlement layer that verifies proofs and releases funds:

- **Oracle Attestation**: Verifies ZK proofs via relay oracle signatures
- **Fund Management**: Releases ADA to employees upon verification
- **Security Controls**: Multi-sig support for large withdrawals, budget limits

Key actions:
- `ClaimSalary` - Claim with Oracle attestation
- `UpdateClaimsRoot` - Sync with Midnight state
- `AddBudget` / `WithdrawBudget` - Budget management

📖 [Full Documentation](./contracts/cardano/README.md)

## Architecture Diagrams

Visual documentation is provided via draw.io diagrams:

| Diagram | Description |
|---------|-------------|
| [`architecture-diagram.drawio`](./architecture-diagram.drawio) | System architecture and component relationships |
| [`user-flow-diagram.drawio`](./user-flow-diagram.drawio) | User journey and interaction flows |

> **Tip**: Open `.drawio` files with [draw.io](https://app.diagrams.net/) or VS Code with the Draw.io extension.

## How It Works

```
1. HR registers employees (encrypted)  ─────▶  Midnight PayrollVault
                                                       │
2. HR deposits ADA budget  ────────────────▶  Cardano Settlement Registry
                                                       │
3. Employee initiates claim  ──────────────▶  DApp
                                                       │
4. DApp requests ZK proof  ────────────────▶  Midnight computeSalaryProof()
                                                       │
5. Proof hash generated  ◀─────────────────────────────┘
                                                       │
6. Relay Oracle verifies & signs  ─────────▶  Oracle Attestation
                                                       │
7. Claim submitted  ───────────────────────▶  Cardano Validator
                                                       │
8. Validator verifies & releases  ─────────▶  ADA sent to Employee
```

## Privacy Guarantees

| Data | Visibility |
|------|------------|
| Employee Identity | **Private** (only hash disclosed) |
| Salary Level | **Private** (never disclosed) |
| Exact Salary Amount | **Private** (only in ZK proof) |
| Total Budget | Public (aggregate on ledger) |
| Total Disbursed | Public (aggregate on ledger) |

## Project Structure

```
on-chain-payroll/
├── README.md                          # This file
├── architecture-diagram.drawio        # System architecture diagram
├── user-flow-diagram.drawio           # User flow diagram
├── contracts/
│   ├── README.md                      # Contracts overview
│   ├── midnight/
│   │   ├── PayrollVault.compact       # Privacy layer contract
│   │   ├── managed/                   # Compiled artifacts
│   │   └── README.md
│   └── cardano/
│       ├── lib/
│       │   └── settlement_registry.ak # Settlement layer validator
│       ├── aiken.toml
│       └── README.md
└── LICENSE
```

## Development

### Prerequisites

- **Midnight SDK**: For Compact contract development
- **Aiken**: For Cardano validator development

### Build Contracts

```bash
# Midnight (Compact)
cd contracts/midnight
compact compile PayrollVault.compact managed/payroll

# Cardano (Aiken)
cd contracts/cardano
aiken build
```

### Run Tests

```bash
# Cardano validator tests
cd contracts/cardano
aiken check
```

## References

- [Midnight Documentation](https://docs.midnight.network/)
- [Compact Language Reference](https://docs.midnight.network/develop/reference/compact)
- [Aiken Language](https://aiken-lang.org/)
- [Cardano Developer Portal](https://developers.cardano.org/)

## License

[MIT](./LICENSE)

