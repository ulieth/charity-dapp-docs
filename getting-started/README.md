# Getting Started

Welcome to the Solana Charity dApp! This guide will help you set up your development environment and get the application running locally.

## What You'll Build

By following this guide, you'll have a fully functional charity donation platform running on your local machine, complete with:

- A Solana program (smart contract) deployed to localnet
- A modern React frontend for interacting with the blockchain
- Wallet integration for secure transactions
- Real-time donation tracking and management

## Overview

The Solana Charity dApp consists of three main components:

1. **Smart Contract (Anchor Program)**: Written in Rust, handles all on-chain logic
2. **Frontend Application**: Built with Next.js and React for the user interface
3. **Integration Layer**: TypeScript code that connects the frontend to the blockchain

## Project Structure

```
charity-dapp/
├── anchor/                    # Solana program (smart contract)
│   ├── programs/charity/      # Rust source code
│   ├── tests/                 # Program tests
│   └── Anchor.toml           # Anchor configuration
├── src/                       # Next.js frontend
│   ├── app/                   # App router pages
│   ├── components/            # React components
│   └── lib/                   # Utility functions
├── scripts/                   # Build and deployment scripts
└── package.json              # Node.js dependencies
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