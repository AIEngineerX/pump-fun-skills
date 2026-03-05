---
name: tokenized-agents
description: Build payment flows for Pump Tokenized Agents using @pump-fun/agent-payments-sdk. Use when accepting payments, building accept-payment transactions, integrating Solana wallets, or verifying that a user has paid an invoice on-chain.
metadata:
  author: pump-fun
  version: "2.0"
---

# Pump Tokenized Agent Payments

Pump Tokenized Agents are AI agents whose revenue is linked to a token on pump.fun. The `@pump-fun/agent-payments-sdk` lets you build payment transactions and verify invoices on Solana.

## Before You Start — Gather Required Information

Before writing any code, ask the user for the following if not already provided:

1. **Agent token mint address** — the token mint created when the agent coin was launched on pump.fun.
2. **Payment currency** — USDC or SOL (determines the `currencyMint`).
3. **Price / amount** — how much to charge per request, in the currency's smallest unit.
4. **Service to deliver** — what the agent should do after payment is confirmed (generate content, access an API, etc.).

Do not assume these values. If any are missing, ask the user before proceeding.

## Safety Rules

- **NEVER** log, print, or return private keys or secret key material.
- **NEVER** sign transactions on behalf of a user — you build the instruction, the user signs.
- Always validate that `amount > 0` before creating an invoice.
- Always ensure `endTime > startTime` and both are valid Unix timestamps.
- Use the correct decimal precision for the currency (6 decimals for USDC, 9 for SOL).

## Supported Currencies

| Currency    | Decimals | Smallest unit example        |
|-------------|----------|------------------------------|
| USDC        | 6        | `1000000` = 1 USDC           |
| Wrapped SOL | 9        | `1000000000` = 1 SOL         |

## Environment Variables

Create a `.env` (or `.env.local` for Next.js) file with the following:

```env
# Solana RPC — server-side (used to build transactions and verify payments)
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com

# Solana RPC — client-side (used by wallet adapter in the browser)
NEXT_PUBLIC_SOLANA_RPC_URL=https://api.mainnet-beta.solana.com

# The token mint address of your tokenized agent on pump.fun
AGENT_TOKEN_MINT_ADDRESS=<your-agent-mint-address>

# Payment currency mint
# USDC: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
# SOL (wrapped): So11111111111111111111111111111111111111112
CURRENCY_MINT=EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
```

Read these values from `process.env` at runtime. Never hard-code mint addresses or RPC URLs.

## Install

```bash
npm install @pump-fun/agent-payments-sdk @solana/web3.js @coral-xyz/anchor
```

## SDK Setup

`PumpAgent` is the main class. It can build payment instructions and verify invoices.

```typescript
import { PumpAgent } from "@pump-fun/agent-payments-sdk";
import { PublicKey } from "@solana/web3.js";

const agentMint = new PublicKey(process.env.AGENT_TOKEN_MINT_ADDRESS!);
```

### Constructor

```typescript
new PumpAgent(mint: PublicKey, environment?: "mainnet" | "devnet", connection?: Connection)
```

| Parameter     | Type                       | Default      | Description                                      |
|---------------|----------------------------|--------------|--------------------------------------------------|
| `mint`        | `PublicKey`                | —            | The tokenized agent's token mint address         |
| `environment` | `"mainnet"` \| `"devnet"` | `"mainnet"`  | Network environment                              |
| `connection`  | `Connection` (optional)    | `undefined`  | Solana RPC connection (enables RPC fallback for verification) |

**Without connection** — enough for building instructions and HTTP-based payment verification:

```typescript
const agent = new PumpAgent(agentMint);
```

**With connection** — also enables RPC-based verification fallback and balance queries:

```typescript
import { Connection } from "@solana/web3.js";

const connection = new Connection(process.env.SOLANA_RPC_URL!);
const agent = new PumpAgent(agentMint, "mainnet", connection);
```

## Wallet Integration (Frontend)

To let users sign transactions in the browser, install the Solana wallet adapter:

```bash
npm install @solana/wallet-adapter-react @solana/wallet-adapter-react-ui @solana/wallet-adapter-wallets
```

### WalletProvider Component

Create a provider that wraps your app:

```tsx
"use client";

import { useMemo } from "react";
import {
  ConnectionProvider,
  WalletProvider as SolanaWalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider } from "@solana/wallet-adapter-react-ui";
import {
  PhantomWalletAdapter,
  SolflareWalletAdapter,
} from "@solana/wallet-adapter-wallets";

import "@solana/wallet-adapter-react-ui/styles.css";

export default function WalletProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  const endpoint =
    process.env.NEXT_PUBLIC_SOLANA_RPC_URL ||
    "https://api.mainnet-beta.solana.com";

  const wallets = useMemo(
    () => [new PhantomWalletAdapter(), new SolflareWalletAdapter()],
    [],
  );

  return (
    <ConnectionProvider endpoint={endpoint}>
      <SolanaWalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>{children}</WalletModalProvider>
      </SolanaWalletProvider>
    </ConnectionProvider>
  );
}
```

### Wrap Your App Layout

```tsx
import WalletProvider from "./components/WalletProvider";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <WalletProvider>{children}</WalletProvider>
      </body>
    </html>
  );
}
```

### Use Wallet Hooks in Components

```tsx
import { useWallet, useConnection } from "@solana/wallet-adapter-react";
import { WalletMultiButton } from "@solana/wallet-adapter-react-ui";

function PaymentComponent() {
  const { publicKey, signTransaction, connected } = useWallet();
  const { connection } = useConnection();

  return (
    <div>
      <WalletMultiButton />
      {connected && <p>Connected: {publicKey?.toBase58()}</p>}
    </div>
  );
}
```

`WalletMultiButton` renders a connect/disconnect button. `useWallet()` gives you the user's `publicKey` and `signTransaction`. `useConnection()` gives you the active `Connection` for sending transactions.

## Building Payment Instructions

Use `buildAcceptPaymentInstructions` to get all the instructions needed for a payment. This is the recommended method — it automatically derives the user's token account and handles native SOL wrapping/unwrapping.

### Parameters (`BuildAcceptPaymentParams`)

| Parameter      | Type                         | Description                                       |
|----------------|------------------------------|---------------------------------------------------|
| `user`         | `PublicKey`                  | The payer's wallet address                        |
| `currencyMint` | `PublicKey`                  | Mint address of the payment currency (USDC, wSOL) |
| `amount`       | `bigint \| number \| string` | Price in the currency's smallest unit             |
| `memo`         | `bigint \| number \| string` | Unique invoice identifier (random number)         |
| `startTime`    | `bigint \| number \| string` | Unix timestamp — when the invoice becomes valid   |
| `endTime`      | `bigint \| number \| string` | Unix timestamp — when the invoice expires         |
| `tokenProgram` | `PublicKey` (optional)       | Token program for the currency (defaults to SPL Token) |

### Example

```typescript
const ixs = await agent.buildAcceptPaymentInstructions({
  user: userPublicKey,
  currencyMint,
  amount: "1000000",       // 1 USDC
  memo: "123456789",       // unique invoice identifier
  startTime: "1700000000", // valid from
  endTime: "1700086400",   // expires at
});
```

### What It Returns

- **For SPL tokens (USDC):** A single `TransactionInstruction` — the accept-payment instruction.
- **For native SOL:** Five instructions that handle wrapping/unwrapping automatically:
  1. Create the user's wrapped SOL token account (idempotent)
  2. Transfer SOL lamports into that token account
  3. Sync the native balance
  4. The accept-payment instruction
  5. Close the wrapped SOL account (returns rent back to user)

You do not need to handle SOL wrapping yourself — `buildAcceptPaymentInstructions` does it for you.

### Important

- The `amount`, `memo`, `startTime`, and `endTime` must exactly match when verifying later.
- Each unique combination of `(mint, currencyMint, amount, memo, startTime, endTime)` can only be paid once — the on-chain Invoice ID PDA prevents duplicate payments.
- Generate a unique `memo` for each invoice (e.g. `Math.floor(Math.random() * 900000000000) + 100000`).

## Full Transaction Flow — Server to Client

This is the complete flow for building a transaction on the server, signing it on the client, and sending it on-chain.

### Step 1: Generate Invoice Parameters (Server)

```typescript
function generateInvoiceParams() {
  const memo = String(Math.floor(Math.random() * 900000000000) + 100000);
  const now = Math.floor(Date.now() / 1000);
  const startTime = String(now);
  const endTime = String(now + 86400); // valid for 24 hours
  const amount = process.env.PRICE_AMOUNT || "1000000"; // e.g. 1 USDC

  return { amount, memo, startTime, endTime };
}
```

### Step 2: Build and Serialize the Transaction (Server)

```typescript
import { Connection, PublicKey, Transaction } from "@solana/web3.js";
import { PumpAgent } from "@pump-fun/agent-payments-sdk";

async function buildPaymentTransaction(params: {
  userWallet: string;
  amount: string;
  memo: string;
  startTime: string;
  endTime: string;
}) {
  const connection = new Connection(process.env.SOLANA_RPC_URL!);
  const agentMint = new PublicKey(process.env.AGENT_TOKEN_MINT_ADDRESS!);
  const currencyMint = new PublicKey(process.env.CURRENCY_MINT!);

  const agent = new PumpAgent(agentMint, "mainnet", connection);
  const userPublicKey = new PublicKey(params.userWallet);

  const ixs = await agent.buildAcceptPaymentInstructions({
    user: userPublicKey,
    currencyMint,
    amount: params.amount,
    memo: params.memo,
    startTime: params.startTime,
    endTime: params.endTime,
  });

  const transaction = new Transaction();
  for (const ix of ixs) {
    transaction.add(ix);
  }

  const { blockhash, lastValidBlockHeight } =
    await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;
  transaction.feePayer = userPublicKey;

  const serialized = transaction
    .serialize({ requireAllSignatures: false })
    .toString("base64");

  return { transaction: serialized, blockhash, lastValidBlockHeight };
}
```

The transaction is serialized without signatures so it can be sent to the client for signing.

### Step 3: Sign and Send the Transaction (Client)

```typescript
import { Transaction } from "@solana/web3.js";
import { useWallet, useConnection } from "@solana/wallet-adapter-react";

async function signAndSendPayment(serializedTx: string) {
  const { signTransaction } = useWallet();
  const { connection } = useConnection();

  const txBytes = Buffer.from(serializedTx, "base64");
  const tx = Transaction.from(txBytes);

  const signed = await signTransaction!(tx);
  const signature = await connection.sendRawTransaction(signed.serialize());
  await connection.confirmTransaction(signature, "confirmed");

  return signature;
}
```

The user's wallet (Phantom, Solflare, etc.) prompts them to approve and sign. After signing, the transaction is submitted and confirmed on-chain.

## Verify Payment

Use `validateInvoicePayment` to confirm that a specific invoice was paid on-chain.

### How It Works

1. Derives the Invoice ID PDA from `(mint, currencyMint, amount, memo, startTime, endTime)`.
2. Queries the Pump HTTP API to check if the payment event exists.
3. If the HTTP API is unavailable and a `Connection` was provided, falls back to scanning on-chain transaction logs via RPC.
4. Returns `true` if a matching payment event is found, `false` otherwise.

### Parameters

All numeric parameters must be `BN` (from `@coral-xyz/anchor`).

| Parameter      | Type        | Description                          |
|----------------|-------------|--------------------------------------|
| `user`         | `PublicKey`  | The wallet that paid                 |
| `currencyMint` | `PublicKey`  | Currency used for payment            |
| `amount`       | `BN`         | Amount paid (smallest unit)          |
| `memo`         | `BN`         | The invoice memo                     |
| `startTime`    | `BN`         | Invoice start time (Unix timestamp)  |
| `endTime`      | `BN`         | Invoice end time (Unix timestamp)    |

### Simple Backend Verification

```typescript
import { PumpAgent } from "@pump-fun/agent-payments-sdk";
import { PublicKey } from "@solana/web3.js";
import { BN } from "@coral-xyz/anchor";

const agentMint = new PublicKey(process.env.AGENT_TOKEN_MINT_ADDRESS!);
const agent = new PumpAgent(agentMint);

const paid = await agent.validateInvoicePayment({
  user: new PublicKey(userWalletAddress),
  currencyMint: new PublicKey(process.env.CURRENCY_MINT!),
  amount: new BN("1000000"),
  memo: new BN("123456789"),
  startTime: new BN("1700000000"),
  endTime: new BN("1700086400"),
});

if (paid) {
  // Payment confirmed — deliver the service
} else {
  // Payment not found
}
```

No `Connection` is needed for basic verification — it uses the HTTP API by default.

### Verification with Retries

Transactions may take a few seconds to confirm. Use a retry loop for reliability:

```typescript
async function verifyPayment(params: {
  user: string;
  currencyMint: string;
  amount: string;
  memo: string;
  startTime: string;
  endTime: string;
}): Promise<boolean> {
  const agentMint = new PublicKey(process.env.AGENT_TOKEN_MINT_ADDRESS!);
  const agent = new PumpAgent(agentMint);

  const invoiceParams = {
    user: new PublicKey(params.user),
    currencyMint: new PublicKey(params.currencyMint),
    amount: new BN(params.amount),
    memo: new BN(params.memo),
    startTime: new BN(params.startTime),
    endTime: new BN(params.endTime),
  };

  for (let attempt = 0; attempt < 5; attempt++) {
    const verified = await agent.validateInvoicePayment(invoiceParams);
    if (verified) return true;
    await new Promise((r) => setTimeout(r, 3000));
  }

  return false;
}
```

### Tip: Converting from Simple Types to BN

If you have `string` or `number` values from your invoice params, convert them to `BN` before calling `validateInvoicePayment`:

```typescript
import { BN } from "@coral-xyz/anchor";

const amount = new BN("1000000");
const memo = new BN("123456789");
const startTime = new BN("1700000000");
const endTime = new BN("1700086400");
```

## End-to-End Flow

```
1. Agent decides on price → generates unique memo → sets time window
       ↓
2. Server: buildAcceptPaymentInstructions(...) → returns TransactionInstruction[]
       ↓
3. Server: wraps in Transaction, sets blockhash + feePayer, serializes to base64
       ↓
4. Client: deserializes, user signs with wallet adapter
       ↓
5. Client: sendRawTransaction → confirmTransaction
       ↓
6. Server: validateInvoicePayment(...) → returns true/false
       ↓
7. Agent delivers the service (or asks user to retry)
```

---

## Scenario Tests

### Scenario 1: Happy Path — Pay and Verify

1. Agent generates invoice: amount `1000000` (1 USDC), memo `42`, startTime `1700000000`, endTime `1700086400`.
2. Server calls `buildAcceptPaymentInstructions` and serializes the transaction.
3. Client signs and submits the transaction on Solana.
4. Server calls `validateInvoicePayment` with the same params.
5. Returns `true`. Agent delivers the service.

### Scenario 2: Verify Before Payment

1. Agent generates invoice: amount `500000`, memo `7777`, valid for 1 hour.
2. Server immediately calls `validateInvoicePayment` (user hasn't paid yet).
3. Returns `false`.
4. Agent tells the user payment is not confirmed and to try again after paying.

### Scenario 3: Duplicate Payment Rejection

1. Agent generates invoice with memo `99`.
2. User pays successfully. `validateInvoicePayment` returns `true`.
3. A second attempt to submit the same `acceptPayment` transaction is rejected by the on-chain program because the Invoice ID PDA is already initialized.

### Scenario 4: Mismatched Parameters

1. Agent generates invoice: amount `1000000`, memo `555`.
2. User pays with a different amount (`2000000`) but same memo.
3. `validateInvoicePayment` with original params returns `false` — the on-chain event has a different amount.

### Scenario 5: Expired Invoice

1. Agent generates invoice with `endTime` in the past.
2. The on-chain program rejects the transaction (timestamp outside validity window).
3. Agent should generate a new invoice with a valid time window.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `validateInvoicePayment` returns `false` but user claims they paid | Transaction may still be confirming, or params don't match | Wait a few seconds and retry. Double-check that `amount`, `memo`, `startTime`, `endTime`, and `user` all match exactly. |
| Invoice already paid | The Invoice ID PDA is already initialized | Generate a new invoice with a different `memo`. |
| Insufficient balance | User's token account doesn't have enough tokens | Tell the user to fund their wallet before paying. |
| Currency not supported | The `currencyMint` is not in the protocol's `GlobalConfig` | Use a supported currency (USDC, wSOL). |
