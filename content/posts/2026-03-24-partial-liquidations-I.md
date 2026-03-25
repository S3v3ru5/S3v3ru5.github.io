---
title: Partial Liquidations I - When the Mechanism Creates Bad Debt
date: 2026-03-24
tags: [DeFi, Research]
slug: partial-liquidations-I
math: false
---

# Partial Liquidations I: When the Mechanism Creates Bad Debt

Partial liquidation restores undercollateralized positions incrementally, reducing collateral loss to borrowers compared to full liquidation. The protocol parameters, liquidation threshold (LT) and the liquidation bonus, determine the collateral loss in partial liquidations. Certain combinations of these values leave the protocol with bad debt and exhaust the borrower's collateral, even for positions that are only slightly unhealthy.

This post covers how partial liquidation works, the parameter condition that determines its effectiveness, the consequences when that condition is violated, and practical guidance for protocol developers and security researchers. The analysis considers positions with a single collateral and single debt asset. A similar analysis can be extended to positions with multiple collateral and borrow assets.

## Background: Positions and Health Factor

A lending position consists of collateral deposited and debt outstanding. The protocol tracks solvency through the **Health Factor**:

```
HF = (collateral_usd * liquidation_threshold) / debt_usd
```

The liquidation threshold (LT) is a risk-adjusted weight on collateral value, reflecting the protocol's safety buffer for that asset. When `HF >= 1`, the position is solvent. When `HF < 1`, the position is eligible for liquidation.

```
collateral = $1000 ETH,  LT = 0.80
debt       = $850 USDC

HF = (1000 * 0.80) / 850 = 0.941  =>  liquidatable
```

## The Partial Liquidation Mechanism and Its Intent

Full liquidation resolves insolvency by repaying all debt and seizing all collateral. It imposes the maximum possible loss on borrowers regardless of the violation size. A position at HF = 0.99 would lose its entire collateral balance under full liquidation.

Partial liquidation allows liquidators to repay only a fraction of outstanding debt per call. The protocol rewards the liquidator with slightly more collateral than the debt repaid. The difference is the **liquidation bonus**:

```
collateral_seized = repay_amount * (1 + bonus)
```

The **close factor** (f) caps how much debt a liquidator can repay in one transaction:

```
max_repay = f * total_debt
```

Each pass reduces both debt and collateral, moving HF back toward 1. Once the position is healthy, liquidation stops. The borrower retains remaining collateral; the liquidator keeps the bonus.

Under this model, the borrower's loss is proportional to the degree of undercollateralization, not the entire position size.

## Why Does Repaying Debt Improve HF?

If the liquidator takes more collateral than the debt they repay, why does HF improve?

HF is not the ratio of raw collateral to raw debt. It is the ratio of *counted* collateral to debt, where collateral is discounted by the liquidation threshold.

Consider a position with `collateral = $1000`, `LT = 0.80`, `debt = $850`:

```
HF = (1000 * 0.80) / 850 = 800 / 850 = 0.941
     counted collateral = $800
     debt               = $850
```

A liquidator repays $100 at a 5% bonus, seizing $105 of collateral:

```
collateral seized    = $100 * 1.05 = $105  (raw collateral removed)
counted collateral   = $105 * 0.80 = $84   (what HF actually loses in numerator)
debt repaid          = $100                (what HF loses in denominator)

new counted collateral = $800 - $84  = $716
new debt               = $850 - $100 = $750

new_HF = 716 / 750 = 0.955  (up from 0.941)
```

Debt drops by $100 while the counted collateral drops by only $84. The $105 of seized collateral was worth $105 at face value, but the protocol only counted it as $84 for the health factor.

Generalizing this, the liquidator repays `r` of debt and receives `r * (1 + bonus)` of collateral. The protocol counts only the LT fraction of collateral towards the health factor computation:

```
counted collateral removed = r * (1 + bonus) * LT
debt removed               = r
```

The ratio of counted-collateral-removed to debt-removed is:

```
r * (1 + bonus) * LT
-------------------- = LT * (1 + bonus)
         r
```

For every $1 of debt repaid, the protocol removes `LT * (1 + bonus)` dollars of the **counted collateral**. The `key_ratio = LT * (1 + bonus)` determines how HF moves after liquidation.

- `LT * (1 + bonus) < 1`: less than $1 of counted collateral removed per $1 of debt. Counted collateral increases relative to debt. Position can recover.
- `LT * (1 + bonus) = 1`: exactly $1 of counted collateral removed per $1 of debt. HF still falls, position gets worse. (The gap stays fixed but both numerator and denominator shrink, and a smaller denominator with the same gap yields a smaller ratio.)
- `LT * (1 + bonus) > 1`: more than $1 of counted collateral removed per $1 of debt. Counted collateral decreases relative to debt. Position gets worse.

### Derivation

After one liquidation pass with repay amount `r`, debt `D`, and collateral value `C`, the updated values are:

```
new_debt       = D - r
new_collateral = C - r * (1 + bonus)

                 (C - r*(1+bonus)) * LT
new_HF         = ──────────────────────
                        (D - r)
```

Define the gap `G` as the difference between debt and the counted collateral:

```
G = D - C*LT
```

When the position is unhealthy, `G > 0`. Restoring health means reducing G to zero or below.

After one pass, the new gap is:

```
new_G = new_debt - new_collateral * LT

      = (D - r) - (C - r*(1+bonus)) * LT

      = D - r - C*LT + r*(1+bonus)*LT

      = (D - C*LT) + r * ((1+bonus)*LT - 1)

      = G + r * (LT*(1+bonus) - 1)
```

The gap changes by `r * (LT*(1+bonus) - 1)`.

```
gap_change = r * (key_ratio - 1)
```

Since `r > 0`, the sign of `key_ratio - 1` determines whether the gap shrinks or grows:

- `key_ratio < 1`: gap_change < 0, gap narrows, position moves toward health
- `key_ratio = 1`: gap_change = 0, gap unchanged, position moves away from health.
- `key_ratio > 1`: gap_change > 0, gap widens, position moves away from health

## Bad Debt: When Partial Liquidation Leaves the Protocol Worse Off

When `key_ratio >= 1`, the liquidation process itself accrues bad debt. As the gap widens with each pass, the position deteriorates until the collateral balance is exhausted before all debt is repaid. The residual debt has no collateral backing it, forcing the protocol to absorb the loss.

This applies to every unhealthy position under such a configuration, regardless of the initial gap size.

The following example shows a position where full liquidation clears all debt, but partial liquidation produces bad debt.

```
collateral = $1020.00,  LT = 0.97,  bonus = 0.05,  close_factor = 0.50
debt       = $1000.00

HF        = (1020 * 0.97) / 1000 = 0.9894   (liquidatable)
key_ratio = 0.97 * 1.05          = 1.0185   (irrecoverable)
```

**Partial liquidation**

Gap grows by `r * (key_ratio - 1) = r * 0.0185` each pass:

```
Pass 1: repay $500.00  seized $525.00  coll $495.00  debt $500.00  HF 0.9603  gap $19.85
Pass 2: repay $250.00  seized $262.50  coll $232.50  debt $250.00  HF 0.9021  gap $24.47
Pass 3: repay $125.00  seized $131.25  coll $101.25  debt $125.00  HF 0.7857  gap $26.79
Pass 4: repay  $62.50  seized  $65.62  coll  $35.62  debt  $62.50  HF 0.5529  gap $27.94
Pass 5: repay  $31.25  seized  $32.81  coll   $2.81  debt  $31.25  HF 0.0873  gap $28.52
Pass 6: repay   $2.68  seized   $2.81  coll   $0.00  debt  $28.57

Collateral exhausted. Protocol bad debt = $28.57
```

The gap started at $10.60 (`1000 - 1020*0.97`) and grew to $28.57 through liquidation. The borrower loses all $1020 of collateral. The protocol is left with $28.57 of uncollateralized debt.

**Full liquidation**

A liquidator repays the entire $1000 debt in one call and receives all available collateral:

```
Liquidator repays:        $1000.00
Collateral seized:        $1020.00  (all available; bonus cap would be $1050 but capped at balance)
Liquidator net profit:      $20.00
Residual debt:               $0.00
Protocol bad debt:           $0.00
```

The liquidator profits $20 on a $1000 outlay. Debt is fully cleared with no bad debt.

Full liquidation removes all collateral regardless of violation size. When `key_ratio < 1`, this is worse for borrowers than partial liquidation. When `key_ratio >= 1`, partial liquidation also exhausts all collateral but additionally leaves bad debt.

**Recoverable configuration: same HF, LT = 0.80, bonus = 0.05**

```
collateral = $1236.75,  LT = 0.80,  bonus = 0.05,  close_factor = 0.50
debt       = $1000.00

HF        = (1236.75 * 0.80) / 1000 = 0.9894   (same starting HF)
key_ratio = 0.80 * 1.05             = 0.840    (recoverable)
```

```
Pass 1: repay $500.00  seized $525.00  coll $711.75  debt $500.00  HF 1.1388

Position restored. Borrower retains $711.75. Protocol bad debt = $0.
```

The close factor allows repaying $500. The borrower retains $711.75 of the original $1236.75.

**Outcome comparison**

| | Irrecoverable partial | Irrecoverable full | Recoverable partial |
|-|-----------------------|-------------------|---------------------|
| key_ratio | 1.0185 | 1.0185 | 0.840 |
| Passes | 6 | 1 | 1 |
| Borrower collateral retained | $0 | $0 | $711.75 |
| Liquidator profit | variable per pass | $20.00 | variable |
| Protocol bad debt | $28.57 | $0 | $0 |

## Why the Configuration Condition is Frequently Missed

LT and bonus are set independently. LT reflects collateral volatility; bonus reflects liquidator economics. The joint constraint `LT * (1 + bonus) < 1` is not part of either analysis:

```
bonus < (1 - LT) / LT
```

For high-LT assets, the ceiling on viable bonus is quite low. At `LT = 0.97`, the bonus must stay below approximately 3.1%, i.e., the maximum bonus before partial liquidations leave bad debt. A 5% bonus, which is a common baseline for liquidator incentives, pushes `key_ratio` above 1.

```
LT = 0.97, bonus = 0.05:  key_ratio = 1.0185  =>  irrecoverable
LT = 0.97, bonus = 0.03:  key_ratio = 1.0000  =>  neutral (boundary)
LT = 0.97, bonus = 0.02:  key_ratio = 0.9894  =>  recoverable, but very slowly
```

eMode is a common example. It restricts positions to correlated assets (ETH/stETH, USDC/USDT) and sets LT in the 95-97% range. The bonus is typically carried over from general market settings without checking the joint constraint.

## Impact on Borrowers

The main consequence of an incorrect configuration is complete collateral loss for borrowers, even with a minimal health factor violation. Partial liquidation exists to protect borrowers by restoring position health incrementally. When `key_ratio >= 1`, this protection fails. Every liquidation pass worsens the position until all collateral is consumed, producing the same outcome as full liquidation but with additional bad debt for the protocol.

In practice, this disproportionately affects efficiency mode (eMode) configurations. eMode positions use high liquidation thresholds (95-97%) to reflect asset correlation, but this leaves very little room for the liquidation bonus before `key_ratio` exceeds 1. The configurations most likely to violate the constraint are the ones designed to be safest, and their borrowers face complete collateral loss from even minor health factor violations.

## Findings from Protocol Deployments

After finding this `key_ratio >= 1` edge case, I shared it with my friend [D4r3_D3v1L_](https://x.com/D4r3_D3v1L_), who's also a blockchain security researcher, and requested him to check on-chain deployments. He took on the work of identifying and analyzing lending protocols across chains.
 
For Aptos, and Sui, projects were selected using DeFiLlama with a TVL threshold of $100k or above. Both open source and decompiled contracts were reviewed for these chains. For Solana, only open source projects were checked. On-chain state was read to determine the configured LT and bonus values for each pool.
 
| Chain | Projects Checked |
|-------|-----------------|
| Aptos | 10 |
| Sui | 3 |
| Solana | 14 |
| EVM | 3 |
 
Of the 30 projects, 4 were dead on-chain with no active users, 4 use a weight-based liquidation model, and 1 uses full liquidation only. The remaining 22 use HF-based partial liquidation and form the relevant set for this analysis.
 
Of those 22 protocols, 4 (18%) are actively configured with `key_ratio >= 1`. Only 6 of the 22 (27%) enforce the constraint in code. The remaining 16 set LT and bonus independently with no joint validation.
 
Disclosure outcomes for the 4 vulnerable protocols:

- One protocol's bug bounty explicitly excludes liquidations leading to bad debt from scope.
- One protocol responded that markets are permissionless and they are aware of this case.
- One protocol has not yet responded.
- One protocol had no available contact information.

The results confirm what the analysis suggests. The joint constraint is not part of standard parameter review. Most protocols are not aware of it, and the 4 vulnerable protocols are actively producing bad debt for positions that enter liquidation under the affected pools.

## Recommendations for Protocol Developers

**Treat `key_ratio < 1` as a protocol invariant.** Encode a check in the parameter-setting logic for LT and bonus. Any update that pushes `key_ratio >= 1` for an active asset should require explicit sign-off, not be caught later in a risk review.

**Set LT to satisfy the joint constraint.** The constraint `LT * (1 + bonus) < 1` must hold for partial liquidation to be recoverable. If the required bonus floor is 5%, LT must stay below `1 / 1.05 = 0.952`. Setting LT above that with a 5% bonus produces `key_ratio >= 1` regardless of other configuration. For high-LT modes like eMode, this means accepting either reduced capital efficiency or a different liquidation mechanism.

**Educate governance participants on the joint constraint.** For existing protocols where encoding an invariant in contract logic is not immediately feasible, governance proposals that touch LT or bonus should include an explicit discussion of `LT * (1 + bonus)` for affected assets and confirm it stays below 1. Participants with proposal or voting privileges should understand that each parameter appears reasonable in isolation and the interaction is what produces the harmful configuration.

## Audit Considerations

**Ensure protocol has on-chain constraints on `liquidation_threshold` and `liquidation_bonus` preventing key_ratio >= 1.**

**Do not treat admin parameters as out of scope.** Audit scopes often exclude admin configuration. For liquidation parameters, this assumption does not hold: a governance update valid on each parameter individually can still produce `key_ratio >= 1`. The constraint `bonus < (1 - LT) / LT` is not enforced in code and will not appear in a standard code review.

**Test mechanism intent, not just execution.** A standard liquidation test checks that the call succeeds and debt falls. A mechanism-correctness test also checks that HF after the call is higher than before. Only the latter confirms the mechanism is working as intended for a given configuration.

## Conclusion

A partial liquidation with `liquidation_threshold * (1 + liquidation_bonus) >= 1` leaves the protocol with bad debt and the borrower with nothing, even for a position that was barely unhealthy.

Liquidation configuration should be evaluated as a system, not parameter by parameter. The boundary between recoverable and irrecoverable should be enforced as part of the design.

Security researchers should verify whether the chosen mechanism is beneficial across all permitted parameter values. For partial liquidation, that means finding the configuration limits that make it advantageous and making sure developers understand them. Verifying the joint constraints behind parameter choices helps prevent configurations that harm users.

