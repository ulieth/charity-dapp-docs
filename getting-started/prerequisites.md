# Prerequisites

Before you can run the Solana Charity dApp, you'll need to install several tools and dependencies. This page will guide you through setting up your development environment.

## Required Tools

### 1. Node.js and npm

The frontend is built with Next.js, which requires Node.js.

```bash
# Install Node.js (version 18 or higher recommended)
# Download from: https://nodejs.org/

# Verify installation
node --version
npm --version
```

### 2. Rust

The smart contract is written in Rust using the Anchor framework.

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Configure PATH
source ~/.cargo/env

# Verify installation
rustc --version
cargo --version
```

### 3. Solana CLI

The Solana CLI tools are essential for blockchain development.

```bash
# Install Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/v1.18.22/install)"

# Add to PATH (add to your shell profile)
export PATH="~/.local/share/solana/install/active_release/bin:$PATH"

# Verify installation
solana --version
```

### 4. Anchor Framework

Anchor simplifies Solana program development.

```bash
# Install Anchor using AVM (Anchor Version Manager)
cargo install --git https://github.com/coral-xyz/anchor avm --locked --force

# Install latest Anchor version
avm install latest
avm use latest

# Verify installation
anchor --version
```

## Solana Wallet Setup

### 5. Create a Local Wallet

You'll need a Solana wallet for testing.

```bash
# Generate a new keypair
solana-keygen new --outfile ~/.config/solana/id.json

# Set as default keypair
solana config set --keypair ~/.config/solana/id.json

# Verify configuration
solana config get
```

### 6. Browser Wallet (Optional)

For the full user experience, install a browser wallet:

- **[Phantom](https://phantom.app/)** - Popular and user-friendly
- **[Solflare](https://solflare.com/)** - Feature-rich wallet
- **[Backpack](https://backpack.app/)** - Modern wallet with great UX

## Development Environment

### 7. Git

Version control is essential for development.

```bash
# Install Git (if not already installed)
# macOS: git is included with Xcode Command Line Tools
xcode-select --install

# Ubuntu/Debian
sudo apt-get install git

# Verify installation
git --version
```

### 8. Code Editor (Recommended)

While any editor works, these have excellent Rust and TypeScript support:

- **[Visual Studio Code](https://code.visualstudio.com/)** with extensions:
  - Rust Analyzer
  - Solana Extension Pack
  - TypeScript support
- **[IntelliJ IDEA](https://www.jetbrains.com/idea/)** with Rust plugin

## System Requirements

### Minimum Requirements

- **OS**: macOS, Linux, or Windows (WSL2 recommended)
- **RAM**: 8GB minimum, 16GB recommended
- **Storage**: 10GB free space
- **Network**: Stable internet connection for blockchain interactions

### Performance Tips

- **SSD Storage**: Faster builds and better performance
- **16GB+ RAM**: Smooth development experience
- **Multi-core CPU**: Faster Rust compilation

## Verification Checklist

Before proceeding, ensure all tools are properly installed:

```bash
# Check all required tools
node --version          # Should show v18+ 
npm --version           # Should show recent version
rustc --version         # Should show recent Rust version
cargo --version         # Should show recent Cargo version
solana --version        # Should show v1.18+
anchor --version        # Should show recent Anchor version
git --version           # Should show Git version
```

## Common Issues

### Solana CLI Not Found

If `solana` command is not found, add it to your PATH:

```bash
# Add to ~/.bashrc or ~/.zshrc
export PATH="~/.local/share/solana/install/active_release/bin:$PATH"

# Reload shell
source ~/.bashrc  # or ~/.zshrc
```

### Anchor Installation Issues

If Anchor installation fails:

```bash
# Update Rust first
rustup update

# Try alternative installation
cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked
```

### Permission Issues (Linux/macOS)

If you encounter permission errors:

```bash
# Fix npm permissions
mkdir ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

## Next Steps

Once all prerequisites are installed, continue to the [Installation](installation.md) guide to clone and set up the project.