# Lesson 4: Fees, Rent & Compute Units — The Economics of Solana

## Why This Matters

Every time you create an account, send a transaction, or store data on Solana,
it costs something. Understanding the fee structure prevents surprises and 
lets you optimize your dApps.

---

## Part 1: Transaction Fees

### The Base Fee

Every Solana transaction costs a **base fee** of **5,000 lamports per signature**.

```
1 SOL = 1,000,000,000 lamports (1 billion)
Base fee = 5,000 lamports = 0.000005 SOL

At SOL price of $150:
  Fee = $0.00075 per transaction
  That's less than 1/10th of a cent

Compare:
  Ethereum: $2 - $50 per transaction (varies wildly with gas)
  Bitcoin:  $1 - $30 per transaction
  Solana:   $0.00075 per transaction (nearly free)
```

### Where Do Fees Go?

```
Transaction fee breakdown:
  50% → BURNED (removed from circulation forever)
  50% → Goes to the Leader validator who processed the block

This burning mechanism is deflationary pressure on SOL supply.
```

### Priority Fees (Congestion Pricing)

During high demand (meme coin launches, NFT mints), the network gets congested.
Priority fees let you pay extra to get processed faster:

```javascript
import { ComputeBudgetProgram } from "@solana/web3.js";

// Add a priority fee to your transaction
const priorityFeeIx = ComputeBudgetProgram.setComputeUnitPrice({
  microLamports: 50000,  // price per compute unit in micro-lamports
});

// 1 micro-lamport = 0.000001 lamports
// If your tx uses 200,000 compute units:
// Priority fee = 200,000 × 50,000 / 1,000,000 = 10,000 lamports
// Total fee = 5,000 (base) + 10,000 (priority) = 15,000 lamports
```

### How to Estimate Priority Fees

```javascript
// Get recent priority fee data from the network
const recentFees = await connection.getRecentPrioritizationFees();

// This returns an array of recent fee data:
// [
//   { slot: 250000001, prioritizationFee: 0 },
//   { slot: 250000002, prioritizationFee: 10000 },
//   { slot: 250000003, prioritizationFee: 50000 },
//   ...
// ]

// Strategy: Use the median or 75th percentile of recent fees
// During calm periods: 0-1000 micro-lamports
// During congestion: 10,000-1,000,000 micro-lamports
// During extreme congestion (meme coins): 1,000,000+ micro-lamports
```

---

## Part 2: Rent — Paying for Storage

### The Problem: Blockchain Bloat

Every account on Solana takes up space on validators' SSDs.
If creating accounts were free, people would spam billions of 
empty accounts and bloat the blockchain.

**Rent** is Solana's solution: you must deposit SOL to keep an 
account alive. Think of it as a security deposit.

### Rent-Exempt: The Standard Approach

In practice, almost ALL accounts on Solana are **rent-exempt**.
This means you deposit enough SOL upfront that the account can
exist forever without paying ongoing rent.

```
The rent-exempt threshold:
  ~0.00089088 SOL per byte of data per epoch (roughly)

  In practice, the minimum rent-exempt balance is calculated as:
  rent = data_size_in_bytes × 6.96 lamports/byte × 2 years worth

Formula (simplified):
  rent_exempt_minimum = (128 + data_size) × 6,960 lamports

Examples:
  Empty account (0 bytes data):     890,880 lamports (~0.00089 SOL)
  Token Account (165 bytes data):   2,039,280 lamports (~0.002 SOL)
  Mint Account (82 bytes data):     1,461,600 lamports (~0.0015 SOL)
  Custom 100-byte account:          1,586,880 lamports (~0.0016 SOL)
```

### Getting the Exact Rent Cost

```javascript
// Calculate rent-exempt minimum from the RPC
const rentExemptBalance = await connection.getMinimumBalanceForRentExemption(
  165  // size in bytes (165 = token account)
);

console.log(`Rent for token account: ${rentExemptBalance} lamports`);
// Output: "Rent for token account: 2039280 lamports"

console.log(`That's ${rentExemptBalance / LAMPORTS_PER_SOL} SOL`);
// Output: "That's 0.00203928 SOL"
```

### What If You Don't Pay Rent?

If an account's lamport balance falls below the rent-exempt threshold:

```
Old behavior (deprecated):
  The account would be slowly drained each epoch until it hits 0,
  then it gets garbage collected (deleted from the network).

Current behavior:
  The runtime enforces rent-exemption at account creation.
  You MUST deposit the full rent-exempt amount when creating an account.
  If you try to create an account without enough lamports → transaction fails.

  So in practice, you never deal with "paying rent" — you just pay
  the deposit upfront and the account lives forever.
```

### Reclaiming Rent (Closing Accounts)

When you no longer need an account, you can **close it** to get 
the rent SOL back:

```javascript
// Closing a token account returns the rent to you
import { createCloseAccountInstruction } from "@solana/spl-token";

const closeIx = createCloseAccountInstruction(
  tokenAccountAddress,    // the account to close
  owner.publicKey,        // where to send the reclaimed lamports
  owner.publicKey,        // the authority who can close it
);

// After this executes:
// - The token account is deleted from the network
// - The rent SOL (~0.002 SOL) is returned to your wallet
// - You can't use this token account anymore
```

**Pro tip**: Many wallets (like Phantom) have a "close empty accounts" 
feature that finds all your empty token accounts and closes them to 
reclaim rent. If you have 100 empty token accounts, that's ~0.2 SOL 
sitting there doing nothing!

---

## Part 3: Compute Units (CUs) — Solana's Gas

### What Are Compute Units?

Compute Units are Solana's equivalent of Ethereum's gas. They measure 
how much computation a transaction uses.

```
Every instruction in a transaction consumes compute units:
  - Simple math operations: 1 CU
  - SHA256 hash: ~200 CUs  
  - Ed25519 signature verify: ~25,000 CUs
  - Reading account data: varies
  - Writing account data: varies
  - Cross-program invocation (CPI): ~1,000 CUs overhead
```

### Default Limits

```
Per instruction:  200,000 CU (default)
Per transaction:  1,400,000 CU (max across all instructions)

If your transaction exceeds the compute budget → it fails with:
"Program failed to complete: exceeded maximum number of instructions"
```

### Setting Compute Budget

```javascript
import { ComputeBudgetProgram } from "@solana/web3.js";

// Increase compute limit (if your transaction needs more)
const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
  units: 400000,  // request 400,000 CU (instead of default 200,000)
});

// Decrease compute limit (to save on priority fees!)
const computeLimitIx = ComputeBudgetProgram.setComputeUnitLimit({
  units: 50000,  // only need 50,000 CU for a simple transfer
});

// Remember: priority fee = microLamports × compute units
// Lower compute limit = lower total priority fee = cheaper transaction!
```

### Why CU Optimization Matters

```
Scenario: Jupiter swap (complex DeFi transaction)

Unoptimized:
  Compute limit: 200,000 CU (default)
  Priority fee rate: 100,000 micro-lamports per CU
  Priority fee = 200,000 × 100,000 / 1,000,000 = 20,000 lamports

Optimized:
  Actual CU used: 80,000 CU
  Set compute limit: 100,000 CU (with some buffer)
  Priority fee rate: 100,000 micro-lamports per CU
  Priority fee = 100,000 × 100,000 / 1,000,000 = 10,000 lamports

  Saved: 50% on priority fees just by setting an accurate CU limit!
```

### Common CU Costs

| Operation | Approximate CU Cost |
|-----------|---------------------|
| SOL transfer (SystemProgram) | ~450 CU |
| SPL token transfer | ~4,000 CU |
| Create token account | ~5,000 CU |
| Token swap (Jupiter) | ~50,000 - 150,000 CU |
| NFT mint (Metaplex) | ~100,000 - 200,000 CU |
| Complex DeFi interaction | ~200,000 - 800,000 CU |

### Simulating to Find CU Usage

```javascript
// Simulate a transaction to find exact CU usage BEFORE sending
const simulation = await connection.simulateTransaction(transaction);

console.log("Compute units consumed:", simulation.value.unitsConsumed);
// Output: "Compute units consumed: 45231"

// Now set your compute limit to unitsConsumed + 10% buffer
const safeLimit = Math.ceil(simulation.value.unitsConsumed * 1.1);
```

---

## Part 4: Cost Breakdown for Common Operations

Here's a practical reference for how much things cost on Solana:

```
┌─────────────────────────────────────────────────────────────┐
│  Operation                        │  Cost (approx.)         │
├───────────────────────────────────┼─────────────────────────┤
│  Send SOL                        │  ~0.000005 SOL (fee)     │
│  Send SPL Token (ATA exists)     │  ~0.000005 SOL (fee)     │
│  Send SPL Token (create ATA)     │  ~0.002 SOL (rent+fee)   │
│  Create a new token mint         │  ~0.0015 SOL (rent)      │
│  Mint an NFT (Metaplex)          │  ~0.015 SOL (rent+fee)   │
│  Mint a cNFT (compressed)        │  ~0.000005 SOL (!!!)     │
│  Jupiter swap                    │  ~0.000005 SOL (fee)     │
│  Deploy a program (100KB)        │  ~0.7 SOL (rent)         │
│  Close an empty token account    │  +0.002 SOL (reclaimed!) │
└───────────────────────────────────┴─────────────────────────┘
```

---

## Lamports Quick Reference

```
1 SOL = 1,000,000,000 lamports (10^9)

Named after Leslie Lamport, the computer scientist whose work
on distributed systems influenced Solana's design.

Common values:
  0.000000001 SOL =           1 lamport
  0.000005    SOL =       5,000 lamports (base tx fee)
  0.001       SOL =   1,000,000 lamports
  0.01        SOL =  10,000,000 lamports  
  0.1         SOL = 100,000,000 lamports
  1           SOL = 1,000,000,000 lamports
```

---

## Summary

| Concept | Key Details |
|---------|-------------|
| Base fee | 5,000 lamports/signature (~$0.00075) — 50% burned, 50% to leader |
| Priority fee | Optional, per compute unit, for faster processing during congestion |
| Rent | Deposit SOL to keep accounts alive; ~0.002 SOL for a token account |
| Rent-exempt | Standard approach: deposit enough upfront, account lives forever |
| Closing accounts | Delete + reclaim rent SOL |
| Compute Units | Measure of computation; 200K default per instruction, 1.4M max per tx |
| CU optimization | Set accurate limits to reduce priority fee costs |
| Simulation | Test CU usage before sending to optimize costs |
