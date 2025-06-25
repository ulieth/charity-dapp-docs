# Project Structure

The frontend application follows a modern Next.js 14 project structure with feature-based organization and clean separation of concerns.

## Root Directory Structure

```
charity-dapp/
├── src/                          # Source code
│   ├── app/                     # Next.js 14 App Router
│   ├── components/              # React components
│   ├── lib/                     # Utility libraries
│   ├── hooks/                   # Custom React hooks
│   ├── types/                   # TypeScript type definitions
│   ├── constants/               # Application constants
│   └── styles/                  # Global styles
├── public/                      # Static assets
├── anchor/                      # Anchor workspace (if included)
├── .env.local                   # Environment variables
├── next.config.js              # Next.js configuration
├── tailwind.config.js          # Tailwind CSS configuration
├── tsconfig.json               # TypeScript configuration
└── package.json                # Dependencies and scripts
```

## App Router Structure (`src/app/`)

```
src/app/
├── layout.tsx                   # Root application layout
├── page.tsx                     # Home page
├── loading.tsx                  # Global loading UI
├── error.tsx                    # Global error UI
├── not-found.tsx               # 404 page
├── globals.css                 # Global CSS styles
├── charity/                    # Charity routes
│   ├── page.tsx               # Charity listing page
│   ├── loading.tsx            # Charity loading state
│   ├── [charityId]/           # Dynamic charity routes
│   │   ├── page.tsx          # Charity detail page
│   │   ├── edit/             # Edit charity route
│   │   │   └── page.tsx
│   │   └── donations/        # Donation history
│   │       └── page.tsx
│   └── create/                # Create charity route
│       └── page.tsx
├── account/                   # Account management
│   ├── page.tsx              # Account overview
│   ├── donations/            # User's donation history
│   │   └── page.tsx
│   └── charities/            # User's managed charities
│       └── page.tsx
├── api/                      # API routes (if needed)
│   └── health/
│       └── route.ts
└── (auth)/                   # Route groups for authenticated pages
    └── dashboard/
        └── page.tsx
```

## Components Structure (`src/components/`)

### Feature-Based Organization

```
src/components/
├── charity/                     # Charity-related components
│   ├── data-access/            # Data fetching hooks
│   │   ├── charity-data-access.tsx
│   │   ├── use-program.tsx
│   │   └── use-charity-account.tsx
│   ├── feature/               # Complex feature components
│   │   ├── charity-list.tsx
│   │   ├── charity-detail.tsx
│   │   ├── charity-form.tsx
│   │   └── donation-form.tsx
│   ├── ui/                    # Simple UI components
│   │   ├── charity-card.tsx
│   │   ├── charity-stats.tsx
│   │   ├── donation-item.tsx
│   │   └── charity-actions.tsx
│   ├── types/                 # TypeScript interfaces
│   │   └── charity.types.ts
│   └── utils/                 # Utility functions
│       ├── charity.utils.ts
│       └── validation.ts
├── account/                   # Account-related components
│   ├── data-access/
│   │   ├── account-data-access.tsx
│   │   └── use-balance.tsx
│   ├── feature/
│   │   ├── account-overview.tsx
│   │   └── transaction-history.tsx
│   ├── ui/
│   │   ├── balance-display.tsx
│   │   └── wallet-button.tsx
│   └── types/
│       └── account.types.ts
├── cluster/                   # Network cluster components
│   ├── cluster-data-access.tsx
│   ├── cluster-ui.tsx
│   └── cluster-feature.tsx
├── solana/                    # Solana integration
│   ├── solana-provider.tsx
│   └── wallet-adapter.tsx
└── ui/                        # Shared UI components
    ├── ui-layout.tsx
    ├── ui-modal.tsx
    ├── ui-button.tsx
    ├── ui-input.tsx
    ├── ui-toast.tsx
    └── ui-loading.tsx
```

## Component Architecture Patterns

### Data Access Layer

```typescript
// src/components/charity/data-access/charity-data-access.tsx
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useAnchorProvider } from '@/components/solana/solana-provider';
import { getCharityProgram } from '@/lib/charity-program';

export function useCharityProgram() {
  const provider = useAnchorProvider();
  const queryClient = useQueryClient();

  const program = useMemo(() => {
    if (!provider) return null;
    return getCharityProgram(provider);
  }, [provider]);

  // Queries
  const getAllCharities = useQuery({
    queryKey: ['charity', 'all', { cluster: provider?.connection.rpcEndpoint }],
    queryFn: async () => {
      if (!program) throw new Error('Program not initialized');
      return await program.account.charity.all();
    },
    enabled: !!program,
  });

  // Mutations
  const createCharity = useMutation({
    mutationFn: async ({ name, description }: { name: string; description: string }) => {
      if (!program) throw new Error('Program not initialized');
      return await program.methods
        .createCharity(name, description)
        .accounts({
          authority: provider?.wallet.publicKey,
        })
        .rpc();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['charity'] });
    },
  });

  return {
    program,
    getAllCharities,
    createCharity,
    // ... other operations
  };
}
```

### Feature Components

```typescript
// src/components/charity/feature/charity-list.tsx
'use client';

import { useState } from 'react';
import { useCharityProgram } from '../data-access/charity-data-access';
import { CharityCard } from '../ui/charity-card';
import { CreateCharityModal } from './charity-form';

export function CharityList() {
  const [showCreateModal, setShowCreateModal] = useState(false);
  const { getAllCharities, createCharity } = useCharityProgram();

  const charities = getAllCharities.data || [];

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-3xl font-bold">Charities</h1>
        <button
          onClick={() => setShowCreateModal(true)}
          className="btn btn-primary"
        >
          Create Charity
        </button>
      </div>

      {getAllCharities.isLoading && (
        <div className="text-center">Loading charities...</div>
      )}

      {getAllCharities.error && (
        <div className="text-center text-error">
          Error loading charities: {getAllCharities.error.message}
        </div>
      )}

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {charities.map((charity) => (
          <CharityCard
            key={charity.publicKey.toString()}
            charity={charity}
          />
        ))}
      </div>

      {showCreateModal && (
        <CreateCharityModal
          onClose={() => setShowCreateModal(false)}
          onSubmit={createCharity.mutate}
          isLoading={createCharity.isLoading}
        />
      )}
    </div>
  );
}
```

### UI Components

```typescript
// src/components/charity/ui/charity-card.tsx
import { LAMPORTS_PER_SOL } from '@solana/web3.js';
import Link from 'next/link';

interface CharityCardProps {
  charity: {
    publicKey: PublicKey;
    account: {
      name: string;
      description: string;
      donations_in_lamports: number;
      donation_count: number;
      paused: boolean;
    };
  };
}

export function CharityCard({ charity }: CharityCardProps) {
  const { account } = charity;
  const donationsInSol = account.donations_in_lamports / LAMPORTS_PER_SOL;

  return (
    <div className="card bg-base-100 shadow-xl">
      <div className="card-body">
        <h2 className="card-title">
          {account.name}
          {account.paused && (
            <div className="badge badge-warning">Paused</div>
          )}
        </h2>
        
        <p className="text-base-content/70 line-clamp-3">
          {account.description}
        </p>

        <div className="stats stats-horizontal mt-4">
          <div className="stat">
            <div className="stat-title">Total Raised</div>
            <div className="stat-value text-primary">
              {donationsInSol.toFixed(2)} SOL
            </div>
          </div>
          <div className="stat">
            <div className="stat-title">Donations</div>
            <div className="stat-value text-secondary">
              {account.donation_count}
            </div>
          </div>
        </div>

        <div className="card-actions justify-end mt-4">
          <Link
            href={`/charity/${charity.publicKey.toString()}`}
            className="btn btn-primary"
          >
            View Details
          </Link>
        </div>
      </div>
    </div>
  );
}
```

## Utility Libraries (`src/lib/`)

```
src/lib/
├── charity-program.ts          # Anchor program utilities
├── solana-utils.ts            # Solana helper functions
├── format-utils.ts            # Data formatting utilities
├── validation-schemas.ts      # Zod validation schemas
├── constants.ts               # Application constants
└── types.ts                   # Shared TypeScript types
```

### Program Utilities

```typescript
// src/lib/charity-program.ts
import { AnchorProvider, Program } from '@coral-xyz/anchor';
import { PublicKey } from '@solana/web3.js';
import { CharityIDL } from './charity-idl';

export const CHARITY_PROGRAM_ID = new PublicKey('YOUR_PROGRAM_ID_HERE');

export function getCharityProgram(provider: AnchorProvider) {
  return new Program(CharityIDL, CHARITY_PROGRAM_ID, provider);
}

export function getCharityPDA(authority: PublicKey, name: string) {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('charity'), authority.toBuffer(), Buffer.from(name)],
    CHARITY_PROGRAM_ID
  );
}

export function getVaultPDA(charity: PublicKey) {
  return PublicKey.findProgramAddressSync(
    [Buffer.from('vault'), charity.toBuffer()],
    CHARITY_PROGRAM_ID
  );
}
```

## Custom Hooks (`src/hooks/`)

```
src/hooks/
├── use-local-storage.ts       # Local storage hook
├── use-debounce.ts           # Debounce hook
├── use-intersection.ts       # Intersection observer hook
└── use-toast.ts              # Toast notification hook
```

### Local Storage Hook

```typescript
// src/hooks/use-local-storage.ts
import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(() => {
    if (typeof window === 'undefined') return defaultValue;
    
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : defaultValue;
    } catch (error) {
      console.warn(`Error reading localStorage key "${key}":`, error);
      return defaultValue;
    }
  });

  useEffect(() => {
    if (typeof window === 'undefined') return;
    
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.warn(`Error setting localStorage key "${key}":`, error);
    }
  }, [key, value]);

  return [value, setValue] as const;
}
```

## Type Definitions (`src/types/`)

```
src/types/
├── charity.ts                 # Charity-related types
├── solana.ts                 # Solana-specific types
├── ui.ts                     # UI component types
└── api.ts                    # API response types
```

### Charity Types

```typescript
// src/types/charity.ts
import { PublicKey } from '@solana/web3.js';

export interface Charity {
  publicKey: PublicKey;
  account: {
    authority: PublicKey;
    name: string;
    description: string;
    donations_in_lamports: number;
    donation_count: number;
    paused: boolean;
    created_at: number;
    updated_at: number;
    vault_bump: number;
  };
}

export interface CreateCharityForm {
  name: string;
  description: string;
}

export interface DonationForm {
  amount: number;
  charityId: string;
}

export interface CharityStats {
  totalDonations: number;
  donationCount: number;
  averageDonation: number;
  lastDonationDate?: Date;
}
```

## Configuration Files

### Next.js Configuration

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
  webpack: (config) => {
    config.resolve.fallback = {
      fs: false,
      net: false,
      tls: false,
    };
    return config;
  },
  env: {
    NEXT_PUBLIC_SOLANA_NETWORK: process.env.NEXT_PUBLIC_SOLANA_NETWORK,
    NEXT_PUBLIC_RPC_URL: process.env.NEXT_PUBLIC_RPC_URL,
  },
};

module.exports = nextConfig;
```

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: '#9945FF', // Solana purple
        secondary: '#14F195', // Solana green
      },
    },
  },
  plugins: [require('daisyui')],
  daisyui: {
    themes: ['light', 'dark'],
  },
};
```

## File Naming Conventions

- **Components**: PascalCase (`CharityCard.tsx`)
- **Hooks**: camelCase with `use` prefix (`useCharityProgram.ts`)
- **Utilities**: camelCase (`formatSolAmount.ts`)
- **Types**: camelCase with `.types.ts` suffix (`charity.types.ts`)
- **Constants**: UPPER_SNAKE_CASE (`CHARITY_PROGRAM_ID`)

## Import Organization

```typescript
// External libraries
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { PublicKey } from '@solana/web3.js';

// Internal utilities
import { getCharityProgram } from '@/lib/charity-program';
import { formatSolAmount } from '@/lib/format-utils';

// Components
import { CharityCard } from '@/components/charity/ui/charity-card';

// Types
import type { Charity } from '@/types/charity';
```

This project structure provides a scalable, maintainable foundation for the charity dApp frontend while following Next.js 14 and React best practices.