---
title: Solana Beginner Notes
date: 2025-02-25
tags: [Solana,notes]
slug: solana-beginner-notes
---

# Solana Beginner Notes

This document provides an introduction to Solana's architecture and programming model for developers and security researchers familiar with blockchain concepts. These notes are organized into four key sections:

- **Core Concepts**: Explains Solana's transaction model, accounts system, and runtime behavior
- **Solana Virtual Machine**: Details how programs execute, interact with the blockchain, and utilize syscalls
- **Common Vulnerabilities**: Highlights security considerations specific to Solana
- **Anchor**: Introduces the popular framework that simplifies Solana program development

These notes aim to bridge the gap between Ethereum and Solana development paradigms, highlighting key differences and providing practical insights for beginning Solana developers. Security researchers will find this resource valuable for understanding Solana's security model, common vulnerabilities, and attack vectors unique to Solana's account-based architecture. The coverage of bank rules, PDAs, and cross-program invocation provides essential context for auditing Solana programs.

## Core Concepts

### Transaction and Instructions

- A Solana transaction contains instructions, each instruction being a call to a program with calldata. A single instruction is equivalent to single transaction in Ethereum.
- Transactions are atomic, meaning either all instructions succeed or none do.
- This composition of multiple instructions allows some operations to be executed in a single transaction, whereas on Ethereum they would require multiple transactions without guaranteed Atomicity. Example operations include approve + DeFi operations(which led rise to Permit implementations), cross swaps, DeFi batching.

### Transaction Accounts

- Every transaction contains a list of Account public keys. The Solana runtime executes the transaction as if these are the only accounts present in the blockchain state, treating any other account as invalid.
- This accounts list should also include the unintialized new accounts if they will be referred during transaction execution.
- The requirement for upfront declaration of accounts allows the Solana runtime to parallize execution of transactions that don't share any accounts.

### Instruction

- An instruction contains:
    - `program_id`: Address of the program account
    - `accounts`: List of accounts (must be a subset of the transaction accounts list)
    - `instruction_data`: a byte array (calldata)
- Instructions specify accounts using a list of `indexes` into the transaction's accounts list.
- While executing the instruction, the runtime uses the indexes and the accounts list to pass the accounts to the executed program.
- This differs from Ethereum, where programs must retrieve this information using opcodes. Solana directly passes the `program_id`, `accounts` and the `instruction_data` to the programs.

### Program entrypoint

- Every program must contain an `entrypoint` function with the following function signature. The function receives all the information passed with the instruction.

```rust
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
```

- `entrypoint` is equivalent to the `main` function in `C`/`Rust` binaries: It is the first and only function called by the Solana runtime.

At a high level, transaction execution works as follows:
- Fetch the listed accounts
- For each instruction:
    - Call the entrypoint function of the program with: `program_id`, `accounts`, and `instruction_data`.

### Signers

- Solana allows multiple accounts to sign a transaction, meaning multiple accounts can authorize a transaction. As a result, there is no concept of `msg.sender` in Solana.
- Solana provides signer information through the instruction accounts. Each account passed to the program has a `is_signer` field, which is set to true if that account's private key signed the transaction.
- Programs should use the `is_signer` field to verify that required accounts have authorized the operation.

### Writable Accounts

- Accounts that will be modified during transaction execution must be marked as writable.
- The Solana runtime ensures that only writable accounts are modified.
- Similar to `is_signer`, the passed accounts have an `is_writable` field. This field is true if the account is marked writable in the transaction.

The `is_signer` and `is_writable` are per transaction. An account cannot be a  signer in one instruction and not a signer in a different instruction of the same transaction.

### Transactin Fees

- Solana charges fees solely based on number of signatures. There's a consensus parameter, `lamports_per_signature`. The fee for a transaction is `lamports_per_signature` * `number of signatures`
- `1 SOL` = `10**9 lamports`
- Fee Payer: Every transaction must have one writable signer account. The fee amount is deducted from the first signer-writable account in the transaction's accounts list.
- Solana does not charge per-opcode gas/fee. Every transaction is allocated a fixed compute budget. Each opcode consumes some compute units, and the transaction fails if execution exceeds the compute budget limit.
- There's also a limit on the total size of all the accounts loaded by the transaction: 64 MiB.

### Rent

- Each account must pay rent for the space it occupies in the blockchain state. This is how Solana charges for storage.
- Initially, Solana deducted rent from every account's balance for every epoch. If an account's balance reached zero, the account was removed from the blockchain state.
- This approach was later changed. Now, every account must maintain a minimum balance equal to two years' worth of rent for that account.
- All new accounts must statisfy this minimum balance requirement; these accounts are considered `rent-exempt`.
- No rent is deducted from `rent-exempt` accounts. The Solana runtime prevents any transaction from reducing an account's balance below the minimum requirement.
- The balance of a `rent-exempt` account must either be greater than or equal to the minimum balance, or zero. When the balance becomes zero, the account is removed from the blockchain state.
- There may still be older `rent-paying` accounts from before this change. Rent is still deducted from these accounts. These older accounts automatically become `rent-exempt` if they reach the minimum balance. Only accounts with  `0 < balance < minimum_balance` can be rent-paying. All other accounts must be `rent-exempt`, and once an account becomes `rent-exempt`, it cannot revert to `rent-paying`.

### Account and AccountInfo

Every Solana account has the following fields:

```rust
#[repr(C)]
pub struct Account {
    pub lamports: u64,
    pub data: Vec<u8>,
    pub owner: Pubkey,
    pub executable: bool,
    pub rent_epoch: Epoch,
}
```
- `lamports` is acount balance in lamports. 1 SOL = 10^9 lamports.
- `executable` is true if the account is a Program and can be called with an instruction.
- `rent-epoch` is the epoch number when rent was last collected for this account. If the account becomes `rent-exempt`, this value is set to `u64::MAX`, which the Solana runtime uses as a marker for rent-exempt accounts.
- Account `owner` is set to a Program address. The owner program has privileges to modify the account and controls its state.
    - For example, only the owner can deduct from the account's balance.
- `data` is the persistent storage of the account. Every account can store up to 10MB of data in this field.

`AccountInfo` is the type passed to the program's entrypoint and contains two additional fields: `is_signer` and `is_writable`.

### Contract Storage

- Ethereum accounts have `code` and `storage`. Each contract's bytecode is in the code field and `storage` is used for persistent state.
    - There's an implicit dependency between a contract address and its storage. Storage is available because code is stored separately.
    - The EVM implicity prevents contracts from modifying storage at different address (by not providing any such opcodes).
- Solana is different. Data can only be stored in an account's `data` field. Therefore, program bytecode and state exist in separate accounts.
- The Solana runtime provides a way for programs to store data that only they can can modify through the Account `owner` field.
- The `owner` field serves as access control for the account's data.
    - Solana runtime guarantees that only the owner program can modify an account's data field.
- A program can use multiple accounts for storage by setting their owner field to its address.
- A program's state/storage consists of the `.data` field of all accounts owned by the program.

### Native Programs and User programs

- Solana blockchain includes programs that are part of the Solana runtime/core and perform native operations. These are called Native programs.
- Native programs are essential for validators operations.

- User programs do not have additional privileges. The Solana blockchain can operate without the presence of User programs but not without Native programs.
- For example, the System Program is a Native program essential for creating new accounts.

### Native Loader and BPF Loader

- The owner of a program is a `loader`.
- For normal accounts, the owner determins which program can modify the account.
- For program accounts, the owner dictates how that particular program should be executed.
- There are two main loaders: Native Loader and BPF Loader(s).
- All native programs are owned by Native Loader. Programs owned by Native Loader are built into the Node and do not execute in the Solana VM.
- All user programs are owned by BPF Loader. There are multiple versions of BPF Loaders:
    1. BPF Loader (deprecated)
    2. BPF Loader 2
    3. BPF Loader upgradeable
    4. BPF Loader v4 (upcoming for program-runtime v2)
- Programs owned by BPF Loader 2 are immutable, while programs owned by the upgradeable can be upgraded. Upgradeable programs can be made immutable, but once immutable, they cannot be made upgradeable again.
- The BPF Loaders are also native programs, so these Loader accounts are in turn owned by the Native Loader.
  
### Bank Rules

- Bank rules are a set of requirements that transaction must satisfy to be valid. These rules dictate acceptable modifications to accounts (valid state transitions).
- These rules guarantee the security properties of the blockchain. 

1. An Account's `lamports`, `owner`, and `data` can be modified if and only if:
    - The account is Writable
    - The account is **not** executable 
2. An Account's lamports can only be deducted by the Account's owner.
    - The balance of a program is the sum of lamports of all accounts owned by that program (+ balance of program PDAs owned by System).
3. Any program can add lamports to any account as long as the same amount is deducted from other accounts: The sum of account balances before execution must equal the sum after execution.
4. The `owner` of an account can be changed by a program if and only if:
    - The program is the current owner: Only the owner can assign a new owner.
    - The account `data` is empty or contains only **null** bytes
5. An Account's `data` can be resized/modified by a Program if and only if
    - The account is owned by the currently executing program.
6. An Account's `data` c be resized to a length > MAX_PERMITTED_DATA_LENGTH (`10 MiB`). `data.len()` is always `<= 10 MiB`.
7. The total resize of all Account's `data` in a transaction must be less than the per-transaction maximum of `20 MiB`.
    - For example, if Account1 is resized from `2 MiB` to `10 MiB` and Account2 from `3 MiB` to `10 MiB`, then no other account can resize by more than `5 MiB` during the transaction execution.

These rules are verified and asserted:

- Before a call to an external contract
- After returning from the external contract
- At the end of the execution of an instruction
- At the end of the execution of the transaction (same as validations performed at the end of the last instruction)

This validation at multiple points is necessary so that no program sees invalid state regardless of when it's called.These are points during transaction execution where **execution switches to a different program**. This prevents malicious programs from making invalid state changes.

Example exploit scenario if these rules were only validated at the end of the transaction:
    - A malicious program could change state variables of a DEX and call the DEX with this invalid state. The DEX would use the invalid state and lose tokens.

Changing the owner requires the `data` be `null` bytes (of any size). Otherwise, a malicious program could create an account, set `data` to favorable values, and changes its owner to the victim program. The victim program would consider the account to be part of its state and trust these values.
    - For example, a DEX program might store Liquidator positions in an account. A malicious program could create an account with data containing favourable liquidator positions and change the owner of the account to the DEX. The DEX would use this state and lose tokens.
    - Essentially, a malicious program could set new state variables for another program.

The `impl` of the `BorrowedAccount` implements these checks in `sdk/src/transaction_context.rs`.

There are also rules for modifying the executable field of an account:
- The account must be writable
- Only the owner program can make an account executable
- The account must be rent-exempt
- Once an account is executable, it cannot be made non-executable.

### System Program

- Every new unintialized account is by default owned by the **System Program**.
- The System Program must be called to change the account owner for use with other programs.
- All non-state accounts are owned by System Program.

### Durable Nonces

Solana uses Recent Blockhashes for replay protection. A transaction must contain a blockhash value from a recent block (~150). The transaction referencing blockY only be processed in blockX if `blockX <= (blockY + 150)`. This protects from transaction replay, as validators only need to check for the transaction's ID in the last 150 blocks.

However, this approach doesn't support offline signing and similar features, since transactions must be constructed just before being sent to the network.

The solution is to use Durable Nonce accounts.

A user can create a Nonce account under their authority. This account stores a blockhash that can be referenced by transactions. When a transaction uses a blockhash from a Nonce account, the blockhash in the Nonce account gets updated to the most recent blockhash, ensuring uniqueness of processed transactions.

Nonce accounts are owned by the System Program, which handles their creation, updating, and deletion.

A user can use either a recent blockhash or a blockhash from a nonce account. The Solana runtime first checks if a transaction's blockhash exists in the last 150 blocks. If not found, it looks for an instruction calling the `AdvanceNonceAccount` function of the System Program. This instruction indicates that the user wants to use a Nonce account, and the runtime uses the blockhash from that account.

By requiring the presence of `AdvanceNonceAccount`, the Solana runtime ensures the nonce is advanced with every use.

The `AdvanceNonceAccount` function also verifies that the blockhash in the nonce account is not from a recent block. This is because `AdvanceNonceAccount` always sets the blockhash to the recent one, and if the blockhash is already recent, the nonce wouldn't change, potentially allowing replay attacks.

Implications: There can only be one transaction in a block that uses a given Nonce account.

The Solana runtime requires that `AdvanceNonceAccount` is always the first instruction in the transaction.

### Compute Budget Program

Solana has a Compute Budget Program that can be used to increase the compute-unit limit for transactions and set prioritization fees.

Calling the `SetComputeUnitLimit` instruction of the ComputeBudget Program sets the compute limit, while calling `SetComputeUnitPrice` sets the prioritization fee.

PrioritizationFee = ComputeUnitLimit * ComputeUnitPrice

ComputeUnitPrice is denominated in micro lamports.


### Address Lookup Tables

Solana's transaction size is limited to 1232 bytes. Because every account's publiv key must be listed in the transaction, the number of accounts that can be loaded by a transaction is also limited (approximately 35 in best case).

Solana introduced Address Lookup Tables to allow for the use of more accounts. Address Lookup Table is an account that contains a list of account addresses.

A transaction can include an address-lookup-table account address and indexes into that list. The Solana runtime fetches these addresses and makes the accounts available to the transaction.

- A new transaction type is introduced for the use of address-lookup-tables. This transaction type allows addition of address-lookup-table at the end of the fields in the old transaction type.
- Transactions cannot index addresses that are inserted into the table in the same block.
- Addresses loaded from the table cannot be signers. The Solana runtime requires the signer public keys be included directly in the transaction.

For more information, see: https://docs.solanalabs.com/proposals/versioned-transactions

## Solana Virtual Machine

### Solana RBPF

Solana uses EBPF instruction set for its programs. Solana RBPF VM is an implementation for running EBPF programs in user-space. 

Solana programs are object files (.so) compiled for the RBPF VM.

EBPF is a general computing instruction set not specifically built for blockchain usage. Solana benefits from choosing RBPF for its programs as it doesn't have to invest in building compilers, developer toolchains, and VMs from the ground up. 

Using EBPF allowed development of Solana programs in any language that can be compiled to EBPF. LLVM backend supports targeting to EBPF. As a result, any language that supports LLVM backend can be used, including Rust and C.

### Sysvars and Syscalls

Because EBPF only has general computing instructions, it lacks instructions for operations expected in a blockchain VM. Solana uses Sysvars and Syscalls to provide that functionality to the programs.

It is easier to compare with EVM opcodes to see how Solana uses sysvars and syscalls.

Aside from general computing instructions (Arithmetic, Comparison, Memory), EVM has:
1. Storage opcodes (sload, ..)
2. Address opcode
3. Calldata related instructions
4. Info of accounts (BALANCE, CODESIZE, EXTCODECOPY, ...)
5. Info about the blockchain parameters (BLOCKHASH, TIMESTAMP, NUMBER, ...)
6. External calls and return data (CALL, DELEGATECALL, ..., RETURNDATASIZE, RETURNDATACOPY, ...)
7. Keccak256
8. Logging
9. GAS
 

#### Entrypoint Args

1. Storage opcodes: There's no need for separate storage opcodes in Solana. Storage is account's `data` field and each account is passed to the program. Solana runtime writes the changes to the blockchain state at the end of the transaction, making them persistent.
2. Address: Address of the current contract is provided as an argument to the entrypoint function `program_id`.
3. Calldata: Solana runtime passes the calldata as an argument to the entrypoint function `instruction_data`. Solana runtime copies these arguments into the program's memory and invokes the entrypoint function.
4. Info of accounts: Accounts are also passed to the entrypoint as an argument. It's a matter of reading from memory to compute these values.

Remember that EVM doesn't pass any arguments to the contract; it just starts executing from the first opcode. The contract has to include opcodes to fetch these values when required.

#### Info about the blockchain parameters

Sysvars provide information about the blockchain cluster to programs. Sysvars are accounts which contain all this information in their `data` field.

- Every Sysvar account is owned by the Sysvar program.
- Solana runtime updates the data field of these sysvar accounts with the most recent information before executing a transaction.
- If a program requires information, such as the timestamp, the user can provide the corresponding sysvar account in the instruction, and the program can read from the account data without needing additional opcodes.

Most common Sysvar accounts are:
1. Clock: Contains the slot number and block timestamp.
2. Rent: Contains information about the rent payable per byte. Mostly used to calculate the rent-exempt amount so programs can make their accounts rent-exempt.
3. Instructions: Contains information about the raw transaction. A program can read which programs are called in other instructions and the arguments passed to them.

#### External calls, return data, keccak256, logging

Syscalls provide these functionalities. Syscalls are functions that are part of the validator and available to programs. Programs can call a syscall just as they would call their own fuction.

Implementation:
- The Solana SDK has wrappers for each syscall function. The compiled program will have a RBPF instruction for calling internal functions with the syscall function's signature.
- While initializing the VM, Solana fills the Function registry for the program with Rust function pointers. The Function registry is a map where Solana runtime adds each syscall function signature and the function pointer that performs the syscall's operation.
- When a `call*` instruction calls a syscall, the execution is redirected to the function of the syscall which is part of the validator.

When a syscall is executed, execution context switches to Solana runtime. The Solana runtime performs the required operation and returns the context back to program. This is similar to how syscalls between user-space applications and kernel work in operating systems.

The `agave/sdk/program/src/syscalls/definition.rs` contains the list of all supported syscalls.

Notable syscalls include:

- try_find_program_address and create_program_address (PDAs)
- log_* syscalls
- remaining_compute_units (GAS)
- invoke_* (external calls)
- set_return_data and get_return_data
- cryptographic functions:
    - keccak256, sha256, blake3
    - Elliptic: EC point validation, multiplication, addition, pairing, big_mod
    - ZK: poseidon, alt_bn128 operations

### Precompiles

Ethereum adds precompiles to provide standardized support for additional functionalities to EVM. Because Syscalls achieve that for Solana, most of the Ethereum precompiles are available as syscalls in Solana.

Solana has precompiles as well, allowing them to be called using a transaction/instruction. Syscalls are just functions and do not process instructions.

Solana has two precompiles:

- secp256k1_program
- ed25519_program

Precompiles have `verify` function that verifies the signature over the `instruction_data`.

### External Calls (CPI)

External calls are referred to as Cross Program Invocation(CPI) in Solana. A program can invoke an external program by providing the arguments for the entrypoint: External program's id, Vec of AccountMeta, and instruction data.

```rust
pub struct Instruction {
    pub program_id: Pubkey,
    pub accounts: Vec<AccountMeta>,
    pub data: Vec<u8>,
}

#[repr(C)]
pub struct AccountMeta {
    pub pubkey: Pubkey,
    pub is_signer: bool,
    pub is_writable: bool,
}
```

The accounts passed to the external program must be included in the accounts arguments of the current program. The passed-in account can be marked as signer or writable if and only if it is a signer or writable respectively in the current caller program.

The `invoke` syscall is used to execute the external call. The Solana runtime performs the required validations:
- Validate Bank Rules
- Validate that the external program account and the passed in accounts are available to the current program
- Ensure only the accounts which are signers in the current program have `is_signer` set to true in the AccountMeta. Same for `is_writable`.

After performing these validations, execution is passed to the called program.

#### Extension of signer privileges

The ability of a program to extend signer privileges to another program is a powerful mechanism. 

It allows the program to perform any operation the signed user would be able to do with a transaction.

For example, if there's a DEX program and a user signs the `swap` transaction, then the DEX program can call the Token program and use the user's signature to perform transfer operations. The program simply sets `is_signer` to true for the user's `AccountMeta` and the transfer operation succeeds without needing an `Approve` operation for swapping.

As a result, when a user signs a transaction, they are effectively giving full privileges to the program to use their signature/authority.

If a program is malicious and user signed a transaction calling that program, the user could lose all their tokens and anything under their account's control.

The user doesn't have to call the malicious program directly; if a trusted program uses that malicious program and extends the user's signature, the same outcome can occur.

### Program Derived Address (PDAs)

> TLDR;
> - While EOAs authenticate using cryptographic signatures, programs need a way to prove authority over accounts. PDAs serve this purpose.
> - A PDA is an account adress from a sha256 of `program_id || bytearray (&[u8])`. These addresses can be "signed" by the program with address `program_id`.
> - The `invoke_signed` syscall works like `invoke` but takes an additional argument: a list of bytearrays `(&[&[u8]])`. The Solana runtime generates a PDA for each bytearray and allows these PDAs to be signers in the CPI.
> - This mechanism allows programs to sign and prove authority over accounts derived from their address, enabling them to perform operations that EOAs can do.

Solana blockchain has two kinds of actors: EOA and programs. Solana uses cryptographic signatures as an authentication mechanism for EOA accounts. These accounts have addresses derived from public keys, allowing users to sign messages with private keys to prove authority over given addresses. This enables users to own assets, tokens, and interact with programs.

Programs, however, need their own authentication mechanism to have authority over accounts. This differs from chains like Ethereum, where the model is straightforward: there's only one account associated with a call (`msg.sender`), which can be set to the contract's address by the Ethereum runtime. This doesn't work for Solana because of its Account model, where multiple accounts are passed in a call/instruction without a single msg.sender concept. PDAs provide the solution.

When making a call with the `invoke_signed` syscall, Solana runtime allows a program to provide a bytearray (&[u8]). Before executing the called program, the runtime calculates a 32-byte hash of `program_id || bytearray`. It then sets the account whose address equals this 32-byte hash as a signer. The called program sees this derived address as a signer. These addresses are called Program Derived Addresses (PDAs), and the bytearray is treated as a concatenation of seeds `(bytearray = seed[0] || seed[1] || seed[2] …)`.

By including the program_id in the derivation, the runtime ensures that only the currently executing program that derived the address can change an account from non-signer to signer.

PDAs therefore provide signing authority for programs. While a private key has signing authority over accounts with addresses derived from public keys, a program has signing authority over all addresses that are hashes of `program_id || bytearray` (with some exceptions).

The primary use of PDAs is to sign and prove authority. This signing privilege allows programs to create accounts by directly interacting with the System program instead of relying on users. Consequently, using PDAs to store state has become common practice – a secondary use case.

To prevent users with private keys from having authority over a PDA, Solana runtime doesn't allow programs to sign addresses derived from `program_id || bytearray` that could have private keys. The runtime ensures this by using the fact that only valid public keys can have private keys, and valid public keys are points on the ed25519 curve. Since Solana's address derivation can calculate public keys from addresses, it's easy to check if a public key is on the curve. Before making a PDA signer, the runtime verifies that the hash of `program_id || bytearray` is not a point on the curve.

Programs must ensure their bytearrays produce valid PDAs. For this, Solana provides the `find_program_address` syscall, which takes the program_id and seeds to calculate a `u8` value (called the "bump") that, when used as a suffix, produces a valid PDA. The hash of `program_id || seed[0] || .. || seed[-1] || [u8]` must be a valid PDA. There might be multiple `u8` values that work, but `find_program_address` returns the maximum – the canonical bump. Having criteria for the canonical bump allows other systems to generate the same PDA for given seeds.

Recalculating the bump value for every PDA calculation is inefficient. Programs can use the `create_program_address` syscall to calculate a PDA address when they already know the bump. This syscall is similar to `find_program_address` but doesn't add a suffix, expecting the seeds to result in a valid PDA.


## Common Vulnerabilities

The following vulnerabilities are specific to Solana programs and do not include language-based issues such as rounding, overflow/underflow, panics, etc.

These issues mostly involve inadequate validation of input accounts. Since everything is provided by the end user, including program storage (accounts owned by programs), it is crucial to ensure correct accounts are passed to the program.

1. **Missing Signer Check**: Program fails to verify that required accounts have signed the transaction.
2. **Missing Owner Check**: Program doesn't verify the intended owner of an account, allowing attackers to craft data in their own account and provide the attacker's account instead.
3. **Type Cosplay**:
    - Program assumes the passed account contains the expected type of data when it does not. A program might have two types of data structures, and an attacker might supply an account with the first type when the program expects the second type.
    - In other words, ownership checks ensures the program is reading its own storage, while type checks ensure the program is reading the correct state variable.
4. **Arbitrary CPI**: Program calls another program provided by the user without proper validations. The malicious program can exploit the passed-in signatures.
5. **Account Revival attack**:
    - Programs incorrectly close accounts. Programs typically transfer the entire balance of a state account assuming the runtime will delete this account. However, runtime doesn't delete the account until after the transaction completes.
    - Another instruction could increase the balance of this "closed" account, preventing the runtime from deleting it.
    - The recommended approach is to clear the data and transfer ownership to the SystemProgram.
6. **PDA Sharing**: A program uses same PDA for interacting with multiple programs. Best practice is to use different PDAs for different purposes. 
7. **Reinitialization attacks**: Program fails to check if it has already initialized state in an account and reinitializes variables stored in that account.
8. **Seed Collision**:
    - The bytearray used for PDAs is generally constructed from a list of seeds. Each seed is a bytearray. Solana APIs that use bytearray to construct PDAs use the type `&[&[u8]]`. The bytearray is simply a concatenation of these seeds; if seeds are `["A", "B"]`, the bytearray used for the PDA is `["AB"]`.
    - Because of this API, some programers incorrectly assume that seeds `["AB", "C"]` and `["A", "BC"]` result in different accounts when they actually result in the same PDA. 
    - If a program assumes these two would create different accounts, it will not work as intended and may exhibit undefined behavior.
    - This issue is more likely when seeds depend on user inputs.

### Notable Hacks

- **Wormhole hack**: The `verify` function used data from the `instructions` sysvar without checking that the correct account was passed.
    - https://wormholecrypto.medium.com/wormhole-incident-report-02-02-22-ad9b8f21eec6
- **Reported Bug in Jet Lending protocol**:
    - Jet stored LP tokens in PDA accounts derived from user's address. All tokens are under the control of th same account `market_authority` (another PDA).
    - The `withdraw` function didn't validate the LP token's PDA, allowing attacker to provide the LP account of any user.
    - Because the same authority was used for all accounts, the program signed all burn instructios using the `market_authority` and they succeeded.
    - Attackers could burn any user's LP tokens and steal the Position tokens.
    - https://www.sec3.dev/blog/on-a-20m-bug-in-jet-protocol

## Anchor

Anchor is a library that provides utilities to simplify writing Solana programs.

Rust and C are general purpose programming languages that lack specific features for common Solana programming tasks:

- Function dispatching
- Deserialization of instruction/function arguments (calldata)
- Instruction account management
- Account data serialization/deserialization
- Common account validations

Anchor addresses these needs using Rust macros.

### Function Dispatcher and Instruction Data Deserialization

Anchor's `#[program]` macro can be applied to a Rust `mod` definition:

```rust
#[program]
mod hello_anchor {
    pub fn FuncA(ctx: Context<FaAccounts>, data: Data) -> Result<()> {
        ...
    }
    
    pub fn FuncB(ctx: Context<FbAccounts>, s: u64) -> Result<()> {
        ...
    }
    
    ...
}
```

Functions present in this module are considered the program's "public functions". Anchor automatically generates the function dispatcher and deserialization code for arguments.

### Instruction Accounts

Anchor allows defining all accounts expected by an instruction in a struct:

```rust
#[derive(Accounts)]
pub struct FaAccounts<'info> {
    pub acc_1: AccountInfo<'info>
    pub acc_2: AccountInfo<...>,
    pub acc_3: ....,
    ....
}
```

The struct must have `#[derive(Accounts)]` attribute.

Every function in the `#[program]` module must have a first argument of type `Context<T>`, where `T` is an accounts struct with `#[derive(Accounts)]`, for example: `FuncA(ctx: Context<FaAccounts>, ...)`. The accounts are accessible to the program through `ctx.accounts.acc_1`, `ctx.accounts.acc_2`, etc.

Anchor first dispatches to the appropriate function wrapper based on the function signature. This wrapper deserializes the arguments from the instruction data and then constructs the accounts struct using the `accounts` passed with the instruction.

### Account Serialization and Deserialization

Anchor enables defining the data structure stored in an account using a struct:  

```rust
#[account]
pub struct AccountTypeX {
    data: u64,
    var: u8,
    i: bool,
    j: Vec<u8>,
}
```

The struct must have the `#[account]`, which indicates it's stored in an account owned by the program.

This struct should be used with Anchor's `Account<T>` type:

```rust
#[derive(Accounts)]
pub struct FaAccounts<'info> {
    pub acc_1: Account<'info, AccountTypeX>,
    ... 
```

`Account<'info, AccountTypeX>` tells Anchor that `acc_1` (the first account) stores data of type `AccountTypeX`. Anchor generates deserialization code for the struct and creates an instance of `AccountTypeX`

Functions taking `Context<FaAccounts>` can access the deserialized data directly:
```rust
let acc1 = ctx.accounts.acc_1;

acc1.data;
acc1.var;
acc1.i;
```

Anchor also generates serialization code. Any changes made to `ctx.accounts.acc_1` are serialized and written back to the account data.

### Common Account Validations

Anchor allows declaring constraints on each account in the Accounts struct:

```rust
#[derive(Accounts)]
pub struct FaAccounts<'info> {
    #[account(...)]
    pub acc_1: Account<'info, AccountTypeX> // type of the account,
    #[account(...)]
    pub acc_2: SystemAccount<'info>,
```

Anchor performs validations on the instruction accounts using:

1. The type of account data (e.g., `AccountTypeX`)
2. The type of account field (e.g., `Account<...>`)
3. Explicit constraints declared using `#[account(...)]` attributes

#### Type of account data

For `#[account]` structs (stored in an account), Anchor adds a discriminator field to the struct.

For example, an account storing `AccountTypeX` will include the discriminator along with the struct data. The discrimintor is unique for each struct type--it's the first 8 bytes of hash of the struct name: `hash("AccountTypeX")[:8]`

When Anchor deserializes the account, it verifies that the discriminator matches the expected value, preventing type cosplay attacks.

Anchor requires the program to define an `ID` variable storing the the program's address. Anchor associates every `#[account]` struct with this `ID` as its owner.

When an account type is defined as a state account (`acc_1: Account<'info, AccountTypeX>`), Anchor verifies the account's owner is the program.

If `AccountTypeX` is a struct from another Anchor program, Anchor checks that `acc_1`'s owner is that program, preventing missing owner checks.

#### Type of Account Field

The following can be the type of a field defined in an `#[Accounts(...)]` struct (i.e., type of `acc_1`):

```rust
pub enum Ty {
  AccountInfo,
  UncheckedAccount,
  AccountLoader(AccountLoaderTy),
  Sysvar(SysvarTy),
  Account(AccountTy),
  Program(ProgramTy),
  Interface(InterfaceTy),
  InterfaceAccount(InterfaceAccountTy),
  Signer,
  SystemAccount,
  ProgramData,
}
```

- `AccountInfo`, `UncheckedAccount` - No checks on the account
- `AccountLoader`, `Account` - Represents program state; Checks discriminant and owner
- `Sysvar(SysvarTy)` - A sysvar account; verifies the account pubkey for the sysvar
- `Program(ProgramTy)`, `Interface(InterfaceTy)` - A program account (for Interface, one of multiple programs)
- `InterfaceAccount` - State of one of the programs
- `Signer` - Account must sign the transaction
- `SystemAccount` - Account owner must be the System program
- `ProgramData` - Account storing data of a program owned by `UpgradeableBPFLoader`

#### Account Attribute Constraints

Every account in the `#[Accounts]` can have multiple `#[account(constraint)]` attributes.

```rust
#[derive(Accounts)]
pub struct FaAccounts<'info> {
    #[account(address = ...)]
    #[account(mut)]
    pub acc_1: Account<'info, AccountTypeX> // type of the account,
    #[account(...)]
    pub acc_2: SystemAccount<'info>,
```

Anchor ensures all this validations are met by the passed accounts before calling the function.

The full list of constraints is available in the [Anchor documentation](https://docs.rs/anchor-lang/latest/anchor_lang/derive.Accounts.html).

### Function Modifier

Anchor has `access_control` macro that can be applied to functions in th `#[program]` module.

Anchor executes the given modifier before executing the function:

```rust
#[program]
mod hello_anchor {
    #[access_control(check_this(ctx, data)))]
    pub fn FuncA(ctx: Context<FaAccounts>, data: Data) -> Result<()> {
        ...
    }
```

`check_this` is executed before `FuncA`

### Account Reloading Vulnerability

When Anchor deserializes account data, it creates a struct. Two versions of the data exist: the raw `account_info.data` and the Anchor-created struct. All data access resolves to the Anchor-created struct.

When an account is passed to a different program via CPI, modifications by the called program are recorded in `account_info.data` but not in the Anchor-created struct. If the program continues using that struct without reloading the account, it won't see changes made by the called program.

The `Account<...>` type has a reload function that deserializes `account_info.data` and updates the struct.

This issue may apply to non-Anchor programs if they don't deserialize data after modifications.

