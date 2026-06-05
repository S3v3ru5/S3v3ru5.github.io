---
title: Property testing Solana AMM using Proptest
date: 2026-06-06
tags: [Solana,AMM]
slug: solana-property-testing
---

# Property testing Solana AMM using Proptest

Property testing verifies behavior across thousands of inputs by defining what should always be true. AMMs are ideal targets. They have clear economic properties like no arbitrage, path independence, and slippage bounds that must hold regardless of implementation. This makes it easy to generate random swap sequences, check the properties hold, and let the fuzzer find extreme states that manual testing wouldn't reach.

This post walks through the methodology I used to test a novel stableswap Solana AMM during an audit. The curve was complex enough that verifying derivations by hand wasn't practical. Instead, I used proptest to generate thousands of random swap sequences and verify the economic properties held throughout.

## The AMM

The protocol under audit was a stableswap AMM combining two approaches: a constant-sum-like invariant with virtual reserve constraints. The constant-sum component provides capital efficiency near equilibrium, while virtual reserves bound the price within a target range. This hybrid design is novel. The core invariant ties together three interdependent parameters: an amplification factor that controls curve curvature, and two virtual reserve offsets that enforce price bounds.

The swap functions implement four variants: exact-in and exact-out for both token directions. A single swap must preserve the core invariant; sequences of swaps must maintain price bounds and equilibrium properties.

The implementation included two versions of each swap function. The standard versions solve the curve equation directly and are used only inside invariant calculations. They are slow but serve as the reference. The light versions use binary search to save compute units and are the ones actually called during user swaps. The light versions also include reserve-adjustment helpers that aren't obviously necessary if the swap math is correct. Whether the two versions are equivalent isn't visible from reading the code, so verifying the production path against the reference became the primary goal.

During my time at Trail of Bits, I used Echidna to fuzz Solidity contracts. AMMs are a good target for that kind of testing: the swap functions take reserves and return amounts, and most of the relevant invariants can be expressed as assertions on inputs and outputs. Property testing reaches input and state combinations that manual testing never would, building confidence without requiring a full understanding of the derivation.

Given the complexity and the fact that light versions were used in production, I had two goals before writing tests:

- Verify light versions are equivalent to standard versions
- Verify the implementation satisfies economic properties of an AMM: no arbitrage, path independence, slippage bounds, invariant preservation

## Properties

I started with properties from AMM economics and narrowed to seven that were testable and mostly independent.

**Equivalence**: The light versions (binary search) must return the same values as standard versions (direct computation) within a precision tolerance. The light versions are used in production. If they diverge from the reference implementation, the optimization introduced errors. This was the primary goal.

**Invariant preservation**: After every swap, the curve equation must hold. Both _invariant() and _invariant_light() must return true. If the invariant ever breaks, the pool has left the valid state and every subsequent swap is operating off-curve.

**No arbitrage**: Round trip swaps should not profit the user. If traders gain by selling and buying back, they can repeat this to drain the pool, causing loss to liquidity providers. This is a fundamental security property of an AMM. By ensuring that maximum gain is zero, the test also validates that rounding direction favors the pool throughout the swap calculation.

**Input/output consistency**: If swap_x_for_y(100) returns 98, then swap_x_for_y_out(98) should return approximately 100. Users can specify swaps as "I have X tokens" (exact input) or "I want Y tokens" (exact output). These are two ways of describing the same trade. The implementations must agree for the same swap to produce the same result regardless of how it is specified.

**Path independence**: A 1000 token swap should produce the same output as ten 100 token swaps executed sequentially. For a well-behaved AMM, the output depends only on the total input and the initial state, not on how the input is divided. A violation indicates the swap function depends on execution pattern rather than just the mathematical relationship between input and reserves.

**Slippage bounds**: sell(2x) must return less than 2 * sell(x). This follows from supply and demand. When selling x tokens, the pool receives more supply of x. Increased supply means lower demand, so the pool prices additional x at a lower rate. The first half of a 2x sale gets a better rate than the second half. The curve enforces this by providing diminishing returns for larger trades. A violation indicates the curve is not enforcing basic supply-demand economics.

**Float estimation accuracy**: The light versions use a float estimate as the seed for binary search. The production hint range is `[est - 250, est + 250]`. If the true integer output falls outside that range, the light function returns `None` and the swap instruction fails. This property tests that the float estimate stays close enough to the integer result for the seeded binary search to succeed.

**Skipped property**: 

Sequential swaps having increasing slippage was on the initial list but dropped. It follows from path independence (splitting a trade into smaller parts produces the same total output as one large trade) combined with slippage bounds (larger trades get worse rates than smaller trades). 

After the first swap executes, the pool state changes. But the slippage property still holds: a larger trade gets a worse rate. This means the second swap, operating on a pool with shifted reserves, must return less than the first swap would on that same state. Repeating this logic for each subsequent swap, each one returns less than the previous.

## Methodology

The complete test module is at [solana_stableswap_amm_property_tests.rs](https://gist.github.com/S3v3ru5/3be85868e8907a5cf731446e782984c3). The walkthrough below focuses on the structure and the key choices.

**What I wanted to emulate:**

A pool going through random state transitions. Swaps in both directions, varying amounts, reserves depleting and recovering. After each transition, the pool should still behave like a correct AMM. This is different from testing individual swaps in isolation. I wanted to ensure the AMM maintains its properties while experiencing trading patterns that move it through diverse pool states.

**How I structured it:**

Each test case is a sequence of 1-20 swap operations. I unified all swap types into a single enum:

```rust
enum SwapOp {
    XIn(u64),   // deposit X, receive Y
    YIn(u64),   // deposit Y, receive X
    XOut(u64),  // request X out, compute Y needed
    YOut(u64),  // request Y out, compute X needed
}
```

For each operation in the sequence:
1. Check if the operation is valid for current pool state
2. Test all properties on the current state
3. Execute the swap and update reserves
4. Verify invariant holds on new state

Properties are tested on a dynamically evolving pool state, not a static initial state. This is what makes the slippage and path independence tests meaningful across different reserve configurations.

**Proptest utilities used:**

Proptest is built around strategies. A strategy defines how to generate random values of a type. Simple strategies like `0..100u64` generate integers in a range. Complex strategies compose simpler ones.

For swap operations, I built a strategy that generates a random operation type and amount:

```rust
fn swap_op_strategy() -> impl Strategy<Value = SwapOp> {
    (0..4u8, MIN_SWAP_AMOUNT..MAX_SWAP_AMOUNT).prop_map(|(op_type, amount)| {
        match op_type {
            0 => SwapOp::XIn(amount),
            1 => SwapOp::YIn(amount),
            2 => SwapOp::XOut(amount),
            _ => SwapOp::YOut(amount)
        }
    })
}
```

`vec(swap_op_strategy(), 1..=MAX_SWAP_SEQUENCE)` generates sequences of operations with varying lengths. The fuzzer explores how the system behaves when trading happens repeatedly.

`prop_assume!` filters invalid operations. As pool state changes, some operations become invalid (would return `None` due to insufficient reserves). Rather than fail the test, filtering lets proptest generate alternatives. This ensures only realistic trading scenarios are tested.

`prop_assert!` checks properties with context. Error messages include the operation that failed, current reserves, and computed values.

**Handling edge cases:**

Generated amounts sometimes exceed what the pool can handle. `adjust_swap_amount` caps the input to the largest valid amount for the chosen direction, so the sequence can keep running instead of failing on an out-of-range input.

Near-zero reserves caused property violations. After a sequence of swaps in one direction, reserves can deplete to levels where the math behaves unexpectedly. I added a floor: skip test cases where reserves drop below 1000 units.

Precision differences between implementations required tolerance. I created a `within_precision` helper that checks if two values are within a maximum error bound. The thresholds, all in the smallest unit (6 decimals), are: equivalence between standard and light versions within 10, input/output consistency within 10, path independence within 10 across the split sequence, no-arbitrage round-trip profit strictly less than 1, and float-estimate divergence within the production hint range of ±250.


**Evolution during development:**

The methodology changed as I encountered issues:

Arbitrage tolerance started at 1 USD (1,000,000 units) assuming some rounding was inevitable. I tightened the threshold so that any positive gain at all triggers a failure, since consistent rounding direction should eliminate arbitrage entirely.

Path independence initially used fixed equal-sized splits. I changed to randomized split counts (2-10) and random amounts to test diverse execution patterns.

When a test failed, proptest automatically shrunk the failing case to a minimal sequence that reproduced the issue. During the audit I extracted these into standalone test functions for step-by-step debugging without fuzzer noise.

## Results

Each run executed 1000 sequences of up to 20 swaps and finished in roughly 8 seconds in debug mode, fast enough to keep in a routine test loop. I ran the suite multiple times during the audit, adjusting parameters and adding properties as issues surfaced. All runs used a single curve configuration with equal initial reserves and fixed amplification; varying these would extend coverage and is the obvious next step. Most properties passed across all sequences: equivalence, no arbitrage, invariant preservation, input/output consistency, path independence, and slippage bounds.

Float estimation initially appeared to surface an issue. The property test reported valid swap sequences where the actual output fell outside the [est - 250, est + 250] hint range, and the failure persisted even after widening to ±1e7. The cause turned out to be in the property test itself, not the production code: the float estimator and the integer-math swap function were called in opposite directions (swap_y_for_x_out matched against swap_x_for_y_out_light), so a real divergence was guaranteed by construction. My co-auditor spotted the mismatch in the property. After correcting the pairing, it held across all sequences.

A failing property is evidence that something is wrong, not that the production code is wrong. Both should be checked. Tracing what each side computed at the shrunk input is what surfaced the mismatch. Property tests are code. They have bugs like the code they test.

Running the tests gave us confidence in the math module. The core swap functions, both standard and light versions, behaved correctly across thousands of state transitions. The equivalence property confirmed that the production implementation matched the reference. The economic properties held throughout. This coverage would not have been practical to achieve through manual review or handwritten tests.

## Conclusion

Solana programs can be fuzzed too, and the math layer is the cheapest place to start: a strategy, a handful of properties, and a sequence loop are enough to exercise the curve across the configurations a pool actually traverses. That kind of coverage is what reading alone cannot produce, which is why it belongs in development rather than after. Doing so also nudges the implementation toward separating blockchain-interacting code from pure logic, where fuzzing pays back the most. For auditors, the same setup gives traction on areas that are hard to reason about from the source, and time spent writing strategies and properties at the start of an engagement tends to pay back over the rest of it.
