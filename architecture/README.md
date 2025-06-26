# System Architecture

The Solana Charity dApp is built with a modern, scalable architecture that demonstrates best practices for full-stack blockchain development. This section provides a comprehensive overview of how all the components work together.

## High-Level Architecture

The system follows a three-tier architecture with a Rust/Anchor smart contract layer on Solana, a TypeScript/React frontend, and custom data access hooks bridging blockchain interactions with the user interface.

![High-Level Architecture](../assets/high-level-architecture.png)

## Core Components

### 1. Smart Contract Layer (Rust/Anchor)

**Location**: `anchor/programs/charity/`

A Rust/Anchor Solana program managing charity accounts, donations, withdrawals, and transparent event logging.

The Solana Charity dApp's smart contract is built using the Anchor framework, providing a secure, efficient, and maintainable foundation for charity management and donations.

The smart contract consists of six core instructions:

1. **create_charity**: Initialize new charity with vault
2. **donate_sol**: Transfer SOL from donor to charity vault
3. **withdraw_donations**: Transfer funds from vault to authority
4. **update_charity**: Modify charity description
5. **pause_donations**: Toggle donation acceptance
6. **delete_charity**: Remove charity and withdraw remaining funds

#### Error Handling

Custom error types provide clear feedback:
- `Unauthorized`: Access control violations
- `InvalidNameLength`: Input validation failures
- `DonationsPaused`: Business rule violations
- `InsufficientFunds`: Financial constraint violations

#### Event System

Events enable real-time updates and off-chain indexing:
- `CreateCharityEvent`: New charity creation
- `MakeDonationEvent`: Donation transactions
- `WithdrawDonationsEvent`: Fund withdrawals
- `UpdateCharityEvent`: Charity modifications

This architecture ensures secure, transparent, and efficient charity management on the Solana blockchain.

### 2. Frontend Application (TypeScript/React)

**Location**: `src/`

The charity dApp frontend is a modern React application built with Next.js 14, showcasing contemporary web development practices and seamless blockchain integration. Solana Provider provides blockchain connectivity to the entire application through context providers for wallet connection, network selection, and program interaction. Charity Data Access centralizes all charity-related blockchain operations with React Query for caching and synchronization.

### 3. Data Access Layer

**Location**: `src/components/charity/data-access/`

Custom React hooks that abstract blockchain interactions and provide a clean API for the UI components.

**Key Hooks**:
- `useProgram()` - Core program interactions
- `useAccount()` - Charity account management
- `useDonation()` - Donation operations
- `useBalance()` - Wallet balance tracking

## Data Flow Architecture

### Donation Flow Example

![Donation Flow](../assets/donation-flow.png)

## Account Architecture

### Program Derived Addresses (PDAs)

The application uses PDAs for deterministic account addresses:

```rust
// Charity PDA
// Unique charity identification per authority
// Prevents name collisions
[b"charity", authority.key().as_ref(), name.as_bytes()] -> Charity Account

// Vault PDA
// Secure fund storage
// Program-owned account for donations
[b"vault", charity.key().as_ref()] -> Vault Account

// Donation PDA
// Individual donation tracking
[b"donation", donor.key(), charity.key(), &charity.donation_count.to_le_bytes()] -> Donation Record
```

### Account Relationships

![Account Relationships Diagram](../assets/account-relationships.png)

The diagram shows how accounts are connected: Authority Wallets create and control Charity Accounts, which link to Vault PDAs that store SOL balances. Donor Wallets create Donation Accounts that reference the receiving charity.

## Security Architecture

The system ensures security through authority-based access control, program-owned fund storage, immutable smart contract logic, and transparent on-chain operations with full public auditability.

## Next Steps

Dive deeper into specific architectural components:

- **[Smart Contract Architecture](smart-contract.md)** - Detailed program structure
- **[Frontend Architecture](frontend.md)** - React application design
- **[Data Flow](data-flow.md)** - How data moves through the system
- **[Security Model](security.md)** - Security considerations and best practices

This architecture enables the charity dApp to be secure, scalable, and maintainable while showcasing modern blockchain development practices.
