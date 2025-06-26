# Smart Contract Architecture

The Solana Charity dApp's smart contract is built using the Anchor framework, providing a secure, efficient, and maintainable foundation for charity management and donations.

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
