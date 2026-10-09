# Lesson 2: The Solana Account Model — Everything is an Account

## Why This Lesson Matters Most

If you only deeply understand ONE thing about Solana, it should be the Account Model.
Every single concept in Solana development — tokens, NFTs, DeFi, programs — builds on this.

The Account Model is what makes Solana fundamentally different from Ethereum, and
understanding it early will save you countless hours of confusion later.

---

## Ethereum's Model (What You Might Know)

In Ethereum, a smart contract is a self-contained unit:

```
┌─────────────────────────────────────┐
│        ERC-20 Token Contract        │
│                                     │
│  Code:                              │
│    function transfer(to, amount)    │
│    function balanceOf(address)      │
│    function approve(spender, amt)   │
│                                     │
│  Storage (State):                   │
│    balances["Alice"] = 500          │
│    balances["Bob"]   = 300          │
│    balances["Charlie"] = 200        │
│    totalSupply = 1000               │
│    allowances["Alice"]["Bob"] = 50  │
│                                     │
└─────────────────────────────────────┘
```

**Key point**: The code AND the data live INSIDE the same contract.
When Alice transfers tokens to Bob, the contract modifies its own internal
`balances` mapping. The contract is both the program AND the database.

---

## Solana's Model: Code and Data Are Separated

On Solana, **programs (smart contracts) are stateless**. They contain ONLY
executable code — no storage, no variables, no state.

All data lives in separate **Accounts**.

```
┌──────────────────────┐     ┌──────────────────────┐
│   Token Program      │     │   Alice's Token       │
│   (Executable)       │     │   Account (Data)      │
│                      │     │                       │
│   Code:              │     │   owner: Token Program│
│     transfer()       │     │   mint: USDC_MINT     │
│     mint_to()        │     │   balance: 500        │
│     burn()           │     │   delegate: null      │
│                      │     │                       │
│   Storage: NONE      │     └──────────────────────┘
│                      │
│   This program is    │     ┌──────────────────────┐
│   READ-ONLY. It      │     │   Bob's Token         │
│   cannot store any   │     │   Account (Data)      │
│   data inside itself.│     │                       │
│                      │     │   owner: Token Program│
└──────────────────────┘     │   mint: USDC_MINT     │
                             │   balance: 300        │
                             │   delegate: null      │
                             └──────────────────────┘
```

When Alice transfers 100 USDC to Bob:
1. The **Token Program** is invoked
2. It receives Alice's account and Bob's account as inputs
3. It reads Alice's balance (500), subtracts 100 → writes 400 back
4. It reads Bob's balance (300), adds 100 → writes 400 back
5. The Token Program itself didn't store anything — it just modified external accounts

---

## What Exactly IS an Account?

Every account on Solana is a struct with these fields:

```
┌─────────────────────────────────────────────────────┐
│                   Solana Account                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  pubkey:      The account's address (32 bytes)       │
│               Like: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA │
│                                                      │
│  lamports:    The SOL balance (in lamports)           │
│               1 SOL = 1,000,000,000 lamports         │
│                                                      │
│  data:        A byte array [] of arbitrary data      │
│               Can be empty, or can hold structured   │
│               data (token balances, NFT metadata,    │
│               program state, etc.)                   │
│                                                      │
│  owner:       The program that OWNS this account     │
│               Only the owner program can modify the  │
│               data[] field. This is critical!         │
│                                                      │
│  executable:  Boolean — is this account a program?   │
│               true = this account contains code      │
│               false = this account contains data     │
│                                                      │
│  rent_epoch:  The epoch at which rent was last paid   │
│               (more on rent in Lesson 4)             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### The JavaScript Equivalent (for intuition)

```javascript
// Every account on Solana looks like this under the hood
const account = {
  pubkey: "7xKXtg2CW87d97TXJSDpbD5jBkheTqA...",  // unique address
  lamports: 1000000000,                            // 1 SOL
  data: Buffer.from([...]),                        // raw bytes
  owner: "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",  // who controls this
  executable: false,                               // data account, not a program
  rentEpoch: 361,
};
```

---

## The Three Types of Accounts

### 1. Wallet Accounts (System Accounts)

Your wallet address is an account. When you create a Phantom wallet, 
you're creating a keypair, and the public key becomes your account address.

```
┌──────────────────────────────────┐
│  Wallet Account                  │
│                                  │
│  pubkey:  Your wallet address    │
│  lamports: 2_000_000_000 (2 SOL)│
│  data:    [] (empty!)            │
│  owner:   System Program         │
│  executable: false               │
│                                  │
│  Key facts:                      │
│  - Owned by the System Program   │
│  - data[] is EMPTY               │
│  - Only holds SOL (lamports)     │
│  - You control it because you    │
│    have the private key          │
└──────────────────────────────────┘
```

**Important**: Your wallet doesn't "contain" your tokens or NFTs.
It's just an address with a SOL balance. Your tokens are in 
separate Token Accounts (explained below).

### 2. Program Accounts (Executable Accounts)

Programs (smart contracts) are accounts with `executable: true`.

```
┌──────────────────────────────────┐
│  Program Account                 │
│                                  │
│  pubkey:  Program ID (address)   │
│  lamports: enough for rent       │
│  data:    [compiled BPF bytecode]│
│  owner:   BPF Loader Program     │
│  executable: true                │
│                                  │
│  Key facts:                      │
│  - executable = true             │
│  - data[] contains the compiled  │
│    program code (not user data!) │
│  - Owned by the BPF Loader       │
│  - IMMUTABLE by default (but can │
│    be made upgradeable)          │
│  - Cannot store state/variables  │
└──────────────────────────────────┘
```

Every program you've heard of is just an account:
- **System Program**: `11111111111111111111111111111111`
- **Token Program**: `TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`
- **Token-2022 Program**: `TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`
- **Metaplex Token Metadata**: `metaqbxxUerdq28cj1RbAWkYQm3ybzjb6a8bt518x1s`

### 3. Data Accounts (State Accounts)

These hold the actual data/state for programs. They are the "external USB drives."

```
┌──────────────────────────────────┐
│  Data Account (Token Account)    │
│                                  │
│  pubkey:  Some derived address   │
│  lamports: enough for rent       │
│  data: [                         │
│    mint: USDC mint address,      │
│    owner: Alice's wallet,        │
│    amount: 500_000_000,          │
│    delegate: null,               │
│    state: Initialized,           │
│  ]                               │
│  owner:  Token Program           │
│  executable: false               │
│                                  │
│  Key facts:                      │
│  - executable = false            │
│  - data[] contains structured    │
│    state (serialized with borsh) │
│  - Owned by the PROGRAM, not     │
│    the user!                     │
└──────────────────────────────────┘
```

---

## The Owner Field: The Most Important Concept

The `owner` field determines **which program has permission to modify 
the account's `data[]` field**.

### The Rule

> **Only the owner program can modify an account's data.
> Any other program that tries to modify it will be rejected by the runtime.**

```
Example:

Alice's Token Account:
  owner: Token Program

This means:
  ✅ Token Program CAN modify this account's data (transfer tokens, etc.)
  ❌ System Program CANNOT modify this account's data
  ❌ Your custom program CANNOT modify this account's data
  ❌ Any other program CANNOT modify this account's data

BUT:
  ✅ Any program CAN read this account's data
  ✅ Anyone (any program) CAN add lamports to this account
  ✅ Only the owner program CAN deduct lamports from this account
```

### Don't Confuse Two Different "Owners"

There are TWO different concepts of "owner" in Solana. This confuses everyone:

```
1. Account Owner (the `owner` field on the account)
   → This is a PROGRAM that controls the account's data
   → Example: Token Program owns all token accounts

2. Token Account Owner (a field INSIDE the token account's data)
   → This is a WALLET that is authorized to use the tokens
   → Example: Alice's wallet address is stored inside her token account

They are completely different things!
```

Visual:
```
┌──────────────────────────────────────────┐
│  Alice's USDC Token Account              │
│                                          │
│  Account-level owner: Token Program ◄──── This program controls the account
│                                          │
│  data: {                                 │
│    token_owner: Alice's Wallet  ◄──────── This wallet can spend the tokens
│    mint: USDC mint address               │
│    amount: 500 USDC                      │
│  }                                       │
└──────────────────────────────────────────┘

When Alice sends USDC:
1. Alice signs the transaction (proves she's the token_owner inside data)
2. Token Program is invoked (it's the account-level owner)
3. Token Program checks: "Is the signer the token_owner in this account? Yes."
4. Token Program modifies the data: deducts from Alice, credits Bob
```

---

## Program Derived Addresses (PDAs) — Programs Creating Accounts

Regular accounts are created from keypairs (you generate a public/private key).
But what if a PROGRAM needs to own an account? Programs don't have private keys!

This is where **PDAs** come in.

### What is a PDA?

A PDA is an account address that is **derived deterministically** from:
1. A set of **seeds** (arbitrary bytes — like a string, a pubkey, a number)
2. The **program ID** that's deriving it

```javascript
// Conceptually:
const [pdaAddress, bump] = PublicKey.findProgramAddressSync(
  [
    Buffer.from("user-stats"),      // seed 1: a label
    userWallet.toBuffer(),           // seed 2: the user's wallet
  ],
  programId                          // the program that will own this PDA
);
```

### The Critical Property of PDAs

PDAs are addresses that **do NOT lie on the Ed25519 elliptic curve**.
This means:
- **No private key exists for a PDA**
- No human can sign transactions as a PDA
- ONLY the program that derived the PDA can "sign" for it (via CPI)

### Why PDAs Are Everywhere

PDAs solve the problem: "How does a program create and manage accounts 
that only IT can control?"

```
Use cases:
  - A staking program creates a vault PDA to hold staked SOL
    (only the staking program can withdraw from the vault)
  
  - A token program creates an ATA (Associated Token Account) for each user
    (derived from the user's wallet + the token mint)
  
  - A game stores each player's stats in a PDA
    (derived from "player-stats" + player's wallet)
  
  - An escrow program holds funds in a PDA until conditions are met
    (only the escrow program can release funds)
```

### The Bump Seed

When deriving a PDA, sometimes the initial derivation lands ON the elliptic curve
(which would mean a private key exists — that's not what we want).
The **bump** is a number (0-255) that is subtracted until the address falls OFF 
the curve. The runtime finds the first valid bump automatically.

```
Derivation attempt with bump=255: lands on curve ❌
Derivation attempt with bump=254: lands on curve ❌
Derivation attempt with bump=253: lands OFF curve ✅ → this is the PDA!

The bump=253 is called the "canonical bump" and should always be used.
```

---

## Accounts in Practice: A Counter dApp Example

Let's say you want to build a simple counter dApp. Here's what the 
account structure looks like:

```
┌─────────────────────────┐
│  Counter Program         │
│  (Executable Account)    │
│                          │
│  executable: true        │
│  owner: BPF Loader       │
│  data: [program bytecode]│
│                          │
│  Instructions:           │
│    - initialize()        │
│    - increment()         │
│    - decrement()         │
└────────────┬────────────┘
             │ modifies
             ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  Counter Data Account    │     │  User's Wallet           │
│  (PDA)                   │     │  (System Account)        │
│                          │     │                          │
│  executable: false       │     │  executable: false       │
│  owner: Counter Program  │     │  owner: System Program   │
│  data: {                 │     │  data: [] (empty)        │
│    count: 42,            │     │  lamports: 5 SOL         │
│    authority: User wallet│     │                          │
│    bump: 253             │     │  Signer: has private key │
│  }                       │     │                          │
│  lamports: rent-exempt   │     └─────────────────────────┘
│                          │                │
└─────────────────────────┘                │ signs
                                           ▼
                                    Transaction:
                                    "increment the counter"
```

**To increment**:
1. User signs a transaction calling `increment()` on the Counter Program
2. The transaction includes the Counter Data Account as a writable account
3. Counter Program checks: "Is the signer the `authority` stored in the data?"
4. If yes → reads `count` (42), increments to 43, writes back
5. Counter Program can do this because it's the `owner` of the data account

---

## How Accounts Are Created

Accounts don't just appear — they must be explicitly created and 
funded with SOL (for rent, covered in Lesson 4).

### Creating a System Account (Wallet)

```javascript
// When you create a wallet, you generate a keypair
import { Keypair } from "@solana/web3.js";

const newKeypair = Keypair.generate();
console.log("Public Key:", newKeypair.publicKey.toBase58());
console.log("Private Key:", newKeypair.secretKey);

// This keypair exists locally. The account on-chain doesn't exist yet.
// It only "exists" on Solana once someone sends SOL to it.
```

### Creating a Data Account (Program-Owned)

To create an account that a program owns, you use `SystemProgram.createAccount`:

```javascript
import { SystemProgram, Transaction } from "@solana/web3.js";

const createAccountIx = SystemProgram.createAccount({
  fromPubkey: payer.publicKey,        // who's paying for it
  newAccountPubkey: newAccount.publicKey,  // the new account's address
  lamports: 1_000_000,                // SOL for rent exemption
  space: 100,                         // bytes of storage space
  programId: myProgramId,             // which program will OWN this account
});

// After this instruction executes:
// - The new account exists on-chain
// - It has 100 bytes of data[] space
// - Its owner field is set to myProgramId
// - Only myProgramId can modify its data[]
```

### Creating a PDA (Program Derived Address)

PDAs are created differently — since no keypair exists, the program 
itself must create it using a Cross-Program Invocation (CPI) to the 
System Program:

```javascript
// From client side, you derive the address:
const [pda, bump] = PublicKey.findProgramAddressSync(
  [Buffer.from("counter"), userWallet.toBuffer()],
  counterProgramId
);

// The actual account creation happens INSIDE the program via CPI
// (We'll cover this in detail in the Rust/Anchor phases)
```

---

## Account Size and Data Layout

Every account has a fixed size that's set at creation time.
You cannot resize an account after creation (unless you use `realloc`).

```
Account space examples:

Wallet (System Account):        0 bytes of data
Token Account:                  165 bytes of data
Mint Account:                   82 bytes of data
Token Metadata Account:         ~679 bytes of data

The data inside is serialized using Borsh (Binary Object
Representation Serializer for Hashing) — Solana's preferred
serialization format.
```

### Borsh Serialization (Preview)

When data is stored in an account, it's packed into bytes:

```javascript
// Conceptual: What a Token Account's data[] looks like

// The struct definition (in Rust, on the program side):
// struct TokenAccount {
//   mint: Pubkey,      // 32 bytes
//   owner: Pubkey,     // 32 bytes  
//   amount: u64,       // 8 bytes
//   delegate: Option<Pubkey>,  // 36 bytes (4 + 32)
//   state: u8,         // 1 byte
//   ...
// }

// When stored on-chain, it's just a flat byte array:
// [32 bytes mint][32 bytes owner][8 bytes amount][36 bytes delegate][1 byte state]...
// Total: 165 bytes
```

On the client side (JavaScript), you'll decode these bytes back into 
readable data. We'll do this extensively in Phase 3.

---

## Solana vs Ethereum: Side-by-Side Comparison

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ETHEREUM MODEL                                │
│                                                                      │
│  Contract = Code + State (combined)                                  │
│  One contract stores ALL user balances internally                    │
│  State is accessed via storage slots (key-value)                     │
│  Contract can modify its own state freely                            │
│  Deploying = creating one contract with all logic                    │
│                                                                      │
│  Pros: Simple mental model, easy for beginners                       │
│  Cons: Sequential execution, storage bottleneck, hard to parallelize │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                        SOLANA MODEL                                  │
│                                                                      │
│  Program = Code ONLY (stateless, read-only)                          │
│  Each user has their OWN data account for each program               │
│  State is accessed via separate accounts passed as inputs            │
│  Program can only modify accounts it owns                            │
│  Deploying = creating program + creating data accounts separately    │
│                                                                      │
│  Pros: Parallel execution! (different accounts = different threads)  │
│  Cons: More complex mental model, account management overhead        │
└──────────────────────────────────────────────────────────────────────┘
```

### Why Separation Enables Parallelism

This is the killer advantage of Solana's model:

```
Ethereum (Sequential):
  Tx1: Alice sends USDC to Bob    ← touches contract storage
  Tx2: Charlie sends USDC to Dave ← touches SAME contract storage
  → Must execute Tx1 THEN Tx2 (sequential, because same storage)

Solana (Parallel):
  Tx1: Alice sends USDC to Bob    ← touches Alice's account + Bob's account
  Tx2: Charlie sends USDC to Dave ← touches Charlie's account + Dave's account
  → Can execute Tx1 AND Tx2 simultaneously! (different accounts = no conflict)
  → This is what Sealevel runtime does (Solana's parallel execution engine)
```

**Sealevel** is Solana's runtime that looks at which accounts each transaction
touches. If two transactions touch completely different accounts, they can 
run in parallel across different CPU cores. This is a MASSIVE performance win.

---

## Summary

| Concept | Details |
|---------|---------|
| Everything is an account | Wallets, programs, data, tokens — all accounts |
| Programs are stateless | Code only, no internal storage |
| Data lives in separate accounts | Programs modify external accounts |
| Owner field controls write access | Only the owner program can modify data[] |
| Two kinds of "owner" | Account owner (program) vs token owner (wallet) |
| PDAs | Deterministic addresses without private keys, program-controlled |
| Accounts have fixed sizes | Set at creation, data is Borsh-serialized bytes |
| Separation enables parallelism | Sealevel runs non-conflicting txns in parallel |

---

## Mental Model Cheat Sheet

Think of Solana like a **post office**:

- **Programs** = Post office workers (they process mail but don't keep any)
- **Accounts** = PO Boxes (each box holds someone's stuff)
- **Owner** = Which department manages which boxes
- **PDAs** = Boxes that only the post office can open (no customer key exists)
- **Transactions** = Customers submitting forms to the workers
- **Signers** = Customers showing their ID to prove they own a PO Box
