# Lesson 5: SPL Tokens & Associated Token Accounts (ATAs)

## Why One Token Program Rules Them All

On Ethereum, every token is its OWN smart contract. USDC has its own contract,
LINK has its own contract, SHIB has its own contract. Each one is a separate 
deployment with its own code.

On Solana, there is a **single Token Program** that manages ALL tokens.
USDC, BONK, JUP, RAY — they all use the same program. The tokens are
differentiated by their **Mint Accounts**, not by separate programs.

```
Ethereum:
  USDC Contract (code + state)
  LINK Contract (code + state)
  SHIB Contract (code + state)
  → Each is a separate smart contract with duplicated logic

Solana:
  Token Program (single program, shared by ALL tokens)
    ├── USDC Mint Account (defines USDC properties)
    ├── BONK Mint Account (defines BONK properties)
    ├── JUP Mint Account  (defines JUP properties)
    └── ... every SPL token ever created
  → One program, many mints. Massively efficient.
```

---

## The Token Ecosystem: Three Account Types

To understand SPL tokens, you need to understand three account types:

### 1. Mint Account — The Token Definition

A Mint Account defines a token. It's like the "birth certificate" of a token.

```
┌──────────────────────────────────────────┐
│          MINT ACCOUNT (82 bytes)          │
├──────────────────────────────────────────┤
│                                          │
│  mint_authority:  Pubkey | null           │
│    → Who can mint (create) new tokens    │
│    → Set to null = fixed supply forever  │
│                                          │
│  supply:  u64                            │
│    → Total tokens currently in existence │
│    → Updates when tokens are minted/burnt│
│                                          │
│  decimals:  u8                           │
│    → Number of decimal places            │
│    → USDC = 6 (1 USDC = 1,000,000 units)│
│    → SOL  = 9 (1 SOL = 10^9 lamports)   │
│    → NFTs = 0 (1 NFT = 1 unit, no fracs)│
│                                          │
│  is_initialized:  bool                   │
│    → Has this mint been set up?          │
│                                          │
│  freeze_authority:  Pubkey | null        │
│    → Who can freeze token accounts       │
│    → Used for compliance/regulatory      │
│    → null = nobody can freeze            │
│                                          │
│  Account-level owner: Token Program      │
│  Size: 82 bytes                          │
│  Rent: ~0.0015 SOL                       │
└──────────────────────────────────────────┘
```

**Real example — USDC on Solana:**
```
Mint Address:  EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
Decimals:      6
Supply:        ~2,500,000,000,000,000 (2.5 billion USDC in base units)
Mint Authority: Circle's address (they can mint more USDC)
Freeze Authority: Circle's address (they can freeze accounts for compliance)
```

### 2. Token Account — Holds Someone's Balance

A Token Account holds a specific token for a specific wallet.
Every wallet-token combination needs its own Token Account.

```
┌──────────────────────────────────────────┐
│       TOKEN ACCOUNT (165 bytes)           │
├──────────────────────────────────────────┤
│                                          │
│  mint:  Pubkey                           │
│    → WHICH token this account holds      │
│    → Points to the Mint Account          │
│                                          │
│  owner:  Pubkey                          │
│    → WHOSE tokens these are              │
│    → The wallet that can spend them      │
│    → (This is the "token owner", NOT the │
│       account-level owner!)              │
│                                          │
│  amount:  u64                            │
│    → How many tokens are in here         │
│    → In base units (with decimals)       │
│                                          │
│  delegate:  Pubkey | null                │
│    → Someone else authorized to spend    │
│    → Like ERC-20 approve() in Ethereum   │
│                                          │
│  delegated_amount:  u64                  │
│    → How much the delegate can spend     │
│                                          │
│  state:  enum                            │
│    → Initialized / Frozen / Uninitialized│
│                                          │
│  Account-level owner: Token Program      │
│  Size: 165 bytes                         │
│  Rent: ~0.002 SOL                        │
└──────────────────────────────────────────┘
```

### 3. Associated Token Account (ATA) — The Standard Token Account

Here's the problem: Token Accounts can have ANY address. If Alice has USDC,
how does Bob know WHERE Alice's USDC account is to send her tokens?

**ATAs solve this** by deriving the token account address deterministically:

```
ATA address = PDA derived from:
  seeds: [wallet_address, TOKEN_PROGRAM_ID, mint_address]
  program: Associated Token Account Program

This means:
  Given a wallet address + a token mint
  → You can ALWAYS compute exactly where their token account is
  → No need to ask, no need to store it, no need to look it up
```

```javascript
import { getAssociatedTokenAddress } from "@solana/spl-token";

// Given Alice's wallet and USDC mint, compute her ATA:
const aliceUsdcATA = await getAssociatedTokenAddress(
  new PublicKey("EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"),  // USDC mint
  aliceWallet.publicKey,  // Alice's wallet
);

// This ALWAYS returns the same address for the same wallet + mint combo
// No randomness, no lookup needed — it's deterministic!
console.log("Alice's USDC ATA:", aliceUsdcATA.toBase58());
```

---

## How It All Connects

```
Alice wants to hold USDC and BONK tokens:

Alice's Wallet (System Account)
  ├── pubkey: Alice...
  ├── lamports: 5 SOL
  ├── data: [] (empty — wallet doesn't store token data!)
  └── owner: System Program

Alice's USDC Token Account (ATA)
  ├── pubkey: derived from [Alice + USDC mint]
  ├── mint: EPjFWdd5... (USDC)
  ├── token_owner: Alice's wallet
  ├── amount: 500,000,000 (500 USDC)
  └── account owner: Token Program

Alice's BONK Token Account (ATA)
  ├── pubkey: derived from [Alice + BONK mint]
  ├── mint: DezXAZ8... (BONK)
  ├── token_owner: Alice's wallet
  ├── amount: 1,000,000,000,000 (1M BONK)
  └── account owner: Token Program

Key insight:
  Alice's wallet does NOT "contain" her tokens.
  Her tokens are in SEPARATE accounts (ATAs).
  Her wallet just has SOL for fees and rent.
  
  When you "connect your wallet" to a dApp, the dApp:
  1. Gets your wallet address
  2. Derives your ATA addresses for each token it cares about
  3. Reads the balance from each ATA
  4. Displays your balances!
```

---

## Token Operations (Client-Side)

### Creating a Token (Mint)

```javascript
import {
  createMint,
  getMint,
  getOrCreateAssociatedTokenAccount,
  mintTo,
  transfer,
  burn,
  TOKEN_PROGRAM_ID,
} from "@solana/spl-token";

// Step 1: Create a new token mint
const mint = await createMint(
  connection,
  payer,                    // who pays for the account creation
  mintAuthority.publicKey,  // who can mint new tokens
  freezeAuthority.publicKey, // who can freeze accounts (or null)
  9,                        // decimals (9 = same as SOL)
);

console.log("New token mint:", mint.toBase58());
// Output: "New token mint: 7xKXtg2CW87d97TXJSDpb..."
```

### Creating an ATA

```javascript
// Step 2: Create (or get) a token account for a wallet
const tokenAccount = await getOrCreateAssociatedTokenAccount(
  connection,
  payer,                    // who pays rent for the new account
  mint,                     // which token
  owner.publicKey,          // whose wallet
);

console.log("Token account:", tokenAccount.address.toBase58());
console.log("Balance:", tokenAccount.amount.toString());
// Initially: "Balance: 0"
```

### Minting Tokens

```javascript
// Step 3: Mint tokens into the token account
await mintTo(
  connection,
  payer,                    // who pays the fee
  mint,                     // which token to mint
  tokenAccount.address,     // where to send the minted tokens
  mintAuthority,            // must be the mint authority (signer!)
  1_000_000_000,            // amount in base units (1 token with 9 decimals)
);

// Now the token account has 1,000,000,000 base units = 1 token
```

### Transferring Tokens

```javascript
// Step 4: Transfer tokens between wallets
// First, ensure the recipient has an ATA
const recipientATA = await getOrCreateAssociatedTokenAccount(
  connection, payer, mint, recipientWallet.publicKey
);

await transfer(
  connection,
  payer,                    // fee payer
  senderATA.address,        // from
  recipientATA.address,     // to
  senderWallet,             // authority (signer — proves ownership)
  500_000_000,              // amount (0.5 tokens with 9 decimals)
);
```

### Burning Tokens

```javascript
// Step 5: Burn (destroy) tokens permanently
await burn(
  connection,
  payer,                    // fee payer
  tokenAccount.address,     // which token account to burn from
  mint,                     // the token mint
  owner,                    // authority (signer)
  100_000_000,              // amount to burn
);

// The token supply in the Mint Account decreases by this amount
```

---

## Token-2022 (Token Extensions Program)

Solana has a NEWER token program called **Token-2022** (also called 
Token Extensions). It's a superset of the original Token Program with 
powerful additional features:

```
Original Token Program:
  ✅ Basic mint, transfer, burn
  ✅ Freeze/thaw accounts
  ✅ Delegate (approve spending)
  ❌ No transfer fees
  ❌ No interest
  ❌ No confidential transfers
  ❌ No permanent delegate
  
Token-2022 (Token Extensions):
  ✅ Everything above PLUS:
  ✅ Transfer Fees — take a % on every transfer (built-in royalties!)
  ✅ Interest-Bearing — tokens accrue interest over time
  ✅ Confidential Transfers — hide amounts using zero-knowledge proofs
  ✅ Permanent Delegate — an authority that can ALWAYS transfer/burn
  ✅ Non-Transferable — soulbound tokens
  ✅ Default Account State — new accounts start frozen
  ✅ CPI Guard — prevent certain CPI patterns
  ✅ Metadata — store name/symbol/image directly in the mint (no Metaplex needed!)
  ✅ Transfer Hook — run custom logic on every transfer
```

### Token-2022 Example: Transfer Fees

```javascript
import { createMintWithTransferFee } from "@solana/spl-token";

// Create a token where 1% of every transfer goes to a fee wallet
const mint = await createMintWithTransferFee(
  connection,
  payer,
  mintAuthority.publicKey,
  freezeAuthority.publicKey,
  9,                    // decimals
  100,                  // fee basis points (100 = 1%)
  BigInt(1_000_000),    // max fee (cap on the fee amount)
  feeAuthority.publicKey,
  withdrawAuthority.publicKey,
);

// Now every time anyone transfers this token:
// 1% of the transfer amount is withheld in the recipient's account
// The withdraw authority can collect all accumulated fees
```

This is massive for real-world tokens — built-in revenue on every transfer!

---

## NFTs on Solana (Preview)

NFTs are just SPL tokens with specific properties:

```
NFT = SPL Token where:
  - Decimals = 0 (no fractions — you can't have 0.5 of an NFT)
  - Supply = 1 (exactly one token exists)
  - Mint Authority = null (nobody can mint more — it's unique)

The NFT metadata (name, image, attributes) is stored in a 
SEPARATE Metadata Account managed by the Metaplex Token Metadata program.

┌─────────────────┐     ┌──────────────────────┐
│  Mint Account    │     │  Metadata Account     │
│  (Token Program) │     │  (Metaplex Program)   │
│                  │     │                       │
│  supply: 1       │     │  name: "Cool Ape #42" │
│  decimals: 0     │     │  symbol: "CAPE"       │
│  mint_auth: null │     │  uri: "https://..."   │
│                  │     │  creators: [...]       │
└─────────────────┘     │  royalties: 5%        │
                        └──────────────────────┘

The URI points to a JSON file (usually on Arweave or IPFS) 
that contains the image URL and attributes.
```

We'll dive deep into NFTs in Phase 4 when we use Metaplex UMI.

---

## Summary

| Concept | Details |
|---------|---------|
| Token Program | Single program manages ALL SPL tokens on Solana |
| Token-2022 | Extended version with transfer fees, hooks, confidential transfers |
| Mint Account | Defines a token: supply, decimals, authorities (82 bytes) |
| Token Account | Holds tokens for a wallet: balance, owner, delegate (165 bytes) |
| ATA | Deterministic token account: derived from wallet + mint, always findable |
| Mint Authority | Can create new tokens; set to null for fixed supply |
| Freeze Authority | Can freeze/unfreeze token accounts |
| NFTs | SPL tokens with decimals=0, supply=1, null mint authority |
| Token operations | create mint → create ATA → mint → transfer → burn |

---

## Key Mental Model

```
Want to check Alice's USDC balance?

  1. You know Alice's wallet address
  2. You know USDC's mint address
  3. Derive the ATA: seeds = [alice_wallet, TOKEN_PROGRAM, usdc_mint]
  4. Read the ATA's `amount` field
  5. Divide by 10^decimals (10^6 for USDC)
  6. Display: "Alice has 500.00 USDC"

This is EXACTLY what Phantom/Solflare do when you open your wallet!
```
