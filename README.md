# Denaria-165k-PoC
>
>**Chain:** Linea  
>**Block:** [30067821](https://lineascan.build/tx/0xcb0744a0d453e5556f162608fae8275dabd14292bffbfcd8394af4610c606447)  
>**Attacker:** `0x8D6778d7FAe00aD2e0bc12194cF03B756FED9Db3`  
>**Victim (Vault):** `0x61cE9B51010BA52F701444f0F3D1e563F6ae8d91`  
>**Loss:** ~165,617 USDC  

---

## Summary

Denaria is a perpetual DEX on Linea that uses a virtual AMM (vAMM) model, the protocol has three core components

| Contract | Address | Role |
|----------|---------|------|
| **Vault** | `0x61cE9B51010BA52F701444f0F3D1e563F6ae8d91` | Holds USDC collateral for all users |
| **PerpPair** | `0xB68396dD4230253d27589e2004Ac37389836AE17` | Virtual AMM — tracks positions, liquidity, and curve state |
| **CurveMath** | `0x0Ef31752A4D7bef5A46b378873613F706255B9CD` | Library computing position values against the curve |

users deposit USDC into the Vault as collateral, then interact with PerpPair to either
- **Trade** — open leveraged long/short positions
- **Provide liquidity** — supply virtual stable/asset liquidity to the AMM curve

LP positions earn PnL based on how the curve moves relative to their entry point, the function `realizePnL()` materializes this PnL and adds it to the LP collateral balance via `addPnlToCollateral()`, making it withdrawable

---

## Root Cause

**`realizePnL()` computes LP PnL against the instantaneous vAMM curve state with zero manipulation resistance.**

The PnL calculation path is

```
realizePnL(bytes)
  └─> _calcPnL()
        └─> CurveMath.computeShortReturn()
              reads: globalLiquidityStable, globalLiquidityAsset
              returns: LP position value based on current curve ratio
```

`computeShortReturn()` (at `0x78197FE9...`) values the LP virtual asset surplus by reading the **current** `globalLiquidityStable / globalLiquidityAsset` ratio.

- No TWAP (time-weighted average price)
- No minimum time delay between operations
- No cap on PnL relative to collateral
- No slippage protection on PnL realization

a single large trade in the same transaction can distort the curve ratio arbitrarily, inflating the LP apparent PnL by orders of magnitude

The attacker executes everything in a **single transaction** using an Aave USDC flashloan

```
Attacker EOA
  └─> Orchestrator contract (flash loan receiver)
        ├─> Contract 0 (LP role)
        ├─> Contract 1 (Trader role — sacrificial)
        ├─> Contract 2 (LP role — round 2)
        └─> Contract 3 (Trader role — round 2, sacrificial)
```

The attacker flashloan **60,000 USDC** from Aave on Linea

// Round 1

| Step | Actor | Action | Detail |
|------|-------|--------|--------|
| 1 | LP (Contract 0) | `addCollateral(30,000 USDC)` | Deposits 30K USDC into Vault |
| 2 | LP (Contract 0) | `addLiquidity(20,000e18)` | Provides 20K virtual stable to the AMM curve |
| 3 | Trader (Contract 1) | `addCollateral(15,000 USDC)` | Deposits 15K USDC into Vault |
| 4 | Trader (Contract 1) | `trade(long, 100,000e18)` | Opens 100K notional long — **massively skews the curve** |
| 5 | LP (Contract 0) | `realizePnL()` | PnL computed against distorted curve = **183,283 USDC** |
| 6 | LP (Contract 0) | `removeCollateral(PnL)` | Withdraws 183,283 USDC from Vault |

30K collateral in, 183K out, trader 15K is sacrificed (position is underwater due to the massive skew, but the LP extracted far more)

// Round 2 

The vault still holds ~27K USDC of pre-existing user funds plus the 15K deposited by Round 2 actors

| Step | Actor | Action | Detail |
|------|-------|--------|--------|
| 7 | LP2 (Contract 2) | `addCollateral(10,000)` + `addLiquidity(8,000e18)` | Smaller position |
| 8 | Trader2 (Contract 3) | `addCollateral(5,000)` + `trade(long, 30,000e18)` | Skews curve again |
| 9 | LP2 (Contract 2) | `realizePnL()` → `removeCollateral()` | Withdraws **42,364 USDC** (capped by vault balance) |

```
Total extracted:   225,647 USDC  (183,283 + 42,364)
Flash loan repaid:  60,030 USDC  (60,000 + 30 fee)
─────────────────────────────────
Net profit:        165,617 USDC
Vault balance:           0 USDC  (drained)
```

---

how `addLiquidity` creates Virtual Exposure

when the LP calls `addLiquidity(20,000e18)`, the PerpPair

1. Records the LP entry point on the curve (virtual stable/asset amounts)
2. Updates `globalLiquidityStable` and `globalLiquidityAsset`
3. The LP effectively holds a "short" position against the curve if the curve moves in a certain direction, the LP virtual asset surplus increases in value

when the trader opens a **100K notional long** (6.67x leverage on 15K collateral)

1. The vAMM simulates buying 100K worth of the asset
2. `globalLiquidityStable` increases (more stable pushed into the pool)
3. `globalLiquidityAsset` decreases (asset pulled from the pool)
4. The ratio `globalLiquidityStable / globalLiquidityAsset` shifts dramatically

This ratio change is **not** reflected in the oracle price (the oracle returns the same value `0x615706c3a01` throughout), the distortion exists purely in the **virtual AMM's internal accounting**.

`realizePnL` gets Inflated

When the LP calls `realizePnL()`

```
PerpPair.realizePnL(bytes(""))
  ├─ oracle.updatePrice()           // Oracle price unchanged
  ├─ CurveMath._calcPnL()
  │     ├─ reads distorted globalLiquidityStable/Asset
  │     └─ computeShortReturn()     // Values LP surplus against skewed curve
  │           └─ returns 183,283e18  // Massively inflated
  ├─ Vault.addPnlToCollateral(LP, 183,283e18, true)
  └─ returns (183,283e18, true)
```

LP collateral jumps from 30,000 to 213,283 USDC (30K + 183K PnL), the LP then withdraws the PnL portion.

after Round 1 drains ~183K, Vault only holds ~42K USDC (27K pre-existing + 15K new deposits from Round 2 actors), although LP2 computed PnL is ~159K, the withdrawal is capped by the Vault actual USDC balance

```solidity
uint256 withdrawAmount = pnl < totalCollateral ? pnl : totalCollateral;
```

---

## Addresses

| Role | Address |
|------|---------|
| Attacker EOA | `0x8D6778d7FAe00aD2e0bc12194cF03B756FED9Db3` |
| Orchestrator (flash loan receiver) | `0xB87275489272ce1c4bE358FC5856Ea3273093CF8` |
| Contract 0 — LP (Round 1) | `0xf1b409B06C27Ba013C64f62306Dbb4df6C6B9217` |
| Contract 1 — Trader (Round 1) | `0xE205b04DB446f37BE087A42421E9ce51a0eAef16` |
| Contract 2 — LP (Round 2) | `0xE84414ed110D3Aaf3809cd3ffD83c5ae8fE9DbA5` |
| Contract 3 — Trader (Round 2) | `0xb9c7D5C554B20FBBF13945B63605Fb9baa286194` |
| Vault (victim) | `0x61cE9B51010BA52F701444f0F3D1e563F6ae8d91` |
| PerpPair | `0xB68396dD4230253d27589e2004Ac37389836AE17` |
| CurveMath library | `0x0Ef31752A4D7bef5A46b378873613F706255B9CD` |
| PnL calculator (computeShortReturn) | `0x78197FE93999e34D5A688E1819923c66DCf8F4DB` |
| Oracle | `0x12a9EAd186e29c11dba79E52518DaCe3EC2D1cd0` |
| USDC (Linea) | `0x176211869cA2b568f2A7D4EE941E073a821EE1ff` |
| Aave Pool (Linea) | `0xc47b8C00b0f69a36fa203Ffeac0334874574a8Ac` |

---

## Selectors

| Selector | Inferred Name | Signature |
|----------|---------------|-----------|
| `5cd38b42` | `trade` / `openPosition` | `?(bool,uint256,...,bytes,bytes)` |
| `1cb794ea` | `realizePnL` | `?(bytes)` |
| `8ede96aa` | `calcPositionPnL` | `?(address,uint256)` view |
| `cc4cd881` | `tradePosition` | `?(address)` view |
| `9fd8c908` | `addPnlToCollateral` | `?(address,uint256,bool)` |
| `0fab3b84` | `settle` / `sync` | `?(bytes)` |

---
