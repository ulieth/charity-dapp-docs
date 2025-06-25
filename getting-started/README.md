# Getting Started

Welcome to the Solana Charity dApp! This guide will help you set up your development environment and get the application running locally.

## What You'll Build

By following this guide, you'll have a fully functional charity donation platform running on your local machine, complete with:

- A Solana program (smart contract) deployed to localnet
- A modern React frontend for interacting with the blockchain
- Wallet integration for secure transactions
- Real-time donation tracking and management

## 🏛️ Technical Architecture

This dApp consists of three main components:

- **Solana Smart Contract**: Written in Rust using the Anchor framework
- **Data Access Layer**: TypeScript hooks that interface with the Solana blockchain
- **React UI**: Modern, responsive interface for interacting with the contract

## Project Structure

```
charity_dapp/
├── 📁 scripts/                   # Development automation scripts
│   ├── airdrop.ts               # SOL airdrop utility for testing
│   ├── seed.ts                  # Blockchain seeding with test data
│   ├── checkProgramId.ts        # Program ID validation utility
│   ├── build_anchor.sh          # Anchor program build script
│   ├── deploy_anchor.sh         # Anchor program deployment script
│   └── test_anchor.sh           # Anchor program test runner
├── 📁 anchor/                    # Solana program (smart contract)
│   ├── 📁 programs/charity/      # Main program source
│   │   └── 📁 src/               # Rust source code
│   │       ├── lib.rs           # Program entry point & instruction handlers
│   │       ├── 📁 common/        # Shared utilities (constants, errors, events)
│   │       ├── 📁 instructions/  # Program instructions (create, donate, withdraw, etc.)
│   │       └── 📁 state/         # Account state definitions (charity, donation)
│   └── 📁 tests/                 # Program tests
└── 📁 src/                       # Next.js frontend application
    ├── 📁 app/                   # Next.js App Router
    └── 📁 components/            # Feature-based React components
        └── 📁 charity/           # Core charity functionality
            ├── 📁 data-access/   # React Query hooks, PDA utilities
            ├── 📁 feature/       # Page-level components
            ├── 📁 types/         # TypeScript interfaces
            ├── 📁 ui/            # Reusable UI components
            └── 📁 utils/         # Helper functions
```

## Key Features

- **Create Charities**: Set up fundraising campaigns with custom descriptions
- **Secure Donations**: Make SOL donations directly on-chain
- **Transparent Tracking**: View complete donation history
- **Fund Management**: Withdraw donations with flexible recipient options
- **Emergency Controls**: Pause/resume donations when needed

## Quick Navigation

- **[Prerequisites](prerequisites.md)** - Install required tools and dependencies
- **[Installation](installation.md)** - Clone and set up the project
- **[Quick Start](quick-start.md)** - Run the app in 5 minutes
- **[First Donation](first-donation.md)** - Make your first test donation

Let's get started! Continue to the [Prerequisites](prerequisites.md) page to ensure you have all the necessary tools installed.
