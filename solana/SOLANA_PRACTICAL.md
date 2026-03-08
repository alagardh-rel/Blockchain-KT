# Solana Practical Understanding: Learner's Guide

## Table of Contents
1. [What Are Solana Programs?](#programs)
2. [Accounts and State Storage](#accounts)
3. [Cross-Program Invocation (CPI)](#cpi)
4. [Upgradeable Programs](#upgradeable)
5. [SOL vs SPL Tokens](#sol-vs-spl)
6. [Token Mint and Authorities](#mint)
7. [End-to-End Examples](#examples)
8. [Comparison Tables](#comparisons)
9. [Common Confusions](#confusions)
10. [Glossary](#glossary)
11. [Top 10 Things to Remember](#recap)

---

## What Are Solana Programs? {#programs}

### Layman's Explanation

**Think of a Solana program like a vending machine:**

```
Normal app (Ethereum):
├─ Code runs on a single server
├─ Server stores data
└─ Works 24/7 (unless server crashes)

Solana Program:
├─ Code runs on thousands of validators
├─ Validators execute identical code
├─ Network stores data (not one server)
└─ Anyone can trigger program execution
```

### Technical Definition

**A Solana program is:**
- Smart contract code compiled to BPF bytecode
- Deployed on validators
- Stateless (no internal storage)
- Invoked by transactions
- Receives accounts as parameters
- Modifies accounts based on logic

### Programs vs Traditional Apps

| Aspect | Traditional App | Solana Program |
|--------|-----------------|----------------|
| **Where it runs** | One server | All validators |
| **Code language** | JavaScript, Python, etc | Rust, C, etc |
| **Data storage** | Database | Blockchain accounts |
| **Invocation** | HTTP request | Transaction instruction |
| **Execution model** | Stateful | Stateless |
| **Deterministic** | No (may vary) | Yes (always same result) |

### Program Types

```
System Program
├─ Built-in program
├─ Purpose: Create accounts, transfer SOL
└─ ID: 11111111111111111111111111111111

Token Program
├─ Built-in program
├─ Purpose: Manage tokens
└─ ID: TokenkegQfeZyiNwAJsyFbPVwwQQftas5EMQQqJL8Aw

Metadata Program
├─ Utility program
├─ Purpose: Store NFT metadata
└─ ID: metaqbxxUerdq8d4vs6G7Ux19x6amv2xzy...

Custom Programs
├─ Built by developers
├─ Purpose: Specific business logic (payments, DEX, etc)
└─ Deployed by developers
```

---

## Accounts and State Storage {#accounts}

### Where State Lives

**Unlike Ethereum, programs don't store state.**

**Ethereum:**
```
Program Storage:
├─ mapping(address => balance)
├─ mapping(address => approved)
└─ (built-in to contract)
```

**Solana:**
```
Program (Stateless):
├─ Just code
└─ No built-in storage

State Storage:
├─ Separate accounts (owned by program)
├─ Each account = data container
├─ Program modifies accounts passed to it
└─ Accounts pay for their storage space (rent)
```

### Account as Data Container

```
Account Structure:

Account #1 (User Balance)
├─ Owner: Token Program
├─ Lamports: 2000 (rent-exempt balance)
├─ Data:
│  ├─ mint: token_address
│  ├─ owner: alice_address
│  ├─ amount: 100 tokens
│  ├─ delegate: none
│  └─ delegated_amount: 0
└─ Executable: false

Account #2 (Program Code)
├─ Owner: BPF Loader
├─ Lamports: enough for rent
├─ Data: compiled program bytecode
└─ Executable: true
```

### Account Ownership

```
Who can modify an account?

Rule: Only the account's owner can modify its data

Example:
├─ Token Account
│  ├─ Owner: Token Program
│  └─ Only: Token Program can modify balance
├─ NFT Metadata
│  ├─ Owner: Metadata Program
│  └─ Only: Metadata Program can update metadata
└─ User Data
   ├─ Owner: Custom Program
   └─ Only: Custom Program can update
```

### Read-Only vs Writable Accounts

```
In Transaction, per account specify:

Read-Only Account (is_writable = false):
├─ Program can READ data
├─ Program CANNOT modify
├─ Multiple programs can read simultaneously
└─ Efficient (parallel processing)

Writable Account (is_writable = true):
├─ Program can READ data
├─ Program CAN modify data
├─ Only ONE program at a time
└─ Sequential processing (bottleneck)

Example:
├─ Alice's token account: WRITABLE (balance changes)
├─ Bob's token account: WRITABLE (receives tokens)
├─ Token Program: READ-ONLY (program itself doesn't change)
└─ Mint: READ-ONLY (we just check mint info)
```

### Rent and Storage

```
Storage Cost:
Rent per byte per year = 19.1 lamports

Account (128 bytes):
├─ Annual rent = 128 × 19.1 = 2,444 lamports
├─ 2-year rent = 4,888 lamports

Rent Exemption:
├─ Balance ≥ 2 years rent
└─ Never pay rent (stays on chain forever)

Who pays?
├─ Transaction executor withdraws from payer account
└─ Usually the account that creates it
```

---

## Cross-Program Invocation (CPI) {#cpi}

### What is CPI?

**CPI = One program calling another program**

### Real-World Analogy

```
Scenario: You want to swap tokens on DEX

Without CPI:
├─ You call Token Program to transfer tokens
├─ You manually specify all accounts
├─ You call DEX Program
├─ DEX Program calls Token Program again
└─ Redundant, inefficient

With CPI:
├─ DEX Program uses CPI
├─ Calls Token Program directly (from blockchain)
├─ Passes all proper accounts
└─ Atomic: Either all succeed or all fail
```

### How CPI Works

```
Timeline:

Your Transaction:
├─ Calls DEX Program
│
└─ DEX Program executes:
   ├─ Do some validation
   ├─ Use CPI to call Token Program
   │  ├─ Pass accounts: alice_token, bob_token, token_program
   │  ├─ Token Program modifies accounts
   │  └─ Return control to DEX
   ├─ Do more logic
   └─ Return success

Result:
├─ All changes atomic (all or nothing)
├─ Two programs executed in same transaction
└─ Efficient state changes
```

### Example: DEX Swap via CPI

```
Transaction: Swap 100 USDC for 500 COPE

Step 1: Caller sends transaction
├─ Program: DEX Program
├─ Accounts: [alice_usdc, bob_cope, DEX program, Token Program, USDC mint]
└─ Instruction: swap(100)

Step 2: DEX Program
├─ Checks: Can swap 100 USDC?
├─ Calculates: Get 500 COPE
├─ Uses CPI to call Token Program:
│  ├─ "Transfer 100 USDC from alice_usdc to dex_usdc"
│  └─ Token Program modifies accounts
├─ Uses CPI to call Token Program again:
│  ├─ "Transfer 500 COPE from dex_cope to bob_cope"
│  └─ Token Program modifies accounts
└─ Return success

Result:
├─ alice_usdc: 100 → 0 tokens
├─ bob_cope: 0 → 500 tokens
├─ All atomic (can't have partial swap)
└─ Both programs' logic executed
```

### Program Derived Signers

```
Problem: Only wallets have private keys to sign

CPI Solution: PDA (Program Derived Address)

PDA Characteristics:
├─ No private key (never will have one)
├─ Only program can authorize signing
├─ Deterministic (same inputs = same address)
└─ Used as: "program authority"

Example (DEX Vault):

Vault PDA (controlled by DEX):
├─ Seeds: ["vault", token_mint]
├─ Owner: DEX Program
├─ Holds: Liquidity for swaps
└─ Only DEX Program can authorize its use

When swapping:
├─ DEX Program can "sign" for DEX vault PDA
├─ Vault can transfer tokens
└─ Only DEX logic decides what happens
```

---

## Upgradeable Programs {#upgradeable}

### Two Types of Programs

```
Immutable Program:
├─ Deployed and fixed
├─ No updates ever
├─ Burn after deploying (or keep closed)
└─ Example: Protocol guaranteed never to change

Upgradeable Program:
├─ Deployed with upgrade authority
├─ Can be updated by authority
├─ Authority can be transferred
├─ Authority can be revoked (make immutable)
```

### How Upgradeable Programs Work

```
Structure:

Proxy Account (the "Program ID" you call):
├─ Owner: BPF Loader (Upgradeable)
├─ Data: Pointer to current implementation
└─ Does NOT contain code

Implementation Account (actual program code):
├─ Owner: BPF Loader (Upgradeable)
├─ Data: Program bytecode
└─ Contains actual logic

Upgrade Authority Account:
├─ Special account that can:
│  ├─ Point proxy to new implementation
│  └─ Set new upgrade authority
└─ Only this account can authorize upgrades


Upgrade Flow:

1. Deploy v1 implementation
2. Point proxy to v1
3. Users call proxy

4. Deploy v2 implementation (new account)
5. Update authority updates proxy to point to v2
6. Users call proxy (same address)
7. Proxy now executes v2 code

Result:
├─ Program ID stays same
├─ Code can change
├─ Seamless upgrade
```

### Risks

```
Mutable Programs:
├─ Developers can rug (steal funds)
├─ Code can be exploited via update
└─ Users must trust update authority

Immutable Programs:
├─ 100% safe code (cannot change)
├─ Bugs cannot be fixed
└─ More trust from users
```

---

## SOL vs SPL Tokens {#sol-vs-spl}

### SOL: Native Currency

```
SOL (Solana's native token):
├─ Built-in to protocol
├─ Allocated by System Program
├─ Transferred natively
├─ Used for: fees, rent, staking
└─ 1 SOL = 1 billion lamports

Account with SOL:
├─ account.lamports contains balance
└─ No Token Program needed (built-in)

Example (5 SOL):
├─ lamports = 5,000,000,000
└─ 5 × 10^9
```

### SPL Tokens: Custom Tokens

```
SPL (Solana Program Library) Token:
├─ Implemented by Token Program
├─ Similar to ERC-20 in Ethereum
├─ Arbitrary supply
├─ Custom properties
└─ Examples: USDC, Cope, Wrapped Bitcoin

Account with SPL Token:
├─ Owned by Token Program
├─ account.data contains token info
├─ Balance stored in account data
└─ Need associated token account per person

Example (100 USDC):
├─ account.owner = Token Program
├─ account.data:
│  ├─ mint: USDC_mint_address
│  ├─ owner: your_wallet
│  ├─ amount: 100,000,000 (6 decimals)
│  └─ ...other fields
```

### Comparison

| Aspect | SOL | SPL Token |
|--------|-----|-----------|
| **Storage** | In account.lamports | In account.data |
| **Program** | System Program (built-in) | Token Program |
| **Transfer** | System Program instruction | Token Program instruction |
| **Decimals** | 9 (1 SOL = 1B lamports) | Variable (USDC = 6) |
| **Mint** | Built-in | Configurable |
| **Freezing** | N/A | Possible (freeze authority) |

### Why SPL Tokens?

```
Use cases:
├─ Stablecoins (USDC, USDT)
├─ Project tokens (Cope, RAY)
├─ NFTs (each is unique SPL token)
├─ Utility tokens
└─ Wrapped tokens (Wrapped Bitcoin)

Benefits:
├─ Flexible supply
├─ Custom rules (freeze, burn)
├─ Multiple tokens, one standard
└─ Composable (DEX, lending, etc)
```

---

## Mint and Authorities {#mint}

### What is a Mint?

**A Mint is the definition of a token.**

```
Mint Account:
├─ Owner: Token Program
├─ Data:
│  ├─ supply (total tokens issued)
│  ├─ decimals (6 for USDC, 9 for SOL-like)
│  ├─ mint_authority (who can create tokens)
│  ├─ freeze_authority (who can freeze)
│  └─ is_initialized (true)
└─ Writable: Only by Token Program

Example (USDC Mint):
├─ supply: 43,000,000,000 (43 billion USDC)
├─ decimals: 6
├─ mint_authority: Circle (can issue more)
├─ freeze_authority: Circle (can freeze accounts)
└─ mint_address: EPjFWaJy47gIdJtiEunGoMaxuS1yGqjNzSZThLgMjNj
```

### Mint Authority

```
What it does:
├─ Can create (mint) new tokens
└─ Increases total supply

Example process:
1. Circle (mint authority) issues 1,000,000 USDC
2. Token Program creates tokens
3. supply: 42 billion → 42,000,001 billion
4. Tokens sent to Circle's account

Risks:
├─ If private key leaked: unlimited inflation
├─ If revoked: no more tokens can be minted
└─ Usually: Centralized authority
```

### Freeze Authority

```
What it does:
├─ Can freeze (lock) token accounts
├─ Frozen account: cannot transfer tokens
└─ Can unfreeze

Example:
1. Account has 1000 USDC
2. Freeze authority freezes the account
3. Owner cannot transfer (blocked)
4. Freeze authority unfreezes
5. Can transfer again

Use cases:
├─ Compliance (freeze sanctioned accounts)
├─ Cooling-off periods
├─ Security (temporarily freeze if hacked)
└─ Custom logic
```

### Ownership of Authorities

```
Scenario 1: Centralized Stablecoin (USDC)
├─ Mint Authority: Circle (company)
├─ Freeze Authority: Circle
└─ Trust in: Circle doesn't abuse power

Scenario 2: Decentralized Token
├─ Mint Authority: None (revoked)
└─ Freeze Authority: None (revoked)
└─ Result: Fixed supply, no freezing

Scenario 3: DAO Token
├─ Mint Authority: Treasury (multisig)
├─ Freeze Authority: None
└─ Community controls minting via vote
```

---

## End-to-End Examples {#examples}

### Example 1: Transfer SOL from Alice to Bob

**Scenario:** Alice sends 5 SOL to Bob

#### ASCII Diagram: Seed to Address

```
ALICE'S WALLET:

seed phrase:
"abandon ability able about above absent absolute abuse..."
        ↓
    (BIP39)
        ↓
  master seed
 (64 bytes)
        ↓
   (BIP44)
        ↓
private key
(32 bytes)
 [secret!]
        ↓
  (Ed25519)
        ↓
public key
(32 bytes)
        ↓
[Base58]
        ↓
   Address
"8e74fbE1NHhbw6X9DQ2wE6WbH7NvqvJ2C1VFhcQPg8J2"
```

#### Step-by-Step: SOL Transfer

```
BEFORE:
├─ Alice: 10 SOL
└─ Bob: 2 SOL

STEP 1: Alice creates transaction
├─ TO: System Program
├─ Accounts: [alice, bob]
├─ Instruction: Transfer 5 SOL
└─ Amount: 5,000,000,000 lamports

STEP 2: Sign transaction
├─ Serialize message (accounts, blockhash, instructions)
├─ Sign with Alice's private key
└─ Get 64-byte signature

STEP 3: Submit to network
├─ Send to RPC node
├─ Broadcast to network
└─ Enter mempool

STEP 4: Leader includes in block
├─ Order: Put with other transactions
├─ Leader creates block
└─ Broadcast via Turbine

STEP 5: Execution (Sealevel)
├─ Load Alice's account (10 SOL)
├─ Verify Alice has ≥ 5 SOL (YES)
├─ Alice: 10 - 5 - 0.000005 (fee) = 4.999995 SOL
├─ Bob: 2 + 5 = 7 SOL
└─ Write accounts to ledger

STEP 6: Validation (Tower BFT)
├─ Validators verify block
├─ Vote for block
└─ Accumulate votes

STEP 7: Finality (~6.4 seconds)
├─ 2/3+ of validators voted
└─ ✓ Transaction final

AFTER:
├─ Alice: 4.999995 SOL (5 sent + fee)
└─ Bob: 7 SOL
```

#### Transaction Structure

```
Transaction {
  message: {
    header: {
      num_required_signers: 1,      // Alice signs
      num_readonly_signers: 0,
      num_readonly_non_signers: 0
    },
    account_keys: [
      "8e74fbE1...",  // Alice (signer)
      "7uGhb8Ft..."   // Bob (receiver)
    ],
    recent_blockhash: "9q7RTfSe5...",
    instructions: [
      {
        program_id: "11111111111111111111111111111111",  // System Program
        accounts: [
          {address: "8e74fbE1...", signer: true, writable: true},   // Alice
          {address: "7uGhb8Ft...", signer: false, writable: true}   // Bob
        ],
        data: [2, 0x80, 0x84, 0x1e, 0x00, 0x00, 0x00, 0x00]  // Transfer 5 SOL
      }
    ]
  },
  signatures: [
    "7h8a9d2k1e...w9q0p1r2s3t4u5v6w7x8y9z0a1b2c3d4e5f6g7h8i9j0k"  // Alice's signature
  ]
}
```

---

### Example 2: Transfer Tokens with Durable Nonce

**Scenario:** Alice wants to transfer 100 USDC to Bob in 1 hour, but will sign now. Use durable nonce to avoid blockhash expiration.

#### ASCII Diagram: Durable Nonce Flow

```
ALICE'S WORKFLOW:

HOUR 0 (NOW):
├─ Alice: "I want to send USDC in 1 hour"
├─ Alice creates NONCE ACCOUNT
│  ├─ Allocates new account
│  ├─ Funds with 5000 lamports (rent-exempt)
│  ├─ Initial nonce_value: 0x1a2b3c...
│  └─ Authority: Alice
│
├─ Alice creates TRANSACTION:
│  ├─ Accounts: [alice_usdc, bob_usdc, nonce_account, token_program]
│  ├─ Amount: 100 USDC
│  ├─ Uses: nonce_value = 0x1a2b3c... (instead of blockhash)
│  └─ NO expiration!
│
├─ Alice SIGNS transaction
│  ├─ Serialize with nonce_value
│  ├─ Sign with private key
│  └─ Get signature
│
└─ Saves transaction file (no rush to submit!)


HOUR 1 (LATER):
├─ Alice retrieves saved transaction
├─ Submits to network
│  ├─ Blockchain reads nonce_value
│  ├─ Checks: Is this nonce_value current?
│  ├─ YES: Nonce is still 0x1a2b3c...
│  └─ Transaction accepted!
│
├─ Validator executes:
│  ├─ Transfers 100 USDC from alice_usdc to bob_usdc
│  ├─ Advances nonce_value to 0x4d5e6f...
│  └─ Writes to ledger
│
└─ ✓ Success (nonce updated, old nonce invalid)


IF ATTEMPTED 2ND TIME:
├─ Alice tries to resubmit same TX
├─ Blockchain checks nonce_value
├─ Nonce is now 0x4d5e6f... (NOT 0x1a2b3c...)
└─ ✗ REJECTED (replay protection!)
```

#### Step-by-Step: Delayed Token Transfer

```
PREPARATION (Hour 0):

1. Create Nonce Account
   ├─ Create account: nonce_account
   ├─ Fund: 5000 lamports
   ├─ Initialize with nonce
   └─ Authority: alice_wallet

2. Build Transaction
   ├─ Program: Token Program
   ├─ Accounts:
   │  ├─ alice_usdc (source, writable)
   │  ├─ bob_usdc (destination, writable)
   │  ├─ alice (authority, signer)
   │  ├─ nonce_account (read-only)
   │  └─ System Program (for nonce)
   ├─ Instruction: Transfer 100 USDC
   └─ Message: Uses nonce_value (NO blockhash needed!)

3. Sign Transaction
   ├─ Serialize message
   ├─ Sign with Alice's private key
   └─ Save to file

EXECUTION (Hour 1):

4. Load and Submit
   ├─ Read transaction from file
   ├─ Submit to network
   └─ (No re-signing needed!)

5. Validator Executes
   ├─ Check: nonce_value current? YES
   ├─ Execute: Transfer 100 USDC
   ├─ Update: nonce_value to new random value
   └─ Write: Ledger updated

6. Finality
   ├─ Votes accumulate
   ├─ ~6.4 seconds later: Final
   └─ ✓ Success
```

#### Why This Works

```
Recent Blockhash (time-limited):
├─ Valid for: 2-3 minutes
├─ Cost: Free
├─ Use case: Normal immediate transactions
└─ Problem: Can't do delayed signing

Durable Nonce (time-unlimited):
├─ Valid for: Forever (until used)
├─ Cost: 5000 lamports for nonce account
├─ Use case: Waiting to sign (batch, approval, etc)
└─ Solution: Sign now, submit later!
```

---

## Comparison Tables {#comparisons}

### Solana vs Bitcoin vs Ethereum

| Aspect | Bitcoin | Ethereum | Solana |
|--------|---------|----------|--------|
| **Year** | 2009 | 2015 | 2020 |
| **Purpose** | Digital currency | Smart contracts | Fast apps |
| **Transactions/sec** | ~7 | ~15 | 400-1000+ |
| **Avg Fee** | $0.50-$5 | $2-$100 | $0.000003 |
| **Finality** | ~10 minutes | ~15 seconds | ~6.4 seconds |
| **Block time** | 10 minutes | 12-14 seconds | 400ms slots |
| **Consensus** | Proof of Work | Proof of Stake | Proof of Stake |
| **Energy** | Massive | Low | Very Low |
| **Language** | C++ | Solidity | Rust (primarily) |
| **Account model** | UTXO | Account | Account |
| **Programmability** | Limited | Full | Full |
| **State storage** | Blockchain | Contract storage | Separate accounts |
| **Gas prices** | N/A | 1-1000 gwei | 5000 lamports flat |

### Program Accounts vs Ethereum

| Aspect | Ethereum | Solana |
|--------|----------|--------|
| **Program storage** | Contract code in account | Separate executable account |
| **Data storage** | Included in contract | Separate data accounts |
| **State modification** | Direct (mapping storage) | Via account parameters |
| **Who pays storage** | Gas (per operation) | Rent (per byte per year) |
| **Program upgrade** | Can be (if written) | Optional (BPF Loader) |

### SOL vs SPL Tokens

| Aspect | SOL | SPL Token |
|--------|-----|-----------|
| **Implementation** | Built-in | Token Program |
| **Storage** | account.lamports | account.data |
| **Decimals** | 9 (1 SOL = 1B lamports) | Configurable (USDC=6) |
| **Transfer** | System Program | Token Program |
| **Mint** | Fixed (not mintable) | Configurable |
| **Freezing** | Not possible | Possible |
| **Example** | Native currency | USDC, Cope, NFTs |

---

## Common Confusions and FAQs {#confusions}

### "Programs store state, right?"

**❌ WRONG**

Programs are stateless. They don't store data. Accounts store data.

```
Ethereum: mapping → stored in contract
Solana: accounts → passed in transaction
```

### "Each program has one account?"

**❌ WRONG**

Programs can have multiple data accounts. Each account = separate state.

```
Token Program:
├─ 1 Mint account (defines token)
├─ N Token accounts (user balances)
└─ Metadata account (NFT info)
```

### "I can use the same program ID for different tokens?"

**✓ CORRECT**

Yes! Token Program has ONE program ID but manages ALL tokens.

```
Token Program (ONE ID: TokenkegQfe...):
├─ USDC (10 million accounts)
├─ Cope (1 million accounts)
├─ Wrapped Bitcoin (100k accounts)
└─ NFTs (millions of accounts)
```

### "Nonce accounts are like Ethereum nonces?"

**❌ DIFFERENT**

Ethereum nonces: per account, per transaction, auto-incrementing

Solana nonces: special account, global, manual

```
Ethereum:
├─ alice.nonce = 0
├─ First TX uses nonce=0, becomes nonce=1
└─ Can't replay (nonce=1 not valid anymore)

Solana:
├─ nonce_account.value = random_hash_1
├─ First TX uses random_hash_1
├─ Program updates to random_hash_2
└─ Can't replay (random_hash_1 not valid anymore)
```

### "Programs must be upgradeable?"

**❌ NO CHOICE NEEDED**

You can choose:
- Upgradeable: Deployed via BPF Loader (Upgradeable)
- Immutable: Deployed via BPF Loader (default)

```
Most deployed programs:
├─ Deployed immutable
└─ Cannot be updated (security guarantee)

Corporate programs:
├─ Deployed upgradeable
└─ Can be updated (flexibility)
```

### "What if program has no data account?"

**✓ VALID**

Program can read without modifying.

```
Example (Staking Info):
├─ Program executes
├─ Reads: Validator info (immutable)
├─ Calculation: Rewards due
├─ Returns: Result
└─ No data accounts needed
```

### "Is CPI slower?"

**✓ YES, BUT STANDARD**

Each program call costs computational time, but it's the normal pattern.

```
Cost:
├─ Direct SOL transfer: 5000 lamports
├─ Token transfer (CPI): 5000 lamports + compute
└─ Complex DEX swap (multiple CPI): 5000 lamports + more compute
```

### "Can I call private functions in another program?"

**❌ NO**

All programs are public. You call by instruction discriminator (first bytes of instruction data).

```
What you can do:
├─ Call public functions (expose via discriminator)
└─ Pass required accounts

What you can't do:
├─ Call private functions (don't exist)
├─ Bypass authority checks
└─ Prevent CPI call
```

### "My account's private key was leaked. What happens?"

**☠️ HACKED**

Anyone can:
- Transfer all SOL
- Transfer all tokens
- Sign transactions
- Empty everything

**Solution:**
- Create new account immediately
- Transfer remaining funds
- Never reuse as signing account

**Prevention:**
- Use hardware wallet
- Never paste onto internet
- Use multisig for large amounts

### "Why different address for each token?"

**By design:**

```
Reason: Token Program owns accounts, not accounts themselves

Ethereum:
├─ Your wallet = 1 address
├─ Balance of token X = mapping at address
└─ All tokens in 1 address

Solana:
├─ Your wallet = 1 address
├─ Token X account = different address
├─ Token Y account = different address
└─ Different accounts for different tokens

Benefit:
├─ Accounts pay for own rent
├─ Opt-in model (no spam tokens)
└─ Token Program controls all

Downside:
├─ Lookup required (what's your USDC account?)
└─ Solution: ATA (deterministic derivation)
```

---

## Glossary {#glossary}

| Term | Definition |
|------|-----------|
| **Program** | Smart contract code deployed on Solana |
| **Account** | Data container on blockchain (state storage) |
| **Lamport** | Smallest unit (1 SOL = 1 billion lamports) |
| **SOL** | Native token of Solana |
| **SPL** | Solana Program Library (custom tokens) |
| **Mint** | Account defining token (authority, decimals, supply) |
| **Token Account** | Account holding tokens (owned by Token Program) |
| **ATA** | Associated Token Account (deterministic per user/mint) |
| **PDA** | Program Derived Address (no private key) |
| **CPI** | Cross-Program Invocation (program calling program) |
| **Rent** | Storage cost for accounts (~19.1 lamports/byte/year) |
| **Rent Exempt** | Account with ≥ 2 years rent (never charged) |
| **Nonce** | Used for replay protection (recent blockhash or durable) |
| **Recent Blockhash** | Hash from 100 blocks ago (expires after 2-3 min) |
| **Durable Nonce** | Special account replacing blockhash (no expiration) |
| **BFF Loader** | Program that deploys and manages other programs |
| **Freeze Authority** | Can freeze (lock) token accounts |
| **Mint Authority** | Can create (mint) new tokens |
| **Is Writable** | Whether an account can be modified |
| **Is Signer** | Whether an account must sign transaction |

---

## Top 10 Things to Remember {#recap}

### 1. **Programs Don't Store State**
   - Programs are stateless code
   - State lives in separate accounts
   - Accounts are passed as parameters

### 2. **Accounts Are Flexible**
   - Can store any data (bytes)
   - Token Program interprets as tokens
   - Your Program interprets as you want

### 3. **Only Owner Can Modify**
   - Token Program owns USDC accounts
   - Only Token Program can modify balance
   - Program follows allowed rules

### 4. **Fees Are Flat**
   - Always 5000 lamports per transaction
   - No matter the complexity
   - Way cheaper than Ethereum gas

### 5. **Finality is Fast**
   - ~6.4 seconds on average
   - Bitcoin: 10 minutes
   - Ethereum: 15 seconds

### 6. **SPL Tokens Need ATA**
   - Each person needs account per token
   - ATA = deterministic location
   - Don't need to ask "what's your address?"

### 7. **CPI is Normal**
   - Programs calling programs is standard
   - Happens in transactions
   - Atomic (all or nothing)

### 8. **Rent is Always Needed**
   - Accounts must pay storage
   - Rent exempt = 2 years rent balance
   - Pay once, store forever

### 9. **Durable Nonce = Delayed Signing**
   - Sign now, submit 1 hour later
   - No blockhash needed
   - Costs 5000 lamports for account

### 10. **Deterministic = Replays Possible**
   - Same input = always same output
   - Can't replay same transaction
   - Recent blockhash or nonce prevents it

---

## Quick Start Checklist

- [ ] Understand programs are stateless
- [ ] Know accounts store state
- [ ] Learn only owner can modify
- [ ] Try transferring SOL
- [ ] Try transferring SPL token
- [ ] Understand ATA derivation
- [ ] Learn about CPI
- [ ] Deploy a simple program
- [ ] Use durable nonce
- [ ] Understand replay protection

