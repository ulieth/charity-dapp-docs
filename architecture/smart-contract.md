# Smart Contract Architecture

The Solana Charity dApp's smart contract is built using the Anchor framework, providing a secure, efficient, and maintainable foundation for charity management and donations.

## Core Design Principles

### Security-First Architecture
- **Access Control**: Role-based permissions for charity management
- **Input Validation**: Comprehensive validation of all parameters  
- **Error Handling**: Detailed error messages and proper error propagation
- **Rent Protection**: Ensures accounts maintain rent-exempt balances

### Gas Efficiency
- **Optimized Account Structures**: Minimal data storage for cost efficiency
- **Batch Operations**: Multiple state updates in single transactions
- **Efficient PDAs**: Streamlined Program Derived Address derivation

### Transparency & Auditability
- **Event Logging**: All operations emit events for off-chain indexing
- **Immutable History**: Transaction history preserved on-chain
- **Open Source**: Fully auditable codebase

## Program Structure

The smart contract consists of six core instructions:

1. **create_charity**: Initialize new charity with vault
2. **donate_sol**: Transfer SOL from donor to charity vault
3. **withdraw_donations**: Transfer funds from vault to authority
4. **update_charity**: Modify charity description
5. **pause_donations**: Toggle donation acceptance
6. **delete_charity**: Remove charity and withdraw remaining funds

## Account Architecture

### Charity Account
```rust
pub struct Charity {
    pub authority: Pubkey,              // 32 bytes - Charity owner
    pub name: String,                   // Variable - Charity name
    pub description: String,            // Variable - Description
    pub donations_in_lamports: u64,     // 8 bytes - Total donations
    pub donation_count: u32,            // 4 bytes - Number of donations
    pub paused: bool,                   // 1 byte - Donation status
    pub created_at: i64,                // 8 bytes - Creation timestamp
    pub updated_at: i64,                // 8 bytes - Last update
    pub vault_bump: u8,                 // 1 byte - PDA bump seed
}
```

### Program Derived Addresses (PDAs)

**Charity PDA**: `[b"charity", authority.key(), name.as_bytes()]`
- Unique charity identification per authority
- Prevents name collisions

**Vault PDA**: `[b"vault", charity_pda.key()]` 
- Secure fund storage
- Program-owned account for donations

**Donation PDA**: `[b"donation", donor.key(), charity.key(), timestamp]`
- Individual donation tracking
- Temporal ordering of donations

## Security Features

### Access Control Matrix

| Operation | Authority | Donor | Public |
|-----------|-----------|--------|---------|
| Create Charity | ✅ | ❌ | ❌ |
| Update Charity | ✅ (own) | ❌ | ❌ |
| Delete Charity | ✅ (own) | ❌ | ❌ |
| Pause Donations | ✅ (own) | ❌ | ❌ |
| Withdraw Funds | ✅ (own) | ❌ | ❌ |
| Donate | ✅ | ✅ | ✅ |
| View Charity | ✅ | ✅ | ✅ |

### Validation Layers
- Business logic validation (e.g., donations not paused)
- Financial validation (sufficient funds)
- State consistency validation
- Rent exemption protection

## Error Handling

Custom error types provide clear feedback:
- `Unauthorized`: Access control violations
- `InvalidNameLength`: Input validation failures
- `DonationsPaused`: Business rule violations
- `InsufficientFunds`: Financial constraint violations

## Event System

Events enable real-time updates and off-chain indexing:
- `CreateCharityEvent`: New charity creation
- `MakeDonationEvent`: Donation transactions
- `WithdrawDonationsEvent`: Fund withdrawals
- `UpdateCharityEvent`: Charity modifications

This architecture ensures secure, transparent, and efficient charity management on the Solana blockchain.