# Higher!! 

A provably fair crash game built on **Yellow Network's state channels** for instant, gasless multiplayer gameplay with cryptographic security.

## Overview

Higher!! is a multiplayer crash game where players predict a multiplier (1.01x-10.00x) before the rocket crashes. The game leverages Yellow Network's off-chain state channels for real-time, gas-free gameplay while maintaining the security and finality of on-chain settlements on Base Sepolia testnet.

### Key Features

- **Instant Gameplay**: Off-chain state channels eliminate gas costs and confirmation delays
- **Provably Fair**: Deterministic crash outcomes verified through cryptographic signatures
- **Multi-Player Support**: Real-time synchronization via Yellow Network's WebSocket RPC
- **Secure Settlements**: Multi-signature state updates with atomic on-chain finalization
- **Automated Flow**: Auto-start sessions, auto-sign transactions, and auto-reset lobbies

---

## 🟡 Yellow Network Integration

### What is Yellow Network?

Yellow Network provides **state channel infrastructure** for building scalable, low-latency trading and gaming applications. It enables off-chain state transitions with cryptographic guarantees, settling final results on-chain.

### How We Use Yellow Network

#### 1. **State Channels via Nitrolite SDK**

We integrate Yellow Network through the `@erc7824/nitrolite` SDK (v0.5.3), which provides:

- **Session Management**: Create multi-participant sessions with EIP-712 authorization
- **Off-Chain State Updates**: Apply state transitions (bets, outcomes) without blockchain transactions
- **Withdrawal Primitives**: Move funds between unified balance (off-chain) and on-chain wallets

```typescript
import { Nitrolite, ParticipantAuthorization } from '@erc7824/nitrolite';

// Initialize Nitrolite client
const client = new Nitrolite({ transport: wsRpcUrl });

// Authenticate with Yellow Network
await client.authenticateParticipant({ 
  challenge, 
  signature, 
  authToken 
});
```

#### 2. **Authentication Flow**

Players authenticate using **EIP-712 typed data signatures**:

```typescript
// Request authentication challenge from Yellow Network
const { challenge, authToken } = await client.requestParticipantAuthChallenge({
  appName: 'crash-test-app',
  scope: 'session-participant',
  participant: walletAddress,
  allowances: [{ asset: 'ytest.usd', amount: '1000000' }]
});

// Sign with session private key (client-side)
const signature = await account.signTypedData({
  domain: { name: 'crash-test-app', version: '1' },
  types: { /* EIP-712 types */ },
  primaryType: 'ParticipantAuth',
  message: challenge
});

// Complete authentication
await client.authenticateParticipant({ challenge, signature, authToken });
```

#### 3. **Session Lifecycle**

Each game round is a Yellow Network session:

```typescript
// Start a new session (server-side)
const session = await client.startSession({
  sessionId: crypto.randomUUID(),
  participants: [adminAddress, ...playerAddresses],
  allowances: { 'ytest.usd': '1000000' }
});

// Authorize participant (client-side)
await client.authorizeParticipantSession({
  sessionId,
  participant: walletAddress,
  signature: await wallet.signTypedData(authTypedData)
});
```

#### 4. **Pending Actions (State Updates)**

Game logic (bets, payouts) is encoded as **pending actions**:

```typescript
// Create start action (collects bets)
const startAction = await client.createPendingAction({
  sessionId,
  actionType: 'start',
  balanceChanges: {
    [player1]: { 'ytest.usd': '-100' }, // Bet amount
    [player2]: { 'ytest.usd': '-200' }
  }
});

// Players sign the action
await client.signPendingAction({ 
  actionId: startAction.id, 
  signature 
});

// Finalize when all signatures collected
await client.finalizePendingAction({ actionId: startAction.id });
```

#### 5. **End Action & Settlements**

After the crash, payouts are calculated and applied:

```typescript
// Create end action (distribute payouts)
const endAction = await client.createPendingAction({
  sessionId,
  actionType: 'end',
  balanceChanges: {
    [winner1]: { 'ytest.usd': '+150' }, // 1.5x payout
    [winner2]: { 'ytest.usd': '+0' }     // Lost bet
  }
});

// Multi-sig signing by all participants
// Finalize to commit state changes
await client.finalizePendingAction({ actionId: endAction.id });

// End session (final on-chain settlement)
await client.endSession({ sessionId });
```

#### 6. **Withdrawal Flows**

Yellow Network supports two withdrawal paths:

**A. Unified to On-Chain (via settlement contract)**
```typescript
const { tx } = await client.prepareWithdrawUnifiedToOnchainTransaction({
  participant: walletAddress,
  asset: 'ytest.usd',
  amount: '500'
});

// Submit transaction on-chain
await wallet.sendTransaction(tx);
```

**B. Layer-2 to Hot Wallet (internal transfer)**
```typescript
await client.prepareWithdrawToWallet({
  participant: walletAddress,
  asset: 'ytest.usd',
  amount: '500'
});

// Execute withdrawal
await client.executeWithdrawToWallet({ withdrawalId });
```


## 🚀 Getting Started

### Prerequisites

- Node.js 18+ and npm
- MetaMask or compatible Web3 wallet
- Base Sepolia testnet ETH (for gas)
- yUSDC tokens ([get from faucet](https://clearnet-sandbox.yellow.com))

### Installation

```bash
# Clone the repository
git clone https://github.com/your-repo/higher.git
cd higher/frontend

# Install dependencies
npm install
```

### Environment Variables

Create a `.env.local` file:

```bash
# Admin wallet (server-side signing)
ADMIN_PRIVATE_KEY=0x...

# Yellow Network Configuration
YELLOW_WS_URL=wss://clearnet-sandbox.yellow.com/ws
YELLOW_APP_NAME=crash-test-app
YELLOW_ASSET_ID=ytest.usd
YELLOW_AUTH_ALLOWANCE=1000000

# Blockchain Configuration
NEXT_PUBLIC_BASE_CHAIN_ID=84532
NEXT_PUBLIC_TOKEN_SYMBOL=yUSDC
```

### Run Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) and navigate to `/game`.

### Build for Production

```bash
npm run build
npm start
```

---

## How to Play

1. **Connect Wallet**: Click "Connect Wallet" and select your Web3 wallet
2. **Join Lobby**: Click "Join Game" (approves yUSDC token and joins lobby)
3. **Place Bet**: Enter a multiplier prediction (1.01x - 10.00x)
4. **Auto-Start**: Game auto-starts when players join
5. **Auto-Sign**: Your wallet auto-signs the session authorization
6. **Watch Crash**: The multiplier rises until the rocket crashes
7. **Settlement**: Winners are paid out automatically via Yellow Network
8. **Auto-Reset**: Lobby resets after 5 seconds for the next round


## Development Notes

### Automation Features

- **Auto-Start**: Fires 10s after last player joins (see `tryAutoStartSession` in `/api/game/join`)
- **Auto-Sign**: Client auto-signs pending actions when detected (see `useGameLoop.ts`)
- **Auto-End**: Server auto-ends session 6s after crash (see `page.tsx`)
- **Auto-Reset**: Client triggers reset 5s after game ends

### Minimum Players

Set to `MIN_PLAYERS = 1` for solo testing. Change in `src/server/game-store.ts`:

```typescript
const MIN_PLAYERS = 1; // Change to 2 for multiplayer
```


**Built with 💛 on Yellow Network**
