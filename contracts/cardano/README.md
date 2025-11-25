# Cardano Aiken Validator: Settlement Registry

A Cardano smart contract written in **Aiken** for verifying ZK proof attestations and releasing payroll funds.

## Overview

This validator serves as the **Settlement Layer** in our dual-chain architecture. It receives proof attestations from the Relay Oracle (originating from Midnight) and releases locked ADA to employees upon successful verification.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Cardano Settlement Registry                 │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │   Oracle    │  │   Proof     │  │    Fund         │ │
│  │ Attestation │─▶│ Verification│─▶│    Release      │ │
│  │  (Input)    │  │             │  │                 │ │
│  └─────────────┘  └─────────────┘  └────────┬────────┘ │
│                                              │          │
│                                              ▼          │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 Employee Wallet                  │   │
│  │                 (ADA Received)                   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Key Features

### 1. Proof Attestation Verification
- Validates Oracle signatures over ZK proofs
- Ensures proof matches current payroll round
- Links Midnight computation to Cardano settlement

### 2. Fund Release Logic
- Releases exact claimed amount to recipient
- Updates disbursement tracking in datum
- Prevents overdraw with budget checks

### 3. Multi-Signature Support
- Optional co-signer for large withdrawals
- Configurable threshold amount
- Additional security for high-value settlements

### 4. Replay Protection
- Uses Cardano's UTxO model inherently
- Round-based proof validation
- One-time claim per proof hash

## Contract Interface

### Datum Structure
```aiken
pub type SettlementDatum {
  employer: VerificationKeyHash,      // Admin key
  oracle: VerificationKeyHash,        // Oracle key
  co_signer: Option<VerificationKeyHash>, // Optional co-signer
  co_sign_threshold: Int,             // Threshold for co-sign
  current_round: Int,                 // Must match Midnight
  claims_root: ByteArray,             // Merkle root from Midnight
  total_budget: Int,                  // Locked ADA (lovelace)
  total_disbursed: Int,               // Paid out (lovelace)
}
```

### Redeemer Actions
| Action | Description | Authorization |
|--------|-------------|---------------|
| `ClaimSalary` | Claim with Oracle attestation | Oracle signature |
| `UpdateClaimsRoot` | Sync Merkle root from Midnight | Employer |
| `WithdrawBudget` | Withdraw remaining funds | Employer |
| `AddBudget` | Add more funds | Employer |
| `EmergencyPause` | Emergency fund recovery | Employer |

### ClaimSalary Redeemer
```aiken
ClaimSalary {
  proof_hash: ByteArray,      // From Midnight computeSalaryProof()
  amount: Int,                // Claim amount in lovelace
  recipient: Address,         // Employee's Cardano address
  oracle_attestation: ByteArray, // Oracle's signature
}
```

## How It Works

### Claim Flow

1. **Employee initiates claim** via DApp
2. **DApp retrieves** proof_hash from Midnight
3. **Relay Oracle** verifies and signs the proof
4. **Transaction built** with:
   - Input: Settlement UTxO
   - Redeemer: `ClaimSalary { proof_hash, amount, recipient, attestation }`
   - Output 1: Updated Settlement UTxO (budget reduced)
   - Output 2: Payment to employee
5. **Validator checks**:
   - Oracle attestation valid
   - Amount within budget
   - Co-signer present (if required)
   - Datum updated correctly
   - Recipient receives payment
6. **Transaction submitted** to Cardano

### Validation Logic

```
ClaimSalary Validation:
├── 1. Verify Oracle attestation
│   └── Hash(proof_hash || amount || round || oracle_vkh)
├── 2. Check amount bounds
│   └── 0 < amount <= (total_budget - total_disbursed)
├── 3. Check co-signer (if amount >= threshold)
│   └── Transaction signed by co_signer
├── 4. Verify continuing output
│   ├── Datum.total_disbursed += amount
│   └── Value reduced by amount
└── 5. Verify recipient paid
    └── Output to recipient >= amount
```

## Development

### Prerequisites
- [Aiken](https://aiken-lang.org/installation) v1.0+
- Cardano node or Blockfrost API

### Build
```bash
cd contracts/cardano
aiken build
```

### Test
```bash
aiken check
```

### Deploy
```bash
# Generate blueprint
aiken blueprint convert > plutus.json

# Use cardano-cli or Lucid to deploy
```

## Integration with Midnight

The Settlement Registry expects proof data from Midnight:

1. **Midnight** generates `proof_hash` via `computeSalaryProof()`
2. **Relay Oracle** receives and verifies the proof
3. **Oracle creates attestation**: `hash(proof_hash || amount || round || oracle_vkh)`
4. **Cardano validator** verifies attestation matches

### Data Flow
```
Midnight                    Relay Oracle               Cardano
   │                             │                        │
   │  proof_hash, amount         │                        │
   │─────────────────────────────▶                        │
   │                             │  verify & sign         │
   │                             │──────────────────────▶ │
   │                             │  oracle_attestation    │
   │                             │                        │
   │                             │                     Validate
   │                             │                     & Release
   │                             │                        │
   │                             │         ADA            │
   │◀──────────────────────────────────────────────────── │
   │                        to Employee                   │
```

## Security Considerations

1. **Oracle Trust**: The Oracle is currently a trust assumption. Future versions will use threshold signatures or native ZK verification when Plutus supports it.

2. **Key Management**: 
   - Employer key controls budget and updates
   - Oracle key only for attestations
   - Co-signer for high-value protection

3. **Round Synchronization**: Ensure `current_round` matches Midnight before processing claims

4. **Budget Monitoring**: Monitor `total_disbursed` to prevent unexpected overdraw

## Gas/Fee Estimation

| Operation | Estimated Fee |
|-----------|---------------|
| ClaimSalary | ~0.3-0.5 ADA |
| UpdateClaimsRoot | ~0.2-0.3 ADA |
| AddBudget | ~0.2-0.3 ADA |
| WithdrawBudget | ~0.2-0.3 ADA |

*Fees depend on transaction size and network conditions*

## Testing

### Unit Tests (Included)
```aiken
test datum_creation()      // Verify datum structure
test is_signed_by_works()  // Signature verification
test verify_amount_bounds() // Budget constraint checks
```

### Integration Tests
See `/tests/integration/` for full transaction simulation tests.

## References

- [Aiken Language Tour](https://aiken-lang.org/language-tour)
- [Cardano Developer Portal](https://developers.cardano.org/)
- [Aiken Standard Library](https://aiken-lang.github.io/stdlib/)
- [JPG Store Contracts](https://github.com/jpg-store/contracts-v3) (reference implementation)

