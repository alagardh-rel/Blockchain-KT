# Solana Internals: A Technical Deep Dive

## Table of Contents
1. [Cryptography Fundamentals](#cryptography)
2. [Keys and Addresses](#keys-addresses)
3. [Wallet Architecture](#wallets)
4. [Account Model](#accounts)
5. [Transactions](#transactions)
6. [Transaction Lifecycle](#tx-lifecycle)
7. [Replay Protection](#replay)
8. [Validators and Authority](#validators)

---

## Cryptography Fundamentals {#cryptography}

### Understanding Public Key Cryptography

**Layman's Explanation:**
Imagine two boxes with special locks:
- You have a **private key** (secret key to your lock - never share it)
- You give others your **public key** (the lock itself - public to everyone)
- Anyone can lock something with your public key (send you money)
- Only you with the private key can unlock it (spend money)

**Technical Reality:**
Public key cryptography uses mathematical functions that are easy in one direction, hard in the other:
- Signing: Private key → Signature (easy with private key, hard without it)
- Verification: Public key → Verify signature (easy with public key)

### Ed25519: Solana's Cryptographic Signature Scheme

**What It Is:**
Ed25519 is an elliptic curve cryptography algorithm that Solana uses for signing transactions.

**Why It's Better Than ECDSA (Ethereum's scheme):**

| Feature | ECDSA (Ethereum) | Ed25519 (Solana) |
|---------|------------------|-----------------|
| **Security Level** | 256-bit equivalent | 128-bit equivalent (but sufficient) |
| **Signature Size** | 64-65 bytes | 64 bytes |
| **Speed** | Slower verification | Faster verification |
| **Malleability** | Can have issues | Immune to malleability |
| **Implementation** | Complex | Simpler, fewer bugs |

**How Ed25519 Works (Simplified):**
```
Private Key: 32 random bytes (256 bits)
    ↓ (Hash function)
Seed: 32 bytes
    ↓ (Expand seed)
Scalar (a): 32 bytes used for signing
Point (A): Public key (32 bytes) = Scalar × Base Point
    ↓
To Sign a Message:
├─ Hash message + scalar = r
├─ Point R = r × Base Point
├─ Hash(R, A, message) = k
└─ Signature = (R, k × scalar + r mod order)
    ↓
Result: 64-byte signature (32 bytes R + 32 bytes S)
```

**Key Point:** The entire math happens in a mathematical space (elliptic curve). No "real" multiplication or division—just mathematical operations on numbers.

### Base58 Encoding

**What It Is:**
Base58 is a way to convert binary data (bytes) into readable text characters.

**Why Not Base64?**
Base64 uses 64 characters including `0` (zero), `O` (letter O), `l` (lowercase L), `I` (uppercase I), `+` (plus), `/` (slash).
- These characters look similar or confusing (0 vs O, l vs I)
- Plus and slash are special in URLs
- **Base58 removes all these confusing characters**

**Base58 Alphabet:**
```
123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
(No 0, O, I, l)
```

**Example:**
```
Raw bytes:    0x00, 0x14, 0x6B, 0x88, 0x9C...
Base58:       "11c8GmxWREg8eXPUz7d88yqXjx3b3u1"
```

**Why Solana Uses It:**
- Addresses are human-readable
- Less prone to copy-paste errors
- Compact representation of long key data

---

## Keys and Addresses {#keys-addresses}

### Private Keys, Public Keys, and Addresses

**The Three Tiers:**

```
Tier 1: PRIVATE KEY
├─ 32 bytes (256 bits) of random data
├─ Must be kept SECRET
├─ Used to sign transactions
└─ Never transmitted

    ↓ (Derive using algorithm)

Tier 2: PUBLIC KEY
├─ 32 bytes derived from private key
├─ Can be shared publicly
├─ Used by others to verify your signature
└─ Cannot derive private key from public key (one-way function)

    ↓ (In Solana, usually same as address)

Tier 3: ADDRESS
├─ 32 bytes (same as public key in simple wallets)
├─ Identifies an account on blockchain
├─ Used to receive funds
└─ Base58 encoded for display
```

**Visual Key Derivation:**
```
Random Seed (256 bits)
        ↓
   Ed25519 Algorithm
        ↓
Private Key (32 bytes, secret)
        ↓ (One-way function)
Public Key (32 bytes, public)
        ↓ (In Solana = address directly)
Address (32 bytes, displayed in Base58)
```

### Comparison with Ethereum

| Feature | Bitcoin | Ethereum | Solana |
|---------|---------|----------|--------|
| **Private Key** | 32 bytes | 32 bytes | 32 bytes |
| **Public Key Format** | Compressed 33B | Uncompressed 64B | 32 bytes (Ed25519) |
| **Address Derivation** | Hash of pubkey | Hash of pubkey | = Public key |
| **Address Size** | 20 bytes | 20 bytes | 32 bytes |
| **Signature Algorithm** | ECDSA | ECDSA | Ed25519 |
| **Signature Size** | 64-72 bytes | 64 bytes (compressed) | 64 bytes |

**Why Solana's Addresses are 32 Bytes:**
- Solana's public keys ARE the addresses (no hashing step)
- Provides better collision resistance
- Simpler architecture

---

## Wallet Architecture {#wallets}

### From Seed Phrase to Keypair to Address

**Step-by-Step Flow:**

```
Step 1: SEED PHRASE
├─ 12 or 24 English words (BIP39 standard)
├─ Example: "abandon ability able about above absent..."
└─ Human-readable but hard to memorize

    ↓ (BIP39 algorithm)

Step 2: MASTER SEED
├─ 512 bits (64 bytes) of binary data
├─ Generated deterministically from seed phrase
└─ Same phrase always generates same seed

    ↓ (PBKDF2 key derivation)

Step 3: MASTER KEYPAIR (Root)
├─ Private key derived from master seed
├─ Can't derive this from seed directly (need derivation path)
└─ Used as starting point for HD wallets

    ↓ (BIP44 / BIP32 derivation path)

Step 4: ACCOUNT KEYPAIR
├─ Derived using specific path: m/44'/501'/0'/0'/0'
├─ Private key for this account
└─ Used to sign transactions

    ↓ (Ed25519 derivation)

Step 5: ADDRESS (BASE58)
├─ Public key = Address (32 bytes)
├─ Encoded in Base58 for display
└─ Can receive SOL at this address
```

**Concrete Example:**

```
Seed Phrase:
"abandon ability able about above absent absolute abuse access accident 
account achieve"

    ↓ BIP39 (PBKDF2-HMAC-SHA512)

Master Seed (64 bytes):
0x60c57bcd91b4d8b4cfc656b8c9c9ac7d8c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f
1f0e9d8c7b6a5f4e3d2c1b0a9f8e7d6c5b4a39281726150403f2e1d0c0b0a

    ↓ BIP32 (Master private key)

Master Private Key:
1234567890abcdef... (32 bytes)

    ↓ BIP44 Derivation Path: m/44'/501'/0'/0'/0'
      (Bitcoin=0', Ethereum=60', Solana=501')

Account 0 Private Key:
abcdef1234567890... (32 bytes)

    ↓ Ed25519 Public Key Derivation

Account 0 Public Key (Address):
8e74fbE1NHhbw6X9DQ2wE6WbH7NvqvJ2C1VFhcQPg8J2

    ↓ Base58 Encoding

Solana Address:
8e74fbE1NHhbw6X9DQ2wE6WbH7NvqvJ2C1VFhcQPg8J2 (displayed)
```

### HD Wallets and Derivation Paths

**What is an HD Wallet?**
HD = Hierarchical Deterministic. One seed phrase generates multiple keypairs.

**Why Do We Need This?**
- Generate multiple accounts from one seed
- Each account has different address
- All recoverable from single seed phrase
- Standard way: BIP32/BIP44

**BIP44 Standard (What Most Wallets Use):**

```
m / purpose' / coin_type' / account' / change / address_index

m                 = Root (master key)
purpose'          = 44' (BIP44 standard)
coin_type'        = 501' (Solana-specific)
account'          = 0', 1', 2'... (different accounts in wallet)
change            = 0 (receiving) or 1 (change)
address_index     = 0, 1, 2... (different addresses in account)

Example Paths:
m/44'/501'/0'/0'/0    = First account, first address
m/44'/501'/0'/0'/1    = First account, second address
m/44'/501'/1'/0'/0    = Second account, first address
```

**Derivation Process:**

```
Master Seed
    ↓
Using path: m/44'/501'/0'/0'/0
    ├─ Apply HMAC-SHA512 with "44" parameter
    ├─ Take left 32 bytes = new private key
    ├─ Repeat for 501' (coin type)
    ├─ Repeat for 0' (account)
    ├─ Repeat for 0 (change)
    └─ Repeat for 0 (index)
    ↓
Final Private Key: derived deterministically
```

**Important:** The `'` (called "hardened" in BIP32) means:
- Uses private key for derivation (not public key)
- Cannot derive child from public parent
- More secure, but child cannot be determined without private key

---

## Account Model {#accounts}

### Solana's Account System (vs Ethereum)

**Ethereum Model (Account-based):**
```
EOA (Externally Owned Account)
├─ Has nonce
├─ Has balance
└─ Controlled by private key

Contract Account
├─ No private key
├─ Has code
├─ Has storage
└─ Triggered by transactions
```

**Solana Model (Account-based but different):**
```
Account Structure:
├─ address (32 bytes, public key)
├─ owner (32 bytes, who controls this account)
├─ lamports (balance in lamports)
├─ data (raw bytes of account data)
├─ executable (is this a program?)
└─ rent_epoch (when rent was last collected)
```

**Key Difference:**
Ethereum: Account IS the entity
Solana: Account is a DATA STRUCTURE with owner

### Lamports and Rent

**Lamports: The Currency**
```
1 SOL = 1,000,000,000 lamports (1 billion)
1 lamport = 0.000000001 SOL (1 nano-SOL)
```

**Why Lamports?**
- Avoid decimal issues in computation
- Consistent with satoshis in Bitcoin
- Easier for integer-only operations

**Rent: The Storage Cost**

**Concept:** Accounts cost money to store on blockchain. Similar to hard drive rental.

```
Annual Rent Calculation:
───────────────────────
Rent per byte per year = Base Rate (currently ~19.1 lamports)
Account data size = 128 bytes
Annual rent = 128 bytes × 19.1 = 2443.2 lamports (~0.0000024 SOL)

2-year rent = 4886.4 lamports

Exemption:
├─ If account has rent_exempt = true
├─ Rent is never charged
└─ Requires balance ≥ 2 years rent
```

**Rent Calculation Code (Conceptual):**
```
rent_per_byte_per_year = 19.1
account_size = account.data.len() + 140 bytes (overhead)
annual_rent = (account_size) × rent_per_byte_per_year

// Must keep this much SOL to never pay rent
minimum_balance = annual_rent × 2

is_rent_exempt = (account.lamports >= minimum_balance)
```

**Who Pays Rent?**
- Whoever executes the transaction gets charged
- Usually the account owner
- Can be paid by anyone (rent payer)

### Account Ownership and Executability

**Owner: Who Controls the Account**
```
Account Structure:
├─ owner: 32-byte address (usually program address)
├─ Example owners:
│  ├─ System Program (for SOL transfers)
│  ├─ Token Program (for fungible tokens)
│  ├─ NFT Program (for NFTs)
│  └─ or any custom program
└─ Only owner can modify account data
```

**Ownership Rules:**
```
Only the account's owner can:
├─ Modify the data field
├─ Reduce lamports (withdraw)
└─ Assign new owner

Anyone can:
├─ Add lamports (deposit)
└─ Execute owner's program
```

**Executable Accounts: Smart Contracts**

```
Regular Account:
├─ executable = false
├─ owner = State Program (owns data)
└─ data = User data

Executable Account (Smart Contract):
├─ executable = true
├─ owner = BPF Loader (upgradeable or immutable)
├─ data = Compiled program code (BPF bytecode)
└─ Can be called by instructions
```

**Example: Token Transfer Transaction**

```
Transaction: Transfer 100 tokens from Alice to Bob

Accounts involved:
├─ payer: Alice (pays transaction fee)
├─ alice_token_account (owned by Token Program)
│  ├─ owner = Token Program
│  ├─ data = {amount: 1000, mint: XXX}
│  └─ executable = false
├─ bob_token_account (owned by Token Program)
│  ├─ owner = Token Program
│  ├─ data = {amount: 500, mint: XXX}
│  └─ executable = false
└─ token_program (the program being invoked)
   ├─ owner = BPF Loader
   ├─ data = compiled token contract code
   └─ executable = true

What Happens:
1. Token Program is called with instruction
2. Token Program checks alice_token_account.owner == token_program
3. Token Program checks alice_token_account.amount >= 100
4. Token Program modifies both accounts (can do because it owns them)
5. Balances updated atomically
```

### Associated Token Accounts (ATA)

**The Problem:**

In Ethereum, you have one address, and it automatically supports all tokens:
```
// Ethereum
Alice: 0x1234...
├─ Balance: 5 ETH
├─ Token A: 100 tokens
├─ Token B: 50 tokens
└─ (stored together)
```

In Solana, you need separate accounts for each token:
```
// Solana (without ATA)
Alice: 8e74fbE1...
├─ SOL Balance: 5 lamports
├─ Token A Account: 9k1x2d3f... (different address!)
├─ Token B Account: 7m9n0p1q... (different address!)
└─ (separate accounts)
```

**Solution: Associated Token Account (ATA)**

**Concept:** Derive token account address from owner + mint.

```
To send Token A to Alice:
1. Derive Alice's ATA for Token A
   INPUT: Alice's address + Token A mint address
   FUNCTION: SHA256(...)
   OUTPUT: Same address every time (deterministic)

2. ATA Address = Pubkey::find_pda(&[
     token_account_owner.as_ref(),  // Alice
     token_program.as_ref(),         // Token Program
     mint.as_ref()                   // Token A mint
   ], &token_program)

3. Creates account if doesn't exist:
   owner = Token Program
   data = {mint: Token A, owner: Alice, amount: 0}

4. Send tokens to this ATA
```

**Why It Works:**
- Same owner + mint = same address every time
- No need to ask "what's your Token A account?"
- Standardized location
- Can auto-create if doesn't exist

### PDAs: Program Derived Addresses

**What is a PDA?**
An address derived from:
- A program ID
- Some seeds (unique data)
- Bump (nonce to ensure valid Ed25519 point)

**Important:** PDAs don't have private keys!

```
PDA Generation:
INPUT: Program ID + seeds + bump

  SHA512(
    "ProgramDerived",
    program_id,
    seed_1,
    seed_2,
    ...
    seed_n,
    bump
  )

1. Compute the hash
2. Check if it's a valid Ed25519 point
3. If not, increment bump and retry
4. When valid, that's the PDA address

Result: Address that "belongs to" program but no one has private key
```

**Usage Example: Program Vault**

```
Imagine you're building a staking program that holds users' SOL.

Design:
├─ Vault PDA owned by program (not user)
├─ Seeds: ["vault", user_address]
├─ Bump: found automatically

Creating vault for Alice:
```solana
Seeds: ["vault", alice_address]
Bump: 250 (found by algorithm)

PDA Address: E7x4v8K2... (deterministic for Alice)
Owner: Staking Program
Purpose: Hold Alice's staked SOL

When Alice stakes 10 SOL:
├─ Alice signs transaction
├─ Staking Program transfers 10 SOL to PDA
└─ PDA stores [alice_address, 10 amount]
```

**Why Not Just Use User Address?**
- User would need to hold tokens
- Better security if program controls vault
- Program can enforce rules
- User cannot accidentally drain vault

**Why Not Private Key?**
- Why would we have a key?
- Only program should control it
- Deterministic, no key management needed
- More secure (no private key to steal)

---

## Transactions {#transactions}

### Transaction Structure

**What is a Solana Transaction?**
A signed message that tells validators:
- "I authorize these changes"
- "Change these accounts"
- "Here's my signature proving it"

**Visual Structure:**

```
Solana Transaction
│
├─ Message
│  ├─ Header
│  │  ├─ Num readonly signers
│  │  ├─ Num readonly non-signers
│  │  └─ Num required signers (signatures needed)
│  │
│  ├─ Account Addresses (list of all accounts)
│  │
│  ├─ Recent Blockhash (for replay protection)
│  │
│  └─ Instructions (list of operations)
│     ├─ Program ID (which program to invoke)
│     ├─ Accounts (which accounts to pass)
│     │  ├─ [Account A, is_signer, is_writable]
│     │  ├─ [Account B, is_signer, is_writable]
│     │  └─ [...]
│     ├─ Data (instruction data, program-specific)
│     └─ (more instructions...)
│
└─ Signatures (list of signatures)
   ├─ [Signature 1]
   ├─ [Signature 2]
   └─ [...]
```

### Message Structure Deep Dive

**Header:**

```c
struct Header {
    uint8_t num_required_signers;      // How many signatures needed
    uint8_t num_readonly_signers;      // Readonly but must sign
    uint8_t num_readonly_non_signers;  // Readonly and don't sign
}
// Total accounts = required + readonly_signers + readonly_non_signers
```

**Account Metas:**

```
For each instruction, list which accounts it uses:

[
  {
    address: "TokenkegQfeZyiNwAJsyFbPVwwQQftas5EMQQqJL8Aw",  // Token Program
    is_signer: false,   // Program doesn't need to sign
    is_writable: false  // Program doesn't modify itself
  },
  {
    address: "9B5X4z6z2MPGEg25zZvzG2jhXwQaE...",  // Alice's token account
    is_signer: false,
    is_writable: true   // Will be modified (balance decreases)
  },
  {
    address: "7k2x9c3m1d5f8e...",  // Bob's token account
    is_signer: false,
    is_writable: true   // Will be modified (balance increases)
  },
  {
    address: "8e74fbE1NHhbw6X9DQ2wE6WbH7NvqvJ2C1VFhcQPg8J2",  // Alice (payer/authority)
    is_signer: true,    // Alice must sign it
    is_writable: false  // Alice's account doesn't change
  }
]
```

**Why Mark Fixed Accounts?**

```
Sealevel (parallel processor) uses this info:

is_writable = false (readonly):
├─ Multiple instructions can read same account
├─ They can run in parallel
└─ Safe (no conflicts)

is_writable = true (writeable):
├─ Only ONE instruction can modify
├─ Must run sequentially
└─ Prevents race conditions

is_signer = true:
├─ Account must provide signature
├─ Proves account owner authorized it
└─ Used for authorization
```

**Instruction Data:**

```
Raw bytes passed to program:
├─ First byte(s): Instruction discriminator
│  └─ Example: 0x01 = "transfer", 0x02 = "approve"
├─ Remaining bytes: Encoded parameters
└─ Program-specific format

Example (Token Transfer):
Bytes: [0x03, 0x64, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
       [discriminator, 64 in 8-byte little-endian]
       = Transfer 100 tokens (discriminator 3, amount 100)
```

### Fee Payer

**What is a Fee Payer?**
The account that pays the transaction fee.

```
Transaction:
├─ Payer: Alice (first signer)
├─ Payer pays: 5000 lamports (~0.000005 SOL)
├─ Fee deducted from payer lamports
└─ If payer has < 5000, transaction fails
```

**Fee Calculation:**

```
Base Fee = 5000 lamports (fixed)
Additional Fee = 0 lamports (currently, might change)

Total Fee = Base Fee × (1 + additional_factor)

Currently: 5000 lamports
```

**Who Can Be Payer?**
- Must be a signer
- Usually first signer in transaction
- Usually the transaction initiator
- Can be different from beneficiary

```
Example: Company pays for employees' transactions
├─ Corporate wallet = payer
├─ Employee wallet = beneficiary
├─ Company subsidizes fees
```

### Signatures

**What is a Signature?**
Ed25519 signature proving:
- "I (account X) authorize this transaction"
- "I (account X) authorized these changes"

```
Transaction Signing:
────────────────────────

MESSAGE (what you're signing):
├─ All account addresses
├─ Recent blockhash
├─ All instructions
└─ (serialized to 1000+ bytes)

SIGNING PROCESS (for each signer):
├─ Message → Ed25519 Sign with private key → Signature (64 bytes)
└─ Signature added to transaction

VERIFICATION (validator checks):
├─ Message + Signature + Public key → Valid? (Yes/No)
├─ If signature invalid, transaction rejected
├─ If signature valid, receiver knows it's authentic
└─ Due to Ed25519 math, cannot forge signature without private key
```

**Multiple Signatures:**

```
Transaction with 3 signers:

Message: "Transfer 100 tokens from treasury to Alice"

Signatures:
├─ Alice signs (proves she authorizes)
├─ Bob signs (proves he approves)
└─ Carol signs (proves she approves)

Intent: Requires 3 approvals (multisig)

Verification:
├─ Check Alice's signature (valid, with Alice's pubkey)
├─ Check Bob's signature (valid, with Bob's pubkey)
├─ Check Carol's signature (valid, with Carol's pubkey)
└─ All valid → Transaction proceeds
```

---

## Transaction Lifecycle {#tx-lifecycle}

### Step-by-Step: From Creation to Finality

**Step 1: Create Transaction**
```
User/client creates:
├─ Fetch recent blockhash (needed)
├─ Create message:
│  ├─ All accounts
│  ├─ Blockhash
│  └─ Instructions
└─ Message ready to sign
```

**Step 2: Sign Transaction**
```
For each required signer:
├─ Serialize message to bytes
├─ Sign with Ed25519 private key
├─ Get 64-byte signature
└─ Add to signatures list

Transaction now signed
```

**Step 3: Submit to Network**
```
TX submitted to RPC node:
├─ Node receives transaction
├─ Validates format (proper structure?)
├─ Validates signatures (correct signatures?)
├─ Validates accounts exist
└─ Transaction goes to mempool
```

**Step 4: Reach Leader**
```
Current leader's slot received the TX:
├─ Leader pulls from mempool
├─ Leader orders transactions
├─ Leader bundles into block
└─ Leader adds to its block proposal
```

**Step 5: Execute Transaction**
```
Sealevel processes transaction:
├─ Load all accounts from ledger
├─ Run program code (invoke smart contract)
├─ Program modifies accounts (if allowed)
├─ Changes staged in-memory
└─ If program returns error, all changes reverted (atomicity)
```

**Step 6: Broadcast Block**
```
Leader broadcasts block via Turbine:
├─ Block contains transaction
├─ Validators receive block
├─ Validators verify signatures
└─ Block propagates through network
```

**Step 7: Validators Vote**
```
Validators use Tower BFT:
├─ Check block is valid
├─ Vote on the block
├─ Deeper votes as more validators confirm
└─ When 2/3+ vote deeply, FINALITY reached
```

**Step 8: Finality**
```
After ~6.4 seconds (average):
├─ 2/3+ of validator stake has voted
├─ Transaction permanently in ledger
├─ Cannot be reverted
├─ Even if stake loses connection, this transaction stays
└─ ✓ Transaction finalized
```

### State Changes During Transaction

```
Before TX:
├─ Account A: 1000 lamports, balance: 100 tokens
└─ Account B: 2000 lamports, balance: 50 tokens

Transaction: "Transfer 20 tokens from A to B"

During Execution:
├─ Load Account A: {balance: 100, ...}
├─ Load Account B: {balance: 50, ...}
├─ Verify: A.balance >= 20? YES
├─ Modify A: balance = 80
├─ Modify B: balance = 70
├─ Write back to ledger
└─ Pay fee: Account A lamports -= 5000

After TX (Committed):
├─ Account A: 995 lamports, balance: 80 tokens  (20 tokens sent + 5000 lamports fee)
└─ Account B: 2000 lamports, balance: 70 tokens (received 20 tokens)
```

**Atomicity:**
```
Either ALL changes happen, or NONE:
├─ If Token Program errors mid-execution
├─ Both Account A and B revert
├─ No partial states
└─ Exception: Fee is ALWAYS deducted even if TX fails
```

---

## Replay Protection {#replay}

### The Replay Attack Problem

**What is a Replay Attack?**

Imagine:
```
1. Alice sends "Transfer 10 SOL to Bob" (signed)
2. Attacker hears the transaction
3. Attacker rebroadcasts same transaction 1000 times
4. Bob gets 10000 SOL instead of 10 SOL!
```

**Why This Would Happen:**
- Signature is permanent
- No per-transaction uniqueness
- Attacker can reuse exact same bytes

### How Solana Prevents Replays

**Method 1: Recent Blockhash** (Default)

**Concept:** Include current time in transaction.

```
Transaction includes:
├─ Recent Blockhash (hash of block 100 blocks ago)
├─ Validator checks: "Is this blockhash still valid?"
├─ Default: Yes, if blockhash is timely (within ~2-3 minutes)
├─ If blockhash too old: REJECT (this is a replay!)
└─ If blockhash not recognized: REJECT (on different chain?)
```

**Timeline Example:**
```
Slot 1000: Block created with hash: 0xAABBCC...
           TX created with: recent_blockhash = 0xAABBCC...

Slots 1001-1099: Different blocks, different hashes
                 TX still valid (within window)

Slot 1100: Blockhash 0xAABBCC no longer valid
           TX rejected (too old)

Slot 1101+: TX rejected (outside acceptable age)
```

**Why This Works:**
- Attacker cannot replay after 2-3 minutes
- Network continuously creates new blocks
- Old blockhashes become invalid
- Attacker must use current blockhash
- If they do, it's just a new transaction (fair)

### Method 2: Durable Nonce (Advanced)

**Problem with Recent Blockhash:**
```
You're building an app that submits transactions later:
├─ User signs TX at time T
├─ TX might submit at time T + 5 minutes
├─ Recent blockhash now expired
├─ TX fails!
```

**Solution: Durable Nonce Account**

**Concept:** Use a special account to track transaction uniqueness.

```
Nonce Account Structure:
├─ owner: User (wallet)
├─ lamports: Minimum balance (5000 lamports) for rent exemption
├─ data:
│  ├─ nonce_value: 32-byte hash (changes each TX)
│  ├─ authority: Who can use this nonce
│  └─ fee_calculator: Fee info
```

**How Nonce Replaces Blockhash:**

```
Step 1: Create Nonce Account
├─ User creates PDA account
├─ Allocates 80 bytes data
├─ Initialize with nonce_value = Hash(...)
└─ Costs rent-exempt balance

Step 2: Use Nonce in Transaction
├─ Instead of recent_blockhash
├─ Use nonce_value from Nonce Account
├─ Can be used anytime (not time-limited!)
└─ Nonce advances with each use

Step 3: Validator Checks Nonce
├─ "Is this nonce_value current?"
├─ Only most recent nonce_value is valid
├─ If old nonce used: REJECT (replay!)
└─ If new nonce use: ACCEPT, update nonce

Step 4: Next TX
├─ Nonce has advanced to new value
├─ Must use new nonce_value
├─ Old nonce no longer valid
└─ Prevents replay forever!
```

**Who Creates Nonce Accounts?**

```
The user creates it:
├─ Pay initial rent-exempt balance
├─ Usually ~5000 lamports
├─ User is the owner
├─ User is the authority (controls it)

TX Using Nonce:
├─ Nonce account required
├─ Marked as writeable (nonce advances)
├─ Marked as not signer (doesn't need to sign)
└─ Program that uses it: Nonce Instruction Program
```

**Difference: Recent Blockhash vs Nonce**

| Aspect | Recent Blockhash | Durable Nonce |
|--------|------------------|---------------|
| **Time Valid** | 2-3 minutes | Forever (until used) |
| **Account Needed** | No | Yes (special account) |
| **Cost** | Free | Rent-exempt (~5k lamports one-time) |
| **Use Case** | Normal TX | Delayed/scheduled TX |
| **How Advances** | Auto (blocks advance) | Manual (must include in TX) |
| **Replay Window** | 2-3 minutes | One use only |

---

## Validators and Authority {#validators}

### Validator Accounts and Authorities

**The Validator Architecture:**

A validator isn't just one account. It's actually several accounts with different purposes:

```
Validator Setup (Required Accounts):

1. VALIDATOR IDENTITY (Keypair 1)
   ├─ Purpose: Identify the validator
   ├─ Used for: Signing proposal blocks
   ├─ Keeps: private key on node
   └─ Public key: validator's "name"

2. VOTE ACCOUNT (Owned by Vote Program)
   ├─ Purpose: Stores voting choices
   ├─ Owner: Vote Program (system program)
   ├─ Data: List of recent votes
   ├─ Writable by: Vote Program when voting
   └─ Contains: Links to authorities

3. STAKE ACCOUNT (Owned by Stake Program)
   ├─ Purpose: Holds staked SOL
   ├─ Owner: Stake Program
   ├─ Data: How much SOL is staked
   ├─ Associated with: Validator (via vote account)
   └─ Contains: Authorities for withdrawal

4. COMMISSION ACCOUNT
   ├─ Purpose: Where validator collects commission
   ├─ Owner: Validator
   └─ Receives: Fee rewards
```

### Validator Authority Types

**Important Distinction:**

These are different keys and can be different people!

```
VALIDATOR IDENTITY KEYPAIR
├─ Type: Keypair (has private key)
├─ Signer: Yes
├─ Function: Signs blocks created by validator
├─ Holder: Usually validator operator
├─ Risk: High (if leaked, attacker can sign blocks)
├─ Storage: Usually on air-gapped machine
└─ Updates: Very rarely changed

STAKE AUTHORITY
├─ Type: Account address (not keypair)
├─ Function: Can add/remove stake from stake account
├─ Holder: Does not need to be validator operator
├─ Can be: Different person, multisig, contract
├─ Changes: Less critical (doesn't sign blocks)
└─ Scenario: Corporate wallet manages staking

VOTE AUTHORITY
├─ Type: Account address
├─ Function: Can modify vote account parameters
├─ Changes vote account: yes
├─ Root authority: Can change itself
├─ Withdraw authority: Can withdraw rewards
└─ Can be: Similar to stake authority

WITHDRAW AUTHORITY
├─ Type: Account address
├─ Function: Can withdraw SOL from stake account
├─ Creates: Risk if too permissive
├─ Usually: Same as stake authority
└─ Separation: Can be different for security
```

### Vote Account

**Structure:**

```
Vote Account (owned by Vote Program):
│
├─ Lamports: Account balance (just enough for rent-exempt)
│
├─ Data Field:
│  ├─ node_pubkey (64 bytes)
│  │  └─ Validator identity (who's voting)
│  │
│  ├─ authorized_voters (list)
│  │  └─ Accounts that can vote on this account
│  │
│  ├─ authorized_withdrawer (32 bytes)
│  │  └─ Can withdraw vote account's lamports
│  │
│  ├─ commission (uint8)
│  │  └─ Commission percentage (0-100)
│  │
│  ├─ votes (vec of recent votes)
│  │  ├─ Slot number
│  │  ├─ Signature (block hash)
│  │  └─ (last 40-50 votes kept)
│  │
│  ├─ root_slot
│  │  └─ Most recent finalized slot
│  │
│  └─ prior_voters
│     └─ History of past authorized voters
│
└─ Owner: Vote Program
```

**Who Votes?**

```
Validator's software:
├─ Listens to blocks
├─ Validates block
├─ Creates vote for that block
├─ Calls Vote Program instruction:
│  ├─ "Vote" instruction
│  ├─ Includes: Latest vote
│  ├─ Signed by: Validator identity
│  └─ Updates: Vote account
├─ Vote account state changes
└─ Validators counting votes see new record
```

### Stake Account

**Purpose:**
Stores SOL that's locked up as validator stake.

**Structure:**

```
Stake Account (owned by Stake Program):
│
├─ owner: Stake Program
│
├─ lamports: Amount of SOL staked
│  └─ Example: 10,000 SOL = 10 billion lamports
│
├─ Data Field:
│  ├─ delegation
│  │  ├─ voter_pubkey (32 bytes)
│  │  │  └─ Validator's vote account
│  │  ├─ stake (64-bit)
│  │  │  └─ How much is delegated
│  │  └─ activation_epoch
│  │     └─ When delegation started
│  │
│  ├─ authorized (has two sub-authorities):
│  │  ├─ staker
│  │  │  └─ Can add/remove stake
│  │  └─ withdrawer
│  │     └─ Can withdraw lamports
│  │
│  ├─ lockup
│  │  ├─ custodian
│  │  │  └─ Can change lockup while locked
│  │  ├─ epoch
│  │  │  └─ Epoch when lockup ends
│  │  ├─ unix_timestamp
│  │  │  └─ Unix time when lockup ends
│  │  └─ Locked until BOTH epoch AND timestamp pass
│  │
│  └─ rent_exempt_reserve
│     └─ Minimum that's always reserved for rent
│
└─ rent_epoch
   └─ For rent calculation
```

**How Staking Works:**

```
Step 1: Create Stake Account
├─ Allocate new account
├─ Fund with SOL
├─ Example: 1000 SOL
└─ Account size: 200 bytes

Step 2: Activate Stake
├─ Specify validator (vote account)
├─ Activation epoch = current + some delay
└─ Stake becomes "active" after activation epoch

Step 3: Earning Rewards
├─ Every epoch, rewards calculated
├─ Based on validator performance
├─ Added to stake account
└─ Typically: 3-8% annually

Step 4: Inactivate Stake
├─ Can't happen immediately
├─ Inactivation epoch = current + 1
├─ Stake becomes "inactive" after that epoch

Step 5: Withdraw
├─ After inactivation, can remove SOL
├─ Requires withdrawer authority to sign
└─ Lamports moved to desired account
```

### Leader Schedule

**What is a Leader Schedule?**

A predetermined list of which validator will be leader in each slot for the next epoch.

```
Example Schedule for Epoch 500:

Slot 1000: Validator A is leader
Slot 1001: Validator B is leader
Slot 1002: Validator C is leader
Slot 1003: Validator A is leader
Slot 1004: Validator D is leader
...
Slot 431999: Validator B is leader

(432,000 slots)
```

**How Is It Determined?**

```
Before each epoch:
├─ List all active validators
├─ Sort by stake (higher stake = higher chance)
├─ Use pseudorandom algorithm
├─ Seed randomness: previous epoch hash
├─ Result: Deterministic but unpredictable
└─ Probability(validator i) = stake_i / total_stake

Example:
├─ Validator A: 100,000 SOL staked
├─ Validator B: 50,000 SOL staked
├─ Validator C: 25,000 SOL staked
├─ Total: 175,000 SOL

├─ A: 100,000 / 175,000 = 57% chance per slot
├─ B: 50,000 / 175,000 = 29% chance per slot
└─ C: 25,000 / 175,000 = 14% chance per slot

In 432,000 slots:
├─ A leads: ~246,000 slots
├─ B leads: ~125,000 slots
└─ C leads: ~60,000 slots
```

**Why Predetermined Schedule?**

```
Benefits:
├─ Everyone knows who's leading when
│  └─ Helps planning and coordination
├─ Prevents last-minute surprises
├─ Prevents "leader bribery"
│  └─ Can't determine leader at last second
├─ Validators can prepare in advance
└─ Deterministic from epoch boundary

How Validators Use It:
├─ Download leader schedule
├─ Know when you'll lead
├─ Prepare blocks in advance (via Gulf Stream)
├─ When your slot comes, ready to go
└─ Other validators know who should be leading (detect if wrong)
```

**Rewards Based on Schedule**

```
Leader Rewards:
├─ Base inflation: 8% annually (adjustable)
├─ Each validator gets: (stake / total_stake) × inflation
├─ But ONLY for slots they lead
├─ So: high-stake validators earn more

Example Calculation (per epoch):
├─ Total inflation this epoch: 1000 SOL
├─ Validator A stake: 100k SOL
├─ Total stake: 1M SOL
├─ A's share: 100k / 1M = 10%
├─ A's rewards: 10% × 1000 = 100 SOL
└─ +Transaction fees from blocks they lead

Validator Commission:
├─ Validator can set 0-100% commission
├─ Example: 5% commission
├─ 100 SOL rewards - 5 SOL commission = 95 SOL to delegators
└─ 5 SOL to validator (commission)
```

### Vote Account and Authority Separation

**Why Separate?**

```
Scenario: Validator with $10M staked

Ideal Setup:
├─ Validator Identity: On hot validator machine
│  └─ Signs blocks frequently
├─ Vote Authority: On medium security machine
│  └─ Changes vote account settings rarely
├─ Withdraw Authority: On cold storage
│  └─ Withdraws rewards/unstakes (very rarely)
└─ Stake Authority: On medium security machine

Benefit: If hot machine compromised:
├─ Attacker can ONLY sign blocks
├─ Cannot withdraw funds (separate key)
├─ Cannot vote on other validators (separate key)
└─ Damage limited
```

**Example Topology:**

```
Validator Operator owns 3 keypairs:

KEYPAIR 1 (On Validator Machine)
├─ Validator Identity
├─ Private key: sensitive
├─ Used: To sign blocks
└─ Rotation: Never (rebuild if leaked)

KEYPAIR 2 (On Secure Wallet)
├─ Authority for voting account
├─ Private key: guarded
├─ Used: Setup, change parameters
└─ Rotation: Rarely

KEYPAIR 3 (On Cold Storage)
├─ Withdraw Authority
├─ Private key: air-gapped
├─ Used: Withdraw funds
└─ Rotation: Almost never
```

---

## Summary: Transaction Flow Diagram

```
User creates transaction:
├─ Specifies: accounts, instructions, payer
├─ Fetches: Recent blockhash
└─ Creates: Message

↓

Signing:
├─ Serialize message
├─ Sign with private key (Ed25519)
└─ Get 64-byte signature

↓

Submit to Network:
├─ Send to RPC
├─ RPC validates
└─ Enters mempool

↓

Leader Includes in Block:
├─ Ordered deterministically
├─ Bundled with other TXs
└─ Block proposed

↓

Execution (Sealevel):
├─ Load accounts
├─ Run program logic
├─ Modify account states
└─ Atomically or fail

↓

Broadcast (Turbine):
├─ Broken into pieces
├─ Spread through network
└─ Validators receive

↓

Validation (Tower BFT):
├─ Check signatures
├─ Vote on block
└─ Votes accumulate

↓

Finality (~6.4 sec):
├─ 2/3+ stake voted
├─ Transaction final
└─ Cannot be reverted
```

---

## Key Concepts Table

| Concept | Definition | Size | Notes |
|---------|-----------|------|-------|
| **Private Key** | Secret signing key | 32 bytes | Never shared |
| **Public Key** | Can verify signatures | 32 bytes | Shared publicly |
| **Address** | Account identifier | 32 bytes | Same as public key in Solana |
| **Signature** | Proof of authorization | 64 bytes | Ed25519 format |
| **Lamport** | Smallest unit | - | 1 SOL = 1 billion |
| **Account** | On-chain data storage | Variable | Has owner, lamports, data |
| **PDA** | No private key address | 32 bytes | Derived deterministically |
| **ATA** | Token account per user/mint | 32 bytes | Created deterministically |
| **Nonce** | Transaction uniqueness | 32 bytes | In special account |
| **Vote Account** | Validator voting record | 200+ bytes | Owned by Vote Program |
| **Stake Account** | SOL delegation | 200 bytes | Owned by Stake Program |

