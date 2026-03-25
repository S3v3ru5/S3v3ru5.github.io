---
title: Partial Liquidations II - The Health Factor Threshold
date: 2026-03-25
tags: [DeFi, Research]
slug: partial-liquidations-II
math: false
---


# Partial Liquidations II: The Health Factor Threshold

[Part I]({{< ref "2026-03-24-partial-liquidations-I" >}}) showed that `LT * (1 + bonus) >= 1` causes partial liquidations to produce bad debt for all unhealthy positions. When `k = LT * (1 + bonus)`, the gap between debt and counted collateral changes by `r * (k - 1)` each pass. When `k < 1`, the gap narrows and the position moves toward health.

This post examines the next question: when `k < 1`, do all liquidatable positions recover? The answer depends on where the position's HF sits relative to `k`.

## How HF Changes After a Liquidation Pass

Each liquidation pass with repay amount `r = f * D` updates the position as follows:

```
debt removed:       f * D
collateral seized:  f * D * (1 + bonus)

C' = C - f * D * (1 + bonus)
D' = D * (1 - f)
```

The new health factor is:

```
HF' = (C' * LT) / D'

    = [C - f*D*(1+bonus)] * LT
      ─────────────────────────
             D*(1-f)

    = C*LT/D - f*LT*(1+bonus)
      ─────────────────────────
               (1-f)

    = HF - f*k
      ─────────
        (1-f)
```

where `k = LT * (1 + bonus)`.

Position size cancels out entirely. Two positions with identical `HF`, `LT`, `bonus`, and `f` follow the same HF trajectory pass by pass regardless of size. Whether partial liquidation helps or hurts depends only on the current HF relative to `k`.

## When Does HF Improve?

For HF to improve after a pass, `HF' > HF` must hold:

```
HF - f*k
──────── > HF
  (1-f)

HF - f*k > HF*(1-f)

HF - f*k > HF - f*HF

-f*k > -f*HF

k < HF
```

HF improves if and only if `HF > k`. The direction of each pass depends entirely on this comparison.

- `HF > k`: HF improves each pass. Position moves toward health.
- `HF = k`: gap unchanged, but HF still falls. Both numerator and denominator shrink each pass while their difference stays fixed. A fraction with a smaller denominator and the same absolute difference results in a smaller ratio.
- `HF < k`: HF worsens each pass. Position moves away from health.

`k` is the fixed point of the transformation. When `HF < k`, no finite number of passes can recover the position. This is the hard floor below which partial liquidation cannot help regardless of how many passes execute.

The `key_ratio >= 1` result from Part I is a special case of this. Liquidatable positions have `HF < 1`. When `k >= 1`, every liquidatable position satisfies `HF < 1 <= k` and is below the hard floor, so no position can recover once it becomes liquidatable.

## Three Zones: Recoverable, Solvent but Unrecoverable, and Insolvent

`k` and `LT` together divide the space of liquidatable positions into three regions, each requiring a different response from the protocol.

`k` is the recovery boundary. Above it, partial liquidation restores health. Below it, each pass worsens the position.

`LT` is the raw solvency boundary: when `HF = LT`, raw collateral equals debt. Since `LT < k` always holds, these two values create three distinct zones:

```
  0          LT           k            1
  |───────────|────────────|────────────|
    Zone 3       Zone 2       Zone 1
   Insolvent   Solvent but   Recoverable
              Unrecoverable
```


**Zone 1 (Recoverable): `k < HF < 1`**

Partial liquidation works as intended. Each pass improves HF and the borrower retains remaining collateral once the position is restored.


**Zone 2 (Solvent but Unrecoverable): `LT <= HF < k`**

Partial liquidation worsens the position. Since `HF >= LT` implies `C >= D`, raw collateral still covers the debt.

Full liquidation in one call yields a profit of `C - D` to the liquidator. The standard bonus is not achievable since `C < D*(1+bonus)`: `HF < k` implies `C*LT/D < LT*(1+bonus)`, so `C < D*(1+bonus)`. But the debt clears and the protocol holds no bad debt.

Whether `C - D` covers liquidation costs depends on position, and small Zone 2 positions may go unliquidated.


**Zone 3 (Insolvent): `HF < LT`**

Raw collateral is less than debt. Full liquidation at any positive bonus is unprofitable. No market-driven liquidator will act.

The protocol needs a planned backstop such as an insurance fund or socialised loss mechanism.

## Configuring LT and Bonus for Adequate Zone 1

The zone boundaries are entirely determined by LT and bonus. A high LT compresses Zone 1 significantly.

Zone 1 spans `(k, 1)` with width `1 - k = 1 - LT*(1+bonus)`. Zone 2 spans `(LT, k)` with width `k - LT = LT * bonus`.

```
LT=0.80, bonus=0.05: k=0.840  Zone 1 width=0.160  Zone 2 width=0.040
LT=0.90, bonus=0.05: k=0.945  Zone 1 width=0.055  Zone 2 width=0.045
LT=0.95, bonus=0.05: k=0.998  Zone 1 width=0.003  Zone 2 width=0.048
```

At `LT = 0.95` with a 5% bonus, Zone 1 is only 0.003 wide. A position entering liquidation at HF = 0.99 is already below `k` and in Zone 2. Even though `k < 1`, the protocol behaves similarly to the `key_ratio >= 1` case for most practical liquidation entries.

The constraint `k < 1` is necessary but not sufficient. What matters in practice is that Zone 1 is wide enough to cover the expected distribution of liquidation entries. For a protocol expecting positions to enter liquidation around HF = 0.95-0.99, this requires `k < 0.95` substantially stricter than `k < 1`. **A protocol can satisfy the Part I invariant and still have partial liquidation fail for nearly every realistic liquidation entry.**

## Findings from Protocol Scan
 
The protocol scan accompanying Part 1 checked `key_ratio` across 22 HF-based partial liquidation protocols. 4 had `key_ratio >= 1` and are covered in Part 1. Of the remaining 18, 7 have `key_ratio > 0.97`, leaving Zone 1 narrower than 0.03.
 
For these protocols, Zone 1 is narrow enough that most real liquidation entries fall below `k`. Partial liquidation worsens rather than restores the position even though `key_ratio < 1`. Six of the seven have no on-chain invariant, making them one governance update away from crossing the boundary entirely.
 
```
key_ratio=0.9975: Zone 1 width=0.0025  (position at HF=0.998 is in Zone 2)
key_ratio=0.9900: Zone 1 width=0.0100  (position at HF=0.995 is in Zone 2)  [2 protocols]
key_ratio=0.9863: Zone 1 width=0.0137  (position at HF=0.990 is in Zone 2)
key_ratio=0.9785: Zone 1 width=0.0215  (position at HF=0.990 is in Zone 2)  [2 protocols]
key_ratio=0.9765: Zone 1 width=0.0235  (position at HF=0.990 is in Zone 2)
```
 
These protocols satisfy the Part I invariant but provide almost no Zone 1 coverage for realistic liquidation entries.

## Optimising Partial Liquidations

Using the zone boundaries derived above, along with the goal of reducing collateral loss for borrowers while maintaining protocol safety, the following optimisations can be made.

### Dynamic Close Factor

LT and bonus determine which zone a position lands in and the direction of HF change. The close factor `f` determines the magnitude.

A fixed close factor has two consequences: it may require more than one pass to restore health, and it may seize significantly more collateral than the minimum needed. By using a target HF', exact close factor that makes position healthy can be calculated.

Rearranging the HF transformation to solve for `f` given a target `HF'`:

```
      HF - f*k
HF' = ─────────
        (1-f)

HF'*(1-f) = HF - f*k
HF' - f*HF' = HF - f*k
HF' - HF = f*(HF' - k)

      HF' - HF
f  =  ─────────
      (HF' - k)
```

Given a target HF' of 1.05 or 1.10, this is the exact close factor that restores health in one pass with minimum collateral seizure. Targeting exactly 1.00 leaves the position at the liquidation boundary, vulnerable to re-entry from a small price move. A buffer of 1.05 or 1.10 provides separation.

For `LT = 0.80`, `bonus = 0.05`, `k = 0.84`, `D = $1000`, targeting `HF' = 1.05`:

```
HF=0.99: C=$1237.50  f=0.286  repay=$285.71  seized=$300.00  borrower_retains=$937.50
HF=0.95: C=$1187.50  f=0.476  repay=$476.19  seized=$500.00  borrower_retains=$687.50
HF=0.90: C=$1125.00  f=0.714  repay=$714.29  seized=$750.00  borrower_retains=$375.00
```

Compare with a fixed `f = 0.50` for a position at `HF = 0.99`, `C = $1237.50`, `D = $1000`:

```
Fixed f=0.50:    repay=$500.00  seized=$525.00  borrower_retains=$712.50  HF_after=1.140
Dynamic f=0.286: repay=$285.71  seized=$300.00  borrower_retains=$937.50  HF_after=1.050

Excess collateral seized = $225.00
```

The borrower loses $225 more than necessary. The overpayment is largest for mild violations where HF is just below 1.

Exactly Protocol and Aave V4 have implemented variations of this approach. Exactly Protocol uses a [dynamic close factor](https://docs.exact.ly/guides/features/dynamic-close-factor) that targets position health restoration, and Aave V4 introduced a [Target Health Factor](https://aave.com/blog/aave-v4-liquidations) mechanism that limits liquidation to the minimum repayment needed to reach the target. Both reduce borrower loss for Zone 1 positions compared to a fixed close factor.

### Dynamic Liquidation Bonus

A fixed bonus does not account for position size or liquidation cost. A 5% bonus on a $10,000 repayment yields $500. The same 5% on a $100 repayment yields $5, which may not cover the costs.

The dynamic close factor formula gives the exact repayment for a target recovery HF. Using this repayment as the basis, an estimate of liquidation cost (depends on tokens; liquidity, slippage, gas, ...) can be used to compute the minimum viable bonus for that specific position, rather than a blanket rate that overpays for large positions and potentially underpays for small ones.

For example, if restoring a position requires repaying $500 and the estimated cost is $15, the minimum bonus is 3%, versus a blanket 5% that overpays by $10. This approach reduces borrower loss on large positions while ensuring small positions remain profitable to liquidate.

### Shifting to Full Liquidation in Zone 2

When the position is in Zone 2, partial liquidation worsens it each pass, reducing `C - D` and shrinking the buffer available to a full liquidator.

Shifting to full liquidation while `C - D` is still sufficient to cover gas prevents bad debt and clears the position cleanly. The protocol can check `HF >= k` before applying the close factor and switch to full liquidation when the condition is not met.

## Considerations for Protocol Developers

**Implement Zone detection.** Check whether `HF >= k` before applying the close factor. If the position is in Zone 2, allow full liquidation. Partial liquidation in Zone 2 worsens the position and leaves bad debt even when `k < 1`.

**Target a recovery HF rather than a fixed close factor.** Use `f = (HF' - HF) / (HF' - k)` with a target HF of 1.05 or 1.10. This restores the position in one pass with the minimum collateral seizure required.

**Set a minimum liquidation size for Zone 2.** Small Zone 2 positions where `C - D` does not cover gas will sit unliquidated. A minimum position size threshold or a protocol-owned keeper as fallback prevents accumulation.

**Evaluate Zone 1 width when setting parameters.** `k < 1` is necessary but not sufficient. The Zone 1 width `1 - k` should cover the expected distribution of liquidation entries. At high LT, satisfying `k < 1` with a small bonus may leave Zone 1 too narrow to be useful.

## Conclusion

Partial liquidations are more nuanced than protocols tend to treat them. As shown in Part 1, LT and bonus are often set independently with minimal consideration of how their joint value affects borrower outcomes. Very few protocols design liquidation systems with the goal of reducing borrower loss alongside protocol safety.

Understanding how each parameter contributes to the mechanism allows protocols to reason about liquidations with more precision. The HF transformation and the three zones give a concrete model for this. By understanding the influence of each factor, protocols can dynamically tune parameters with confidence, targeting specific recovery outcomes rather than relying on fixed approximations.
