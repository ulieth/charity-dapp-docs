# API Reference

This comprehensive API reference documents all the interfaces, methods, and types used in the Solana Charity dApp. Use this as a quick reference while developing or integrating with the application.

## Smart Contract API

### Program Instructions

#### createCharity

Creates a new charity account with associated vault.

```rust
pub fn create_charity(
    ctx: Context<CreateCharity>,
    name: String,
    description: String,
) -> Result<()>
```

**Parameters:**
- `name`: String (1-50 characters) - Charity name
- `description`: String (1-500 characters) - Charity description

**Accounts:**
- `authority`: Signer - Charity owner's wallet
- `charity`: PDA - Charity account to create
- `vault`: PDA - Vault account for donations
- `system_program`: Program - Solana system program

**Events Emitted:**
- `CreateCharityEvent`

**Errors:**
- `InvalidNameLength` - Name too long or empty
- `InvalidDescriptionLength` - Description too long or empty

#### donateSol

Transfer SOL from donor to charity vault.

```rust
pub fn donate_sol(
    ctx: Context<DonateSol>,
    amount: u64,
) -> Result<()>
```

**Parameters:**
- `amount`: u64 - Amount in lamports to donate

**Accounts:**
- `donor`: Signer - Donor's wallet
- `charity`: Account - Target charity
- `vault`: PDA - Charity's vault account
- `donation`: PDA - Donation record to create
- `system_program`: Program - Solana system program

**Events Emitted:**
- `MakeDonationEvent`

**Errors:**
- `DonationsPaused` - Charity has paused donations
- `InsufficientFunds` - Donor has insufficient balance
- `InvalidVaultAccount` - Vault PDA mismatch

#### withdrawDonations

Withdraw funds from charity vault to recipient.

```rust
pub fn withdraw_donations(
    ctx: Context<WithdrawDonations>,
    amount: u64,
) -> Result<()>
```

**Parameters:**
- `amount`: u64 - Amount in lamports to withdraw

**Accounts:**
- `authority`: Signer - Charity authority
- `charity`: Account - Charity account
- `vault`: PDA - Charity's vault account
- `recipient`: Account - Recipient wallet

**Events Emitted:**
- `WithdrawCharitySolEvent`

**Errors:**
- `Unauthorized` - Only charity authority can withdraw
- `InsufficientFunds` - Vault has insufficient balance
- `InsufficientFundsForRent` - Would break rent exemption

#### pauseDonations

Toggle donation acceptance for charity.

```rust
pub fn pause_donations(
    ctx: Context<PauseDonations>,
    paused: bool,
) -> Result<()>
```

**Parameters:**
- `paused`: bool - true to pause, false to unpause

**Accounts:**
- `authority`: Signer - Charity authority
- `charity`: Account - Charity account to update

**Events Emitted:**
- `PauseDonationsEvent`

**Errors:**
- `Unauthorized` - Only charity authority can pause

#### updateCharity

Update charity description.

```rust
pub fn update_charity(
    ctx: Context<UpdateCharity>,
    description: String,
) -> Result<()>
```

**Parameters:**
- `description`: String (1-500 characters) - New description

**Accounts:**
- `authority`: Signer - Charity authority
- `charity`: Account - Charity account to update

**Events Emitted:**
- `UpdateCharityEvent`

**Errors:**
- `Unauthorized` - Only charity authority can update
- `InvalidDescription` - Same as current description
- `InvalidDescriptionLength` - Description too long

#### deleteCharity

Delete charity and withdraw remaining funds.

```rust
pub fn delete_charity(
    ctx: Context<DeleteCharity>
) -> Result<()>
```

**Accounts:**
- `authority`: Signer - Charity authority
- `charity`: Account - Charity account to delete
- `vault`: PDA - Charity's vault account
- `recipient`: Account - Recipient for remaining funds

**Events Emitted:**
- `DeleteCharityEvent`

**Errors:**
- `Unauthorized` - Only charity authority can delete

### Account Structures

#### Charity

Main charity account structure.

```rust
#[account]
pub struct Charity {
    pub authority: Pubkey,              // Charity owner
    pub name: String,                   // Charity name
    pub description: String,            // Description
    pub donations_in_lamports: u64,     // Total donations received
    pub donation_count: u32,            // Number of donations
    pub paused: bool,                   // Donation status
    pub created_at: i64,                // Creation timestamp
    pub updated_at: i64,                // Last update timestamp
    pub deleted_at: Option<i64>,        // Deletion timestamp
    pub withdrawn_at: Option<i64>,      // Last withdrawal timestamp
    pub vault_bump: u8,                 // Vault PDA bump seed
}
```

#### Donation

Individual donation record.

```rust
#[account]
pub struct Donation {
    pub donor_key: Pubkey,              // Donor's wallet address
    pub charity_key: Pubkey,            // Target charity address
    pub charity_name: String,           // Charity name snapshot
    pub amount_in_lamports: u64,        // Donation amount
    pub created_at: i64,                // Donation timestamp
}
```

### Events

#### CreateCharityEvent

```rust
#[event]
pub struct CreateCharityEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub description: String,
    pub created_at: i64,
}
```

#### MakeDonationEvent

```rust
#[event]
pub struct MakeDonationEvent {
    pub donor_key: Pubkey,
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub amount: u64,
    pub created_at: i64,
}
```

#### WithdrawCharitySolEvent

```rust
#[event]
pub struct WithdrawCharitySolEvent {
    pub charity_key: Pubkey,
    pub charity_name: String,
    pub donations_in_lamports: u64,
    pub donation_count: u32,
    pub withdrawn_at: i64,
}
```

### Error Codes

```rust
#[error_code]
pub enum CustomError {
    #[msg("Unauthorized: Only charity authority can perform this action")]
    Unauthorized = 6000,
    
    #[msg("Invalid name length: Name must be between 1 and 50 characters")]
    InvalidNameLength = 6001,
    
    #[msg("Invalid description length: Description must be between 1 and 500 characters")]
    InvalidDescriptionLength = 6002,
    
    #[msg("Invalid description: Must be different from current description")]
    InvalidDescription = 6003,
    
    #[msg("Donations are currently paused for this charity")]
    DonationsPaused = 6004,
    
    #[msg("Insufficient funds for this operation")]
    InsufficientFunds = 6005,
    
    #[msg("Insufficient funds to maintain rent exemption")]
    InsufficientFundsForRent = 6006,
    
    #[msg("Invalid vault account")]
    InvalidVaultAccount = 6007,
    
    #[msg("Arithmetic overflow")]
    Overflow = 6008,
}
```

## Frontend API

### Data Access Hooks

#### useProgram()

Main hook for program interactions.

```typescript
export function useProgram(): {
  // Queries
  getAllCharities: UseQueryResult<CharityAccount[]>;
  getMyCharities: UseQueryResult<CharityAccount[]>;
  getMyDonations: UseQueryResult<DonationAccount[]>;
  
  // Mutations
  createCharity: UseMutationResult<string, Error, CreateCharityArgs>;
  updateCharity: UseMutationResult<string, Error, UpdateCharityArgs>;
  donate: UseMutationResult<string, Error, DonateArgs>;
  pauseDonations: UseMutationResult<string, Error, PauseDonationsArgs>;
  withdrawDonations: UseMutationResult<string, Error, WithdrawDonationsArgs>;
  deleteCharity: UseMutationResult<string, Error, DeleteCharityArgs>;
}
```

#### useAccount(charityKey)

Hook for specific charity account data.

```typescript
export function useAccount(charityKey: PublicKey | undefined): {
  charityQuery: UseQueryResult<CharityAccount | null>;
  donationsQuery: UseQueryResult<DonationAccount[]>;
  vaultBalanceQuery: UseQueryResult<number | null>;
}
```

### Type Definitions

#### CharityAccount

```typescript
interface CharityAccount {
  publicKey: PublicKey;
  authority: PublicKey;
  name: string;
  description: string;
  donationsInLamports: BN;
  donationCount: number;
  paused: boolean;
  createdAt: BN;
  updatedAt: BN;
  deletedAt: BN | null;
  withdrawnAt: BN | null;
  vaultBump: number;
}
```

#### DonationAccount

```typescript
interface DonationAccount {
  publicKey: PublicKey;
  donorKey: PublicKey;
  charityKey: PublicKey;
  charityName: string;
  amountInLamports: BN;
  createdAt: BN;
}
```

#### CreateCharityArgs

```typescript
interface CreateCharityArgs {
  name: string;
  description: string;
}
```

#### DonateArgs

```typescript
interface DonateArgs {
  charity: string;  // PublicKey as string
  amount: number;   // Amount in lamports
}
```

#### WithdrawDonationsArgs

```typescript
interface WithdrawDonationsArgs {
  charity: string;    // PublicKey as string
  recipient: string;  // PublicKey as string
  amount: number;     // Amount in lamports
}
```

### Utility Functions

#### PDA Derivation

```typescript
// Find charity PDA
export const findCharityPda = (
  name: string,
  authority: PublicKey,
  programId: PublicKey
): [PublicKey, number] => {
  return PublicKey.findProgramAddressSync(
    [Buffer.from("charity"), authority.toBuffer(), Buffer.from(name)],
    programId
  );
};

// Find vault PDA
export const findVaultPda = (
  charityPda: PublicKey,
  programId: PublicKey
): [PublicKey, number] => {
  return PublicKey.findProgramAddressSync(
    [Buffer.from("vault"), charityPda.toBuffer()],
    programId
  );
};

// Find donation PDA
export const findDonationPda = (
  donor: PublicKey,
  charity: PublicKey,
  timestamp: number,
  programId: PublicKey
): [PublicKey, number] => {
  return PublicKey.findProgramAddressSync(
    [
      Buffer.from("donation"),
      donor.toBuffer(),
      charity.toBuffer(),
      Buffer.from(timestamp.toString())
    ],
    programId
  );
};
```

#### Format Utilities

```typescript
// Format SOL amount
export const formatSol = (lamports: number | BN): string => {
  const sol = typeof lamports === 'number' 
    ? lamports / LAMPORTS_PER_SOL 
    : lamports.toNumber() / LAMPORTS_PER_SOL;
  return `${sol.toFixed(4)} SOL`;
};

// Format date
export const formatDate = (timestamp: number | BN): string => {
  const date = new Date(
    (typeof timestamp === 'number' ? timestamp : timestamp.toNumber()) * 1000
  );
  return date.toLocaleDateString();
};

// Truncate public key
export const truncatePublicKey = (
  publicKey: PublicKey | string,
  length: number = 8
): string => {
  const key = publicKey.toString();
  return `${key.slice(0, length)}...${key.slice(-length)}`;
};
```

## Constants

### Program Constants

```typescript
// Maximum string lengths
export const CHARITY_NAME_MAX_LEN = 50;
export const CHARITY_DESCRIPTION_MAX_LEN = 500;

// PDA seeds
export const CHARITY_SEED = "charity";
export const VAULT_SEED = "vault";
export const DONATION_SEED = "donation";

// Solana constants
export const LAMPORTS_PER_SOL = 1_000_000_000;
export const ANCHOR_DISCRIMINATOR_SIZE = 8;
```

### Network Constants

```typescript
// RPC endpoints
export const RPC_ENDPOINTS = {
  'localhost': 'http://localhost:8899',
  'devnet': 'https://api.devnet.solana.com',
  'testnet': 'https://api.testnet.solana.com',
  'mainnet-beta': 'https://api.mainnet-beta.solana.com',
} as const;

// Program IDs
export const PROGRAM_IDS = {
  'localhost': new PublicKey('9MipEJLetsngpXJuyCLsSu3qTJrHQ6E6W1rZ1GrG68am'),
  'devnet': new PublicKey('9MipEJLetsngpXJuyCLsSu3qTJrHQ6E6W1rZ1GrG68am'),
  'mainnet-beta': new PublicKey('YOUR_MAINNET_PROGRAM_ID'),
} as const;
```

## Usage Examples

### Creating a Charity

```typescript
const { createCharity } = useProgram();

const handleCreateCharity = async () => {
  try {
    const signature = await createCharity.mutateAsync({
      name: "Environmental Fund",
      description: "Supporting environmental conservation projects"
    });
    console.log('Charity created:', signature);
  } catch (error) {
    console.error('Failed to create charity:', error);
  }
};
```

### Making a Donation

```typescript
const { donate } = useProgram();

const handleDonate = async (charityPublicKey: PublicKey) => {
  try {
    const signature = await donate.mutateAsync({
      charity: charityPublicKey.toString(),
      amount: 0.1 * LAMPORTS_PER_SOL // 0.1 SOL
    });
    console.log('Donation successful:', signature);
  } catch (error) {
    console.error('Donation failed:', error);
  }
};
```

### Fetching Charity Data

```typescript
const { getAllCharities } = useProgram();

// Charities are automatically fetched and cached
const charities = getAllCharities.data || [];

// Access individual charity data
charities.forEach(charity => {
  console.log('Charity:', charity.name);
  console.log('Total raised:', formatSol(charity.donationsInLamports));
  console.log('Donations:', charity.donationCount);
});
```

This API reference provides all the information needed to interact with the Solana Charity dApp programmatically. Use it as a quick lookup while building features or integrating with the platform.