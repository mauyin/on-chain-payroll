# On-chain Payroll: Smart Contracts

This directory contains the smart contract implementations for the On-chain Payroll dual-chain architecture.

**Status**: ✅ Both contracts compiled successfully and ready for testnet deployment

**Navigation**:
- 📖 [This Document](#) - Architecture and development guide
- 🌙 [Midnight Contract](./midnight/README.md) - Privacy layer (Compact)
- 🔗 [Cardano Contract](./cardano/README.md) - Settlement layer (Aiken)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        On-chain Payroll System                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────┐         ┌──────────────────────────┐         │
│  │     Midnight Network     │         │    Cardano Network       │         │
│  │    (Privacy Layer)       │         │   (Settlement Layer)     │         │
│  │                          │         │                          │         │
│  │  ┌────────────────────┐  │         │  ┌────────────────────┐  │         │
│  │  │  PayrollVault.     │  │  ZK     │  │  settlement_       │  │         │
│  │  │  compact           │──┼─Proof──▶│  │  registry.ak       │  │         │
│  │  │                    │  │  Hash   │  │                    │  │         │
│  │  │  • Private Data    │  │         │  │  • Proof Verify    │  │         │
│  │  │  • ZK Computation  │  │         │  │  • Fund Release    │  │         │
│  │  │  • Proof Gen       │  │         │  │  • Multi-sig       │  │         │
│  │  └────────────────────┘  │         │  └────────────────────┘  │         │
│  │                          │         │            │             │         │
│  └──────────────────────────┘         │            ▼             │         │
│                                       │     ┌──────────────┐     │         │
│                                       │     │   Employee   │     │         │
│                                       │     │   Wallet     │     │         │
│                                       │     │   (ADA)      │     │         │
│                                       │     └──────────────┘     │         │
│                                       └──────────────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
contracts/
├── README.md                    # This file
├── midnight/
│   ├── PayrollVault.compact    # Privacy-preserving payroll contract
│   └── README.md               # Midnight contract documentation
└── cardano/
    ├── lib/
    │   └── settlement_registry.ak  # Settlement validator
    ├── aiken.toml              # Aiken project configuration
    └── README.md               # Cardano contract documentation
```

## Contract Summary

### Midnight: PayrollVault.compact

**Purpose**: Privacy-preserving payroll computation with ZK proof generation

| Feature | Description |
|---------|-------------|
| **Language** | Compact (Midnight DSL) |
| **Privacy** | Employee data, salary rules, amounts all encrypted |
| **Output** | ZK proof hash for cross-chain verification |
| **State** | Employee count, budget, disbursed (public aggregates) |

**Key Circuits**:
- `computeSalaryProof()` - Generate ZK proof for salary claim
- `registerEmployee()` - Add employee (admin)
- `getAuditSummary()` - Aggregate stats for auditors

### Cardano: settlement_registry.ak

**Purpose**: Verify proof attestations and release funds

| Feature | Description |
|---------|-------------|
| **Language** | Aiken (Cardano) |
| **Verification** | Oracle attestation over ZK proofs |
| **Settlement** | Direct ADA transfer to employees |
| **Security** | Multi-sig, budget controls, replay protection |

**Key Actions**:
- `ClaimSalary` - Claim with Oracle attestation
- `UpdateClaimsRoot` - Sync with Midnight state
- `AddBudget` / `WithdrawBudget` - Budget management

## Data Flow

```
1. HR defines salary rules (encrypted) ─────▶ Midnight PayrollVault
                                                      │
2. HR deposits ADA budget ──────────────────▶ Cardano Settlement Registry
                                                      │
3. Employee triggers claim ─────────────────▶ DApp
                                                      │
4. DApp requests proof ─────────────────────▶ Midnight computeSalaryProof()
                                                      │
5. Midnight returns proof_hash ◀────────────────────┘
                                                      │
6. Relay Oracle verifies & signs ───────────▶ oracle_attestation
                                                      │
7. Transaction submitted ───────────────────▶ Cardano validator
                                                      │
8. Validator verifies:                                │
   • Oracle attestation valid                         │
   • Amount within budget                             │
   • Datum updated correctly                          │
                                                      │
9. ADA released ────────────────────────────▶ Employee Wallet
```

## Development Setup

### Midnight (Compact)

```bash
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/midnightntwrk/compact/releases/download/compact-v0.2.0/compact-installer.sh | sh

# Compile contract
compact update
compact compile PayrollVault.compact managed/payroll
```

### Cardano (Aiken)

```bash
# Install Aiken
curl -sSfL https://install.aiken-lang.org | bash

# Build contract
cd contracts/cardano
aiken build

# Run tests
aiken check

# Generate Plutus blueprint
aiken blueprint convert > plutus.json
```

## Security Model

### Trust Assumptions

| Component | Trust Level | Notes |
|-----------|-------------|-------|
| Midnight | Cryptographic | ZK proofs are mathematically sound |
| Relay Oracle | **Trust Required** | Until native bridge available |
| Cardano | Cryptographic | On-chain verification |
| Admin Keys | Operational | Secure key management required |

### Mitigations

1. **Oracle Risk**: Multi-Oracle threshold signatures (roadmap)
2. **Key Compromise**: Multi-sig for large withdrawals
3. **Budget Overdraw**: On-chain balance checks
4. **Replay Attacks**: Round counter + UTxO model

## Testing Strategy

### Unit Tests
- Individual circuit/validator logic
- Datum/redeemer serialization
- Edge cases and error conditions

### Integration Tests
- Cross-chain proof flow
- Full claim lifecycle
- Budget management scenarios

### Testnet Deployment
- Midnight DevNet for privacy layer
- Cardano Pre-production for settlement
- End-to-end flow validation

## Deployment Checklist

- [ ] Compile Compact contract without errors
- [ ] Build Aiken validator without errors
- [ ] Run all unit tests
- [ ] Deploy to testnet
- [ ] Verify cross-chain flow
- [ ] Security review
- [ ] Mainnet deployment (Cardano)
- [ ] Monitor and validate

## References

- [Midnight Documentation](https://docs.midnight.network/)
- [Midnight Compact Language](https://docs.midnight.network/develop/compact/)
- [Aiken Language](https://aiken-lang.org/)
- [Aiken Standard Library](https://aiken-lang.github.io/stdlib/)
- [Cardano Developer Portal](https://developers.cardano.org/)
- [OpenZeppelin Compact Contracts](https://github.com/OpenZeppelin/compact-contracts) - Reference examples

