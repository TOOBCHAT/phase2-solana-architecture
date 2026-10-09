# Lesson 3: Transactions & Instructions — How Things Actually Happen on Solana

## The Anatomy of a Solana Transaction

A transaction is the ONLY way to change state on Solana. Every time you 
send SOL, mint an NFT, swap tokens, or vote — it's a transaction.

---

## The Structure

A Solana transaction has three parts:

```
┌────────────────────────────────────────────────────────┐
│                  SOLANA TRANSACTION                     │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. SIGNATURES (array)                                 │
│     - Ed25519 signatures from required signers         │
│     - At minimum: the fee payer's signature            │
│     - Can have multiple signers                        │
│                                                        │
│  2. MESSAGE                                            │
│     │                                                  │
│     ├── Header                                         │
│     │   - num_required_signatures                      │
│     │   - num_readonly_signed_accounts                 │
│     │   - num_readonly_unsigned_accounts               │
│     │                                                  │
│     ├── Account Keys (array of PublicKeys)             │
│     │   - Every account the transaction touches        │
│     │   - Ordered: signers first, then non-signers     │
│     │   - Marked as writable or read-only              │
│     │                                                  │
│     ├── Recent Blockhash                               │
│     │   - For replay protection (150 block window)     │
│     │   - Transaction expires if blockhash is too old  │
│     │                                                  │
│     └── Instructions (array)                           │
│         - The actual operations to perform             │
│         - Can have MULTIPLE instructions in one tx     │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Instructions: The Building Blocks

An Instruction is a single operation. A transaction can contain
**multiple instructions** that execute **atomically** — either ALL
succeed, or ALL fail (no partial execution).

Each instruction has three fields:

```
┌────────────────────────────────────────┐
│            INSTRUCTION                  │
├────────────────────────────────────────┤
│                                        │
│  program_id:  Which program to invoke  │
│               (the smart contract)     │
│                                        │
│  accounts:    Array of accounts this   │
│               instruction needs        │
│               Each marked as:          │
│               - isSigner (true/false)  │
│               - isWritable (true/false)│
│                                        │
│  data:        Encoded instruction data │
│               (which function to call  │
│                + parameters)           │
│                                        │
└────────────────────────────────────────┘
```

### Example: Transferring SOL

```javascript
import {
  Connection,
  Keypair,
  Transaction,
  SystemProgram,
  sendAndConfirmTransaction,
  LAMPORTS_PER_SOL,
} from "@solana/web3.js";

// Connect to devnet
const connection = new Connection("https://api.devnet.solana.com", "confirmed");

// The sender (has private key to sign)
const sender = Keypair.generate();

// The recipient (just a public key)
const recipient = new Keypair().publicKey;

// Build the instruction
const transferInstruction = SystemProgram.transfer({
  fromPubkey: sender.publicKey,   // must be a signer
  toPubkey: recipient,            // just needs to be writable
  lamports: 0.5 * LAMPORTS_PER_SOL,  // 0.5 SOL
});

// Under the hood, this instruction looks like:
// {
//   programId: "11111111111111111111111111111111" (System Program),
//   keys: [
//     { pubkey: sender.publicKey, isSigner: true, isWritable: true },
//     { pubkey: recipient, isSigner: false, isWritable: true },
//   ],
//   data: Buffer.from([2, 0, 0, 0, ...])  // encoded: "transfer" + amount
// }

// Wrap instruction(s) in a transaction
const transaction = new Transaction().add(transferInstruction);

// Sign and send
const signature = await sendAndConfirmTransaction(
  connection,
  transaction,
  [sender]  // array of signers
);

console.log("Transaction signature:", signature);
// Output: something like "5xGt7...abc123"
// You can view this on: https://explorer.solana.com/tx/<signature>?cluster=devnet
```

---

## Multiple Instructions in One Transaction

This is one of Solana's superpowers. You can batch multiple operations
into a single atomic transaction:

```javascript
// Example: Create a token account AND transfer tokens in ONE transaction

const transaction = new Transaction();

// Instruction 1: Create an Associated Token Account for the recipient
transaction.add(
  createAssociatedTokenAccountInstruction(
    payer.publicKey,        // who pays for account creation
    recipientATA,           // the new token account address
    recipient.publicKey,    // the wallet that will own the tokens
    mintAddress,            // which token mint
  )
);

// Instruction 2: Transfer tokens to the newly created account
transaction.add(
  createTransferInstruction(
    senderATA,              // source token account
    recipientATA,           // destination token account (just created above!)
    sender.publicKey,       // authority (signer)
    1_000_000,              // amount (in smallest units)
  )
);

// Both instructions execute atomically:
// - If the ATA creation fails → the transfer also fails (rolled back)
// - If the transfer fails → the ATA creation also fails (rolled back)
// - Either BOTH succeed or NEITHER succeeds
const sig = await sendAndConfirmTransaction(connection, transaction, [sender]);
```

### Why Atomicity Matters

```
Scenario WITHOUT atomicity (dangerous):
  Step 1: Create recipient's token account ✅
  Step 2: Transfer tokens to recipient      ❌ (fails for some reason)
  Result: Account exists but is empty. You wasted rent SOL.

Scenario WITH atomicity (safe):
  Step 1: Create recipient's token account
  Step 2: Transfer tokens to recipient      ❌ (fails)
  Result: EVERYTHING rolls back. No account created. No SOL wasted.
```

---

## Transaction Size Limits

Solana transactions have a hard size limit:

```
Maximum transaction size: 1,232 bytes

This must fit:
  - All signatures (~64 bytes each)
  - All account keys (~32 bytes each)
  - The recent blockhash (32 bytes)
  - All instruction data
  - Header bytes

Practical limits:
  - ~20-30 accounts per transaction (depending on instruction data size)
  - ~10-12 instructions per transaction (rough estimate)
```

### When You Hit the Limit: Lookup Tables (ALTs)

If your transaction needs more accounts than can fit in 1,232 bytes,
Solana has **Address Lookup Tables (ALTs)**:

```
Without Lookup Table:
  Account keys: [full 32-byte address, full 32-byte address, ...]
  → 10 accounts = 320 bytes just for addresses!

With Lookup Table:
  1. Pre-store frequently used addresses in an on-chain lookup table
  2. In your transaction, reference them by INDEX (1 byte each!)
  Account keys: [index 0, index 1, index 2, ...]
  → 10 accounts = 10 bytes!
  → Fits WAY more accounts in a single transaction
```

This requires using **Versioned Transactions** (v0) instead of Legacy transactions.
We'll use these in Phase 3.

---

## Signers and Authority

### Who Needs to Sign?

```
A transaction requires signatures from:

1. The FEE PAYER (always required)
   - The first signer in the signers array
   - Pays the transaction fee (~0.000005 SOL / 5000 lamports)

2. Any account marked as `isSigner: true` in any instruction
   - If you're transferring SOL: the sender must sign
   - If you're transferring tokens: the token authority must sign
   - If you're closing an account: the authority must sign

3. Multiple signers are possible
   - Multisig wallets
   - Transactions that need approval from multiple parties
```

### Example: Multiple Signers

```javascript
// An escrow where both parties must agree

const transaction = new Transaction();

transaction.add(
  escrowInstruction({
    escrowAccount: escrowPDA,
    partyA: alice.publicKey,    // isSigner: true
    partyB: bob.publicKey,      // isSigner: true
    tokenAccountA: aliceATA,
    tokenAccountB: bobATA,
  })
);

// Both Alice AND Bob must sign
const sig = await sendAndConfirmTransaction(
  connection,
  transaction,
  [alice, bob]  // both signers required
);
```

---

## Cross-Program Invocations (CPIs)

Programs can call other programs. This is called a **CPI**.

```
Example: Your custom staking program needs to transfer tokens.
It doesn't implement token transfer logic — it calls the Token Program.

User's Transaction:
  └── Instruction: "stake 100 tokens" → Your Staking Program
        │
        └── CPI: "transfer 100 tokens" → Token Program
              │
              └── Modifies: user's token account (-100)
              └── Modifies: vault's token account (+100)
```

### CPI Rules

1. A program can invoke another program via CPI
2. The called program has the same restrictions as a top-level instruction
3. **PDA signing**: A program can "sign" for its PDAs in a CPI
   (this is the ONLY way a PDA can act as a signer)
4. CPI depth limit: 4 levels deep (A calls B calls C calls D)
5. Compute budget is shared across the entire transaction

```
CPI call chain:
  Your Program → Token Program → (no further CPIs)

PDA Signing example:
  Your Program creates a vault PDA
  Your Program invokes Token Program with CPI
  Your Program signs for the vault PDA (because it derived the PDA)
  Token Program sees a valid signature and transfers tokens from the vault
```

---

## Transaction Lifecycle: What Happens After You Send

```
Step 1: BUILD
  Your app builds a Transaction object with instructions

Step 2: GET BLOCKHASH
  Fetch a recent blockhash from the network
  const { blockhash } = await connection.getLatestBlockhash();
  transaction.recentBlockhash = blockhash;

Step 3: SIGN
  Sign the transaction with required private keys
  transaction.sign(signer1, signer2);

Step 4: SEND
  Send the serialized transaction to an RPC node
  const sig = await connection.sendTransaction(transaction, [signer]);

Step 5: SIMULATE (preflight)
  The RPC node simulates the transaction locally
  If it would fail → returns error immediately (saves you fees)

Step 6: FORWARD (Gulf Stream)
  RPC forwards the tx to the current/next leader validator

Step 7: PROCESS
  Leader inserts tx into PoH stream
  Leader executes all instructions sequentially
  If any instruction fails → entire tx reverts

Step 8: CONFIRM
  Block is propagated via Turbine
  Validators vote on the block (Tower BFT)
  
  Confirmation levels:
    "processed"   = leader has processed it (fastest, least safe)
    "confirmed"   = 66% of stake has voted on this block (default)
    "finalized"   = 31+ consecutive confirmations (~12 sec, safest)

Step 9: RESULT
  Your app receives the transaction signature
  Can query: await connection.getTransaction(signature);
```

### Confirmation Levels Explained

```javascript
// When sending a transaction, you can specify commitment level:

const connection = new Connection(rpcUrl, "confirmed");
// "processed"  → acknowledged by leader, but could be rolled back
// "confirmed"  → voted on by supermajority, very safe, DEFAULT
// "finalized"  → impossible to roll back, but takes ~12 seconds

// For most dApps: "confirmed" is the sweet spot
// For exchanges/high-value: use "finalized"
// For displaying real-time updates: "processed" is fine
```

---

## Transaction Fees

Solana fees are very simple and cheap:

```
Base fee: 5,000 lamports per signature (0.000005 SOL)
          At $150/SOL that's $0.00075 per transaction

Priority fee: Optional, paid per compute unit
              Used during congestion to get your tx processed faster

Total fee = base fee + priority fee
```

### Priority Fees (Compute Unit Pricing)

```javascript
import { ComputeBudgetProgram } from "@solana/web3.js";

const transaction = new Transaction();

// Set a priority fee: pay extra to get processed faster
transaction.add(
  ComputeBudgetProgram.setComputeUnitPrice({
    microLamports: 50000,  // price per compute unit
  })
);

// Optionally set compute limit (default is 200,000 CU per instruction)
transaction.add(
  ComputeBudgetProgram.setComputeUnitLimit({
    units: 100000,  // lower = cheaper if your tx doesn't need much compute
  })
);

// Add your actual instruction
transaction.add(transferInstruction);
```

---

## Versioned Transactions (v0 vs Legacy)

Solana has two transaction formats:

```
Legacy Transactions (v0-legacy):
  - Original format
  - All account addresses stored directly in the transaction
  - Simple to build
  - Limited by 1,232 byte max

Versioned Transactions (v0):
  - New format
  - Supports Address Lookup Tables (ALTs)
  - Can reference more accounts
  - Required for complex DeFi operations
```

```javascript
import { VersionedTransaction, TransactionMessage } from "@solana/web3.js";

// Building a Versioned Transaction
const messageV0 = new TransactionMessage({
  payerKey: payer.publicKey,
  recentBlockhash: blockhash,
  instructions: [transferInstruction],
}).compileToV0Message(
  [lookupTableAccount]  // optional: address lookup tables
);

const versionedTx = new VersionedTransaction(messageV0);
versionedTx.sign([payer]);

const sig = await connection.sendTransaction(versionedTx);
```

---

## Real-World Transaction Inspection

Here's what a real Solana transaction looks like when you inspect it
on Solana Explorer (explorer.solana.com):

```
Transaction: 5xGt7...abc123

Status: Success ✅
Block: 250,123,456
Timestamp: June 20, 2026 22:30:00

Fee: 0.000005 SOL (5,000 lamports)
Compute Units Used: 12,345 / 200,000

Signers:
  1. 7xKXtg2CW87d97TXJSDpbD5jBkheTqA... (fee payer)

Instructions:
  #1 — System Program: Transfer
      From: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA...
      To:   9aBcDeFgHiJkLmNoPqRsTuVwXyZ12345...
      Amount: 500,000,000 lamports (0.5 SOL)

Account Inputs:
  #0: 7xKXtg... [SIGNER, WRITABLE] — sender
  #1: 9aBcDe... [WRITABLE] — recipient
  #2: 111111... [PROGRAM] — System Program

Log Messages:
  Program 11111111111111111111111111111111 invoke [1]
  Program 11111111111111111111111111111111 success
```

---

## Summary

| Concept | Details |
|---------|---------|
| Transaction | Container for 1+ instructions, executed atomically |
| Instruction | Single operation: which program, which accounts, what data |
| Signatures | Ed25519, from fee payer + all required signers |
| Recent Blockhash | Replay protection, valid for ~150 slots (~90 sec) |
| Atomicity | All instructions succeed OR all fail — no partial execution |
| Transaction limit | 1,232 bytes max |
| Lookup Tables | Reference accounts by index to fit more in one tx |
| Versioned Tx (v0) | New format supporting lookup tables |
| CPI | Programs calling other programs |
| PDA Signing | Programs can sign for their PDAs during CPIs |
| Confirmation | processed → confirmed → finalized |
| Base fee | 5,000 lamports (~$0.0008) per signature |
| Priority fee | Optional, per compute unit, for faster processing |
