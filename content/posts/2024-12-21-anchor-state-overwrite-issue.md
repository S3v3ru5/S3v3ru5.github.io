---
title: Anchor State OverWrite Issue
date: 2024-12-21
tags: [Solana,Anchor, Vulnerability]
slug: anchor-state-overwrite-issue
---

# Anchor State Overwrite Issue

> TL;DR: Anchor's `#[derive(Accounts)]` macro creates an in-memory copy of the account data for `Account<'a, T>` accounts. It modifies the memory and then writes the changes back to the account data. If two accounts are the same, serializing the second account overwrites the changes made to the first account, leading to undefined behavior. 

## Vulnerability

Consider the following example vulnerable implementation of a token program written using the Anchor framework. The `transfer` instruction is vulnerable to `state-overwrite`: executing a self-transfer increases the caller's balance without any deductions.

```rust
#[account]
#[derive(Debug, Default, InitSpace)]
pub struct TokenHolder {
    pub authority: Pubkey,
    pub balance: u64,
}

#[derive(Accounts)]
pub struct Transfer<'info> {
    #[account(address = sender.authority)]
    pub authority: Signer<'info>,

    pub sender: Account<'info, TokenHolder>,
    pub receiver: Account<'info, TokenHolder>,
}


#[program]
pub mod vulnerable_token {
    use super::*;

    pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()>{
        require_gte!(ctx.accounts.sender.balance, amount);

        ctx.accounts.sender.balance -= amount;
        ctx.accounts.receiver.balance += amount;
        Ok(())
    }
    
    [...]
}
```

The vulnerability arises when sender and receiver reference the same account. This issue stems from how Anchor handles deserialization and serialization of account data during instruction execution.

## How Anchor Works

Anchor's expanded code for a instruction works as follows at a high-level:

1. Construct `Context`
    - Anchor creates `Context` for the instruction using the `Accounts` struct. It deserializes the `AccountInfo::data` into `T` for `Account<'a, T>` types.
    - For e.g, if account `X` is passed for `sender`, Anchor deserializes `X.data` into type `TokenHolder` and stores it in memory as `ctx.accounts.sender`.
2. Call instruction handler
    - The `transfer` function is called. Any changes made to `ctx.accounts.sender` or `ctx.accounts.receiver` are stored in memory.
3. Serialize Accounts
    - After the instruction handler finishes execution, Anchor serializes the updated accounts and writes them back to `AccountInfo::data`. The account data is persistent and hence the state changes are saved.

## Undefined behavior in the `transfer` Instruction

When the `sender` and `receiver` are the same accounts, the following sequence occurs:

1. Deserialization
    - a. `ctx.accounts.sender = deserialize(X.data)`
    - b. `ctx.accounts.receiver = deserialize(X.data)`
2. Transfer function
    - a. `ctx.accounts.sender -= amount`
    - b. `ctx.accounts.receiver += amount`
3. Serialize accounts
    - a. `X.data = serialize(ctx.accounts.sender)`
    - b. `X.data = serialize(ctx.accounts.receiver)`

During serialization, the second write (`3b`) overwrites the changes made in the first write (`3a`). This means only the balance increase for the `receiver` is recorded, allowing the caller to double their balance in a self-transfer scenario.

The serialization order is determined by the field order in the `Accounts` struct. Since `receiver` is defined after the `sender` in the `Transfer` struct, the changes to `receiver` are preserved.

## Mitigation

Ensure that no two accounts of type `Account<'a, T>` are the same to prevent the issue.

## How I Found the issue

I identified this issue in a Sherlock contest (Issue M-3 in the [report](https://audits.sherlock.xyz/contests/535/report)). Two key factors have helped me in identifying the bug:
1. Experience with Similar Bugs in Solidity
    In Solidity, issues often arise when storage variables are copied into memory, updated and written back. Anchor uses the same pattern.
2. Understanding Anchor's Internals
    Knowledge of how Anchor works internally and understanding of the root-cause of the well-known `Account Reloading` issue.
    - `Account Reloading` issue is present because Anchor keeps an in-memory copy and doesn't deserialize the `account.data` after a CPI: If CPI changes `account.data` then the program would be using out-dated in-memory copy leading to undefined behavior.

## Conclusion

Understanding the internals of frameworks like Anchor helps in identifying unique vulnerabilities. Many of these issues stem from unknown or less-documented "footguns" that other researches may overlook. In this case, Anchor's in-memory deserialization model introduces bugs like the one described.
