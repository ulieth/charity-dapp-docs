# Solana RPC Integration

The charity dApp integrates with the Solana blockchain through RPC connections, providing real-time access to blockchain data and transaction capabilities.

## Overview

The application uses multiple integration layers to interact with Solana:

- **Solana Web3.js**: Low-level blockchain operations
- **Anchor Client**: Type-safe program interactions  
- **Wallet Adapter**: Secure wallet connectivity
- **React Query**: Data caching and synchronization

## RPC Connection Setup

### Connection Configuration

```typescript
import { Connection, clusterApiUrl, Commitment } from '@solana/web3.js';

export type Cluster = 'devnet' | 'testnet' | 'mainnet-beta';

const COMMITMENT: Commitment = 'confirmed';

export function getConnection(cluster: Cluster): Connection {
  const endpoint = getClusterEndpoint(cluster);
  
  return new Connection(endpoint, {
    commitment: COMMITMENT,
    confirmTransactionInitialTimeout: 60000,
    disableRetryOnRateLimit: false,
  });
}
```

## Program Integration

### Anchor Client Setup

```typescript
import { AnchorProvider, Program, Wallet } from '@coral-xyz/anchor';
import { Connection, PublicKey } from '@solana/web3.js';

export const CHARITY_PROGRAM_ID = new PublicKey(
  process.env.NEXT_PUBLIC_CHARITY_PROGRAM_ID || 'YOUR_PROGRAM_ID'
);

export function getCharityProgram(
  connection: Connection,
  wallet: Wallet
): Program {
  const provider = new AnchorProvider(connection, wallet, {
    commitment: 'confirmed',
  });
  
  return new Program(CharityIDL, CHARITY_PROGRAM_ID, provider);
}
```

## Data Fetching

### Query Implementation

```typescript
import { useQuery } from '@tanstack/react-query';
import { useConnection, useWallet } from '@solana/wallet-adapter-react';

export function useAllCharities() {
  const { connection } = useConnection();
  const wallet = useWallet();
  
  return useQuery({
    queryKey: ['charities', 'all', connection.rpcEndpoint],
    queryFn: async () => {
      const program = getCharityProgram(connection, wallet as any);
      const accounts = await program.account.charity.all();
      
      return accounts.map(account => ({
        publicKey: account.publicKey,
        ...account.account,
      }));
    },
    staleTime: 30000, // 30 seconds
    refetchInterval: 60000, // 1 minute
  });
}
```

## Transaction Handling

### Transaction Execution

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useConnection, useWallet } from '@solana/wallet-adapter-react';

export function useTransaction() {
  const { connection } = useConnection();
  const { sendTransaction, publicKey } = useWallet();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (transaction: Transaction) => {
      if (!publicKey) throw new Error('Wallet not connected');
      
      const signature = await sendTransaction(transaction, connection);
      
      // Wait for confirmation
      const confirmation = await connection.confirmTransaction(signature, 'confirmed');
      
      if (confirmation.value.err) {
        throw new Error('Transaction failed');
      }
      
      return signature;
    },
    onSuccess: (signature) => {
      // Invalidate relevant queries
      queryClient.invalidateQueries({ queryKey: ['charities'] });
    },
  });
}
```

## Real-time Updates

### Event Listening

```typescript
export function useProgramEvents() {
  const { connection } = useConnection();
  const wallet = useWallet();
  const queryClient = useQueryClient();

  useEffect(() => {
    if (!connection || !wallet.publicKey) return;

    const program = getCharityProgram(connection, wallet as any);
    
    const listener = program.addEventListener('MakeDonationEvent', (event) => {
      // Update cache with new donation data
      queryClient.invalidateQueries({ 
        queryKey: ['charity', event.charityKey.toString()] 
      });
    });
    
    return () => {
      program.removeEventListener(listener);
    };
  }, [connection, wallet.publicKey]);
}
```

## Error Handling

### RPC Error Management

```typescript
export function handleRPCError(error: any): SolanaError {
  if (error.code === 4001) {
    return new SolanaError('Transaction rejected by user', 'USER_REJECTED');
  }
  
  if (error.message.includes('insufficient funds')) {
    return new SolanaError('Insufficient funds for transaction', 'INSUFFICIENT_FUNDS');
  }
  
  if (error.message.includes('DonationsPaused')) {
    return new SolanaError('Donations are currently paused', 'DONATIONS_PAUSED');
  }
  
  return new SolanaError('Transaction failed', 'UNKNOWN', error);
}
```

This integration provides efficient, reliable communication with the Solana blockchain while maintaining optimal user experience.