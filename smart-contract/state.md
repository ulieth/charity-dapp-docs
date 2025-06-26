# State Management

The Anchor program implements a sophisticated state management system that tracks charity information, donations, and program state across the Solana blockchain. This section covers account structures, lifecycle management, and state relationships.

## Understanding Blockchain State

Unlike traditional databases, blockchain state management has unique characteristics that shape how we design data structures:

**Immutability**: Once written to the blockchain, data cannot be modified arbitrarily. Changes must be explicit and go through program instructions.

**Cost Awareness**: Every byte stored on-chain incurs ongoing rent costs. Efficient data structures directly impact user costs.

**Concurrent Safety**: Multiple users may interact with the program simultaneously. State transitions must be atomic and safe.

**Transparency**: All state changes are publicly visible and auditable, making data integrity paramount.

## Design Philosophy

Our state management follows several key principles:

**Separation of Concerns**: Financial data (SOL) is stored separately from metadata to minimize complexity and risk.

**Audit Trail**: Critical events are timestamped and preserved, creating an immutable history of operations.

**Predictable Costs**: Account sizes are calculated upfront to ensure rent-exemption and predictable user costs.

**Security First**: State transitions are validated at every step to prevent invalid or malicious modifications.

## Account Overview

The program manages three primary account types, each serving a specific purpose in the charity ecosystem:

1. **Charity Account** - Core charity information and metadata
2. **Donation Account** - Individual donation transaction records
3. **Vault Account** - SOL storage for charity funds

### Account Relationship Model

```
┌─────────────────┐    owns    ┌─────────────────┐
│   Authority     │─────────────│ Charity Account │
│   (Wallet)      │             │   (Metadata)    │
└─────────────────┘             └─────────────────┘
                                        │
                                   derives│
                                        ▼
                                ┌─────────────────┐
                                │  Vault Account  │
                                │   (SOL Funds)   │
                                └─────────────────┘
                                        ▲
                                receives│
                                        │
┌─────────────────┐   creates   ┌─────────────────┐
│    Donors       │─────────────│ Donation Records│
│   (Wallets)     │             │   (History)     │
└─────────────────┘             └─────────────────┘
```

This separation ensures that:
- Metadata and funds are isolated for security
- Donation history is preserved immutably
- Each account type can be optimized for its specific purpose

## Design Rationale

**Why Separate Metadata from Funds?**
- Reduces complexity when updating charity information
- Minimizes risk to funds during metadata operations
- Allows for more efficient account space usage
- Enables atomic operations on either metadata or funds independently

**Field Selection Rationale:**
Each field in the charity account was chosen to serve specific operational or security needs while minimizing storage costs.

### Space Calculation

Understanding space calculation is crucial for rent optimization and cost predictability.

```rust
impl Space for Charity {
    const INIT_SPACE: usize =
        32 +                   // authority
        4 + 50 +               // name (4 bytes length + max 50 chars)
        4 + 500 +              // description (4 bytes length + max 500 chars)
        8 +                    // donations_in_lamports
        4 +                    // donation_count
        1 +                    // paused
        8 +                    // created_at
        8 +                    // updated_at
        1 + 8 +                // deleted_at (Option<i64>)
        1 + 8 +                // withdrawn_at (Option<i64>)
        1;                     // vault_bump
}
```

The charity account serves as the central metadata store for each charity organization. It contains all the information needed to identify, manage, and track the charity without storing actual funds.
