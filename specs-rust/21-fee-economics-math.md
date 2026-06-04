# 21 — Fee & Economics Math (consensus-critical formulas)

> **Status:** Conforms to `00-overview-and-conventions.md` (binding). In particular
> §6.1 (determinism): **all consensus-path arithmetic is checked integer
> arithmetic on fixed-width integers (`u64` / `U256`); no floating point ever
> touches a value that goes into a block, a state root, or a fee charge.** Where
> the Go code derives a constant from a float (e.g. `… / ln(2)`), the *result* is
> hard-coded as an integer literal and reproduced verbatim here.
>
> Cross-refs: `03-core-primitives.md` (`safemath` discipline, `Fork`/`UpgradeConfig`
> fork-gating model, the per-crate source-mapping table), `08-platformvm-pchain.md`
> (P-Chain dynamic/static fees, staking rewards, L1 validator lifecycle),
> `09-avm-xchain.md` (X-Chain shares the ACP-103 mechanism), `10-cchain-evm-reth.md`
> (EVM AP3/AP4/ACP-176 fees), `11-saevm.md` (SAE gas-as-time), `02-testing-strategy.md`
> (golden vectors + differential testing vs Go).

This document is the single consolidated reference for **every** fee/economics
formula in the node. An off-by-one in any of these splits the chain, so each
mechanism is given as (a) the exact math, (b) the exact integer algorithm with Go
citations and per-network constants, (c) a Rust sketch using checked arithmetic,
and (d) **worked numeric examples that double as golden test vectors**.

## Go source covered

`vms/components/gas/{gas,state,dimensions,config}.go` ·
`vms/platformvm/txs/fee/{dynamic_calculator,simple_calculator,complexity}.go` ·
`vms/platformvm/validators/fee/fee.go` · `vms/platformvm/reward/{calculator,config}.go` ·
`vms/evm/acp176/acp176.go` ·
`graft/coreth/plugin/evm/customheader/{base_fee,block_gas_cost,dynamic_fee_windower,dynamic_fee_state}.go` ·
`graft/coreth/plugin/evm/upgrade/{ap3,ap4,ap5,etna}/*.go` ·
`vms/saevm/gastime/{gastime,acp176,config}.go` · `vms/saevm/proxytime/proxytime.go` ·
`vms/saevm/intmath/intmath.go` · `genesis/genesis_{mainnet,fuji,local}.go`.

---

## 0. The shared exponential: `fake_exponential` (`CalculatePrice`)

**Five of the six mechanisms** route through one routine:
`gas.CalculatePrice` in `vms/components/gas/gas.go`. It is the single most
error-prone piece of the entire fee subsystem; get it bit-exact and most of the
rest follows. It computes an integer approximation of

```math
\text{CalculatePrice}(m, x, k) \;\approx\; m \cdot e^{x/k}
```

where `m = minPrice`, `x = excess`, `k = excessConversionConstant` (a.k.a. K).
It is the EIP-4844 `fake_exponential` (a truncated Taylor series of `e^{x/k}`):

```python
# Reference (EIP-4844). The Go code is an integer-exact, overflow-bounded port.
def fake_exponential(factor: int, numerator: int, denominator: int) -> int:
    i = 1
    output = 0
    numerator_accum = factor * denominator
    while numerator_accum > 0:
        output += numerator_accum
        numerator_accum = (numerator_accum * numerator) // (denominator * i)
        i += 1
    return output // denominator
```

Here `factor = m`, `numerator = x`, `denominator = k`. The series term is
`m·x^i / (k^i · i!)`; summing and dividing by `k` once at the end yields
`m·Σ (x/k)^i / i! = m·e^{x/k}`.

### Go integer algorithm (verbatim semantics)

From `gas.go`. The Go optimizes with the invariant that any result `> MaxUint64`
is clamped to `MaxUint64`, which bounds every intermediate to ≤ `MaxUint193`, so
`uint256` (256-bit) is sufficient and never itself overflows.

```text
numerator        := x                       # u256, init from u64
denominator      := k                       # u256, init from u64
i                := 1
output           := 0
numerator_accum  := m * k                    # u256 (≤ MaxUint128)
max_output       := k * MaxUint64            # u256 (≤ MaxUint128); clamp threshold
while numerator_accum > 0:
    output += numerator_accum                # if output >= max_output -> return MaxUint64
    numerator_accum = numerator_accum * x
    numerator_accum = numerator_accum / k
    numerator_accum = numerator_accum / i
    i += 1
return (output / k) as u64
```

> **Precision traps (must replicate exactly):**
> 1. The clamp test is `output >= max_output` (i.e. `>=`, not `>`), checked
>    **before** the next `numerator_accum` update, and returns `MaxUint64`.
> 2. The two divides are **separate**: `/k` then `/i` (NOT `/(k*i)`). Integer
>    truncation differs between `(a/k)/i` … no: Go divides `numerator_accum/k`
>    then `/i` as written — replicate that exact order. (The EIP pseudocode
>    divides by `k*i` in one step; Go splits it. Match **Go**, not the EIP.)
> 3. Final result is `output / k` (the trailing `÷denominator`), truncated.
> 4. `m=0` → loop body never runs (accum starts 0) → returns 0. `k=0` would
>    divide by zero; callers guarantee `k ≥ 1`.
> 5. Loop terminates because `numerator_accum` strictly decreases once
>    `i > x/k` (the factorial dominates), reaching 0 by truncation.

### Rust sketch

```rust
// crate: ruint  (U256). No floats. Mirrors vms/components/gas/gas.go exactly.
use ruint::aliases::U256;

pub fn calculate_price(min_price: u64, excess: u64, k: u64) -> u64 {
    debug_assert!(k != 0, "excessConversionConstant must be >= 1");
    let numerator   = U256::from(excess);
    let denominator = U256::from(k);
    let max_u64     = U256::from(u64::MAX);

    let mut i = U256::from(1u64);
    let mut output = U256::ZERO;
    let mut accum = U256::from(min_price) * denominator;       // ≤ MaxUint128
    let max_output = denominator * max_u64;                    // ≤ MaxUint128

    while accum > U256::ZERO {
        output += accum;                                       // ≤ MaxUint193
        if output >= max_output {
            return u64::MAX;                                   // clamp (>=, not >)
        }
        accum = (accum * numerator) / denominator / i;         // /k THEN /i
        i += U256::from(1u64);
    }
    (output / denominator).to::<u64>()                         // fits: output < max_output
}
```

> Use `ruint::Uint<256,4>` (or `primitive-types::U256`). Do **not** use
> `u128`: intermediates reach `MaxUint193`. `to::<u64>()` is safe only on the
> non-clamped path because `output < max_output = k·MaxUint64` ⇒
> `output/k ≤ MaxUint64`.

### Golden vectors — `CalculatePrice(minPrice, excess, k)`

Verbatim from `vms/components/gas/gas_test.go`:

| minPrice (m) | excess (x) | k | expected price |
|---|---|---|---|
| 1 | 0 | 1 | 1 |
| 1 | 1 | 1 | 2 |
| 1 | 2 | 1 | 6 |
| 1 | 10 000 | 10 000 | 2 |
| 1 | 1 000 000 | 10 000 | `MaxUint64` (clamped) |
| 10 | 10 000 000 | 1 000 000 | 220 264 |
| `MaxUint64` | `MaxUint64` | 1 | `MaxUint64` |
| `MaxUint32` (4 294 967 295) | 1 | 1 | 11 674 931 546 |
| 6 786 177 901 268 885 274 (≈MaxUint64/e) | 1 | 1 | `MaxUint64 − 11` (= 18 446 744 073 709 551 604) |

Note `x = k ⇒ price ≈ m·e` (rows: `1,10000,10000 → 2`; `1,1,1 → 2`), and
`x = 0 ⇒ price = m`. The last row is the most sensitive: the truncation order
yields exactly `MaxUint64 − 11`, so it pins the algorithm bit-exactly.

---

## 1. ACP-103 dynamic gas fee (P-Chain & X-Chain)

> **Spec:** [ACP-103](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/103-dynamic-fees).
> **Code:** `vms/components/gas/*.go` + `vms/platformvm/txs/fee/dynamic_calculator.go`.
> **Activation:** P-Chain `Etna` fork (X-Chain likewise). Pre-Etna uses the static
> fees of §2.

### Pieces

- **Dimensions** (`dimensions.go`): a tx's resource use is a 4-vector
  `[Bandwidth, DBRead, DBWrite, Compute]` (`Dimensions = [u64; 4]`).
- **Weights → Gas** (`Dimensions.ToGas`): collapse to scalar gas via dot product
  `gas = Σ_d complexity[d] · weights[d]` (checked; overflow → error).
- **Fee** (`dynamicCalculator.CalculateFee`): `fee = gas · price`
  (`Gas.Cost` = checked `gas * price`).
- **Price** = `CalculatePrice(minPrice, state.Excess, K)` from §0.
- **State** (`state.go`): `State { capacity: Gas, excess: Gas }` carried on-chain.
  - `AdvanceTime(maxCapacity, capacityRate, targetRate, dt)`:
    `capacity = min(capacity + capacityRate·dt, maxCapacity)`,
    `excess   = sat_sub(excess, targetRate·dt)` (floor 0).
  - `ConsumeGas(g)`: `capacity = capacity − g` (error if `g > capacity`, this is
    `ErrInsufficientCapacity`), `excess = sat_add(excess, g)` (cap `MaxUint64`).

`Gas.AddOverTime/SubOverTime` are the saturating `rate·dt` helpers: on overflow
`AddOverTime` returns `MaxUint64`, `SubOverTime` returns 0.

### Constants (`genesis/*.go`, `gas.Config`) — identical mainnet & Fuji

| Field | Value | Note |
|---|---|---|
| `Weights[Bandwidth]` | 1 | max block ~1 MB |
| `Weights[DBRead]` | 1 000 | ≤1 000 reads/block |
| `Weights[DBWrite]` | 1 000 | ≤1 000 writes/block |
| `Weights[Compute]` | 4 | ~250 ms/block |
| `MaxCapacity` | 1 000 000 | |
| `MaxPerSecond` | 100 000 | refill ~10 s |
| `TargetPerSecond` | 50 000 | = half of max |
| `MinPrice` | 1 | nAVAX/gas |
| `ExcessConversionConstant` (K) | **2 164 043** | "double every 30 s"; `=(MaxPerSecond−TargetPerSecond)·30/ln2`, hard-coded |

`capacityRate = MaxPerSecond`, `targetRate = TargetPerSecond` when advancing.

### Rust sketch

```rust
pub struct GasState { pub capacity: u64, pub excess: u64 }

impl GasState {
    pub fn advance_time(&self, max_cap: u64, cap_rate: u64, tgt_rate: u64, dt: u64) -> GasState {
        GasState {
            capacity: (self.capacity.checked_add(saturating_mul(cap_rate, dt))
                       .unwrap_or(u64::MAX)).min(max_cap),
            excess:   self.excess.saturating_sub(saturating_mul(tgt_rate, dt)),
        }
    }
    pub fn consume_gas(&self, g: u64) -> Result<GasState, FeeError> {
        let capacity = self.capacity.checked_sub(g).ok_or(FeeError::InsufficientCapacity)?;
        Ok(GasState { capacity, excess: self.excess.saturating_add(g) }) // cap == MaxUint64
    }
}
fn saturating_mul(r: u64, dt: u64) -> u64 { r.checked_mul(dt).unwrap_or(u64::MAX) }

pub fn dot_to_gas(c: [u64;4], w: [u64;4]) -> Result<u64, FeeError> {
    let mut g = 0u64;
    for i in 0..4 { g = g.checked_add(c[i].checked_mul(w[i]).ok_or(FeeError::Overflow)?)
                              .ok_or(FeeError::Overflow)?; }
    Ok(g)
}
```

### Worked examples (mainnet constants, K = 2 164 043, minPrice = 1)

1. **Price at excess = 0:** `CalculatePrice(1, 0, K) = 1`. A tx with complexity
   `[600, 1, 1, 1000]` → `gas = 600·1 + 1·1000 + 1·1000 + 1000·4 = 6 600`;
   `fee = 6 600 · 1 = 6 600` nAVAX.
2. **Price after one doubling (excess = K):** `CalculatePrice(1, 2 164 043, 2 164 043) = 2`
   (the `x=k` identity ⇒ `≈ e ≈ 2` after truncation). Same tx → `fee = 13 200`.
3. **Advance-then-consume round trip.** Start `{capacity: 1 000 000, excess: 100 000}`.
   Advance `dt = 1 s`: `capacity = min(1 000 000 + 100 000, 1 000 000) = 1 000 000`,
   `excess = 100 000 − 50 000 = 50 000`. Consume `g = 6 600`:
   `{capacity: 993 400, excess: 56 600}`.

---

## 2. P-Chain static fees + L1 validator continuous fee

### 2a. Static (pre-Etna) tx fees — `SimpleCalculator`

`CalculateFee` returns a flat `txFee` independent of the tx (`simple_calculator.go`).
Genesis-configured (`genesis/params.go::TxFeeConfig`):

| Network | `TxFee` | `CreateAssetTxFee` |
|---|---|---|
| Mainnet | `MilliAvax` = 1 000 000 nAVAX | `10·MilliAvax` = 10 000 000 |
| Fuji | `MilliAvax` | `10·MilliAvax` |
| Local | `MilliAvax` | `MilliAvax` |

(Unit ladder, `utils/units/avax.go`: `NanoAvax=1`, `MicroAvax=1e3`,
`MilliAvax=1e6`, `Avax=1e9`, `KiloAvax=1e12`, `MegaAvax=1e15`.) Trivial in Rust —
a stored `u64` returned as-is.

### 2b. L1 validator continuous fee (ACP-77) — `validators/fee/fee.go`

Each active L1 (Subnet→L1) validator continuously burns balance at a price that
*itself* rises with the number of active validators. This reuses §0's exponential
but **iterates per-second** when `current ≠ target`.

State `State { current: Gas, excess: Gas }` (`current` = # active L1 validators).
Config `Config { capacity, target, minPrice, K }`.

- **`AdvanceTime(target, seconds)`** changes only `excess`:
  - `current < target`: `excess = sat_sub(excess, (target − current)·seconds)` (floor 0)
  - `current > target`: `excess = sat_add(excess, (current − target)·seconds)` (cap MaxU64)
  - `current = target`: unchanged.
- **`CostOf(c, seconds)`** = total fee over `seconds`:
  - If `current == target`: price is constant ⇒ `cost = seconds · CalculatePrice(minPrice, excess, K)` (checked; overflow → MaxU64).
  - Else loop `i` from 0: `AdvanceTime(target, 1)`; **if `excess` hits 0 it stays 0
    forever**, so add `minPrice · remaining_seconds` and return (fast path);
    otherwise `cost += CalculatePrice(minPrice, excess, K)` each second.
- **`SecondsRemaining(c, maxSeconds, funds)`** — inverse: how long `funds` lasts
  (same loop; same zero-excess fast path; `minPrice == 0` ⇒ returns `maxSeconds`).

> **Trap:** when `current != target`, the cost is the **sum of per-second prices
> after advancing one second at a time**, NOT `seconds · price(excess)`. The
> excess is advanced *before* pricing each second. Replicate the loop order
> exactly (and the zero-excess short-circuit, which both bounds runtime and is
> consensus-load-bearing).

### Constants (`genesis/*.go`, `fee.Config`)

| Field | Mainnet | Fuji |
|---|---|---|
| `Capacity` | 20 000 | 20 000 |
| `Target` | 10 000 | 10 000 |
| `MinPrice` | `512·NanoAvax` = 512 | `512·NanoAvax` = 512 |
| `K` (`ExcessConversionConstant`) | **1 246 488 515** ("double every day") | **51 937 021** ("double every hour") |

### Rust sketch

```rust
pub struct L1State { pub current: u64, pub excess: u64 }
pub struct L1Config { pub target: u64, pub min_price: u64, pub k: u64 }

impl L1State {
    fn advance_one(&self, target: u64) -> L1State {
        let excess = if self.current < target {
            self.excess.saturating_sub((target - self.current))     // ·1 second
        } else if self.current > target {
            self.excess.saturating_add(self.current - target)
        } else { self.excess };
        L1State { current: self.current, excess }
    }

    pub fn cost_of(&self, c: &L1Config, seconds: u64) -> u64 {
        if self.current == c.target {
            let p = calculate_price(c.min_price, self.excess, c.k);
            return seconds.checked_mul(p).unwrap_or(u64::MAX);
        }
        let mut s = L1State { current: self.current, excess: self.excess };
        let mut cost: u64 = 0;
        for i in 0..seconds {
            s = s.advance_one(c.target);
            if s.excess == 0 {
                let rem = seconds - i;
                return cost.checked_add(c.min_price.saturating_mul(rem)).unwrap_or(u64::MAX);
            }
            cost = cost.checked_add(calculate_price(c.min_price, s.excess, c.k))
                       .unwrap_or(u64::MAX);
        }
        cost
    }
}
```

### Golden vectors — `validators/fee/fee_test.go` (test constants: `minPrice = 2048`,
`target = 10 000`, `K = excessConversionConstant` "double every day"; values below
use that test's `K`, NOT the genesis `K`):

| state.current | state.excess | seconds | expectedCost | expectedExcess after advance |
|---|---|---|---|---|
| 10 | 0 | 60 (minute) | 122 880 | 0 (no underflow) |
| 10 000 | 0 | 60 | 122 880 | 0 |
| 10 000 | `excessIncreasePerDoubling` | 60 | 245 760 | unchanged |
| 10 000 | K | 60 | 334 020 | K |
| 15 000 | 0 | 60 | 122 880 | 5 000·60 = 300 000 |
| 9 000 | 6h·1000 | 86 400 (day) | 177 321 939 | 0 (underflows mid-window) |
| 10 000 | K | 86 400 | 480 988 800 | K |
| 15 000 | 0 | 86 400 | 211 438 809 | 5 000·86 400 |
| 10 000 | 0 | 604 800 (week) | 1 238 630 400 | 0 |
| `MaxUint64` | 1 | 1 (target 0) | `MaxUint64` (no overflow) | `MaxUint64` |
| `MaxUint32` | 0 | 10 (target 0) | 1 948 429 840 780 833 612 | `MaxUint32·10` |

The `122 880 = 60·2048` rows confirm the constant-price path; the `177 321 939`
row is the critical per-second-loop-with-underflow vector.

---

## 3. P-Chain staking reward — `reward/calculator.go`

> **Code:** `vms/platformvm/reward/{calculator,config}.go`. **Activation:** always
> (Primary-Network minting from genesis). Pure `big.Int` (→ `U256`/`BigUint` in
> Rust), **no exponential** — this is its own formula.

```math
\begin{aligned}
\text{remaining} &= \text{supplyCap} - \text{supply} \\
\text{reward} &= \frac{\text{remaining} \cdot \big(\text{maxSubMin}\cdot\Delta t + \text{minRate}\cdot P\big)
                       \cdot \text{stake} \cdot \Delta t}
                      {P \cdot D \cdot \text{supply} \cdot P}
\end{aligned}
```

where `Δt = stakedDuration` (nanoseconds), `P = MintingPeriod` (ns),
`D = consumptionRateDenominator = PercentDenominator = 1 000 000`,
`maxSubMin = MaxConsumptionRate − MinConsumptionRate`, `minRate = MinConsumptionRate`.

### Exact integer order (must match `big.Int` op order, `calculator.go`)

```text
adjConsumNum = maxSubMin * Δt + minRate * P
adjConsumDen = P * D
reward = remaining                # u256/bignum
reward *= adjConsumNum
reward *= stake
reward *= Δt
reward /= adjConsumDen
reward /= supply
reward /= P
if reward > MaxUint64: return remaining        # cap
return min(remaining, reward as u64)
```

> **Traps:** (1) **All multiplications happen before any division** — do not
> simplify or reorder; truncation points are part of consensus. (2) Final divides
> are three separate steps `/adjConsumDen`, `/supply`, `/P`. (3) Result is
> `min(remaining, reward)` and is capped at `remaining` if it doesn't fit `u64`.
> (4) `Δt` is in **nanoseconds** (`time.Duration`), `P` likewise.

`Split(total, shares)` (delegator/validator fee split) — `shares ≤ PercentDenominator`:

```text
remainderShares = PercentDenominator - shares
remainderAmount = (remainderShares * total) / PercentDenominator   # checked; on overflow:
                  = remainderShares * (total / PercentDenominator)  # fallback (less precise)
fromShares      = total - remainderAmount
```

### Constants (`genesis/*.go`, `reward.Config`) — identical mainnet & Fuji

| Field | Value |
|---|---|
| `MaxConsumptionRate` | 0.12·1e6 = 120 000 |
| `MinConsumptionRate` | 0.10·1e6 = 100 000 |
| `MintingPeriod` (P) | 365·24 h = 31 536 000 s = 31 536 000 000 000 000 ns |
| `SupplyCap` | 720·MegaAvax = 720 000 000 000 000 000 nAVAX |
| `PercentDenominator` (D) | 1 000 000 |

### Rust sketch

```rust
use ruint::aliases::U256;
pub struct RewardCalc { max_sub_min: u64, min_rate: u64, minting_period: u64, supply_cap: u64 }
const PERCENT_DENOM: u64 = 1_000_000;

impl RewardCalc {
    /// staked_duration & minting_period in nanoseconds.
    pub fn calculate(&self, staked_duration_ns: u64, stake: u64, supply: u64) -> u64 {
        let p = U256::from(self.minting_period);
        let dt = U256::from(staked_duration_ns);
        let adj_num = U256::from(self.max_sub_min) * dt
                    + U256::from(self.min_rate) * p;
        let adj_den = p * U256::from(PERCENT_DENOM);
        let remaining = self.supply_cap - supply;            // caller guarantees supply<=cap

        let mut r = U256::from(remaining);
        r *= adj_num; r *= U256::from(stake); r *= dt;        // ALL muls first
        r /= adj_den; r /= U256::from(supply); r /= p;        // THEN three divides

        match u64::try_from(r) { Ok(v) => v.min(remaining), Err(_) => remaining }
    }
}
```

### Worked examples (mainnet config)

Let `P = 31 536 000 000 000 000` ns, `D = 1e6`, `cap = 720e15`.

1. **Full-period stake.** `Δt = P` (1 year), `stake = 2 000·1e9 = 2e12`,
   `supply = 400e15`. Then `adjConsumNum = 20 000·P + 100 000·P = 120 000·P`
   ⇒ effective rate `120 000/1e6 = 0.12`. `remaining = 320e15`. Plugging through:
   `reward = 320e15 · 0.12 · (2e12 / 400e15) · 1 ≈ 192 000 000 000` nAVAX
   = 192 AVAX (verify against Go with the exact integer pipeline — this is the
   approximate target a golden test must reproduce bit-exactly).
2. **Zero duration** (`Δt = 0`): `reward = 0` (the `·Δt` factor zeroes it).
3. **Supply near cap.** `supply = 719e15`, `remaining = 1e15`, full period,
   `stake = 1e15`: reward is `min(remaining, computed)`; if `computed > remaining`
   it's clamped to `1e15`. Use this to test the `min(remaining, …)` cap.

> Generate the canonical golden table by running `reward.NewCalculator(cfg).Calculate`
> over a grid of `(Δt, stake, supply)` and freezing outputs (see `calculator_test.go`).

---

## 4. EVM Apricot Phase 3 fee window + Apricot Phase 4 block gas cost

> **Code:** `graft/coreth/plugin/evm/customheader/{dynamic_fee_windower,block_gas_cost}.go`
> + `upgrade/{ap3,ap4,ap5,etna}`. **Activation:** `ApricotPhase3` (base-fee window),
> `ApricotPhase4` (block gas cost), modified by `ApricotPhase5`, `Etna`, `Granite`,
> `Fortuna`. All `big.Int` → `U256`.

### 4a. AP3 rolling gas window + base fee (`baseFeeFromWindow`, `feeWindow`)

A **10-second rolling window** `Window = [u64; 10]` (`ap3/window.go`) tracks gas
consumed. Operations:
- `Add(amounts…)`: saturating-add into the **last** slot (overflow → `MaxUint64`).
- `Shift(n)`: drop the oldest `n` slots (`n ≥ 10` ⇒ all-zero), zero-fill the front.
- `Sum()`: saturating sum of all slots.

Window update per block (`feeWindow`): given `timeElapsed = timestamp − parent.Time`,
```text
window  = parse(parent.Extra)            # genesis / first-AP3 block -> Window{}
window.Add(parent.GasUsed, parentExtraStateGasUsed, blockGasCost)
window.Shift(timeElapsed)
```
where `blockGasCost` is: `ap3.IntrinsicBlockGas (1_000_000)` pre-AP4; the AP4
`BlockGasCostWithStep(parentCost, ap4.BlockGasCostStep, timeElapsed)` in AP4;
`0` from AP5 on (block cost is paid via tips, not added to the window).

Base fee update (`baseFeeFromWindow`), with `target` & `denom` per phase:
```text
totalGas = window.Sum()
baseFee  = parent.BaseFee
if totalGas == target: return baseFee          # exact-target => unchanged (even out of bounds!)
if totalGas > target:
    delta = max(1, (totalGas - target) * parent.BaseFee / target / denom)
    baseFee += delta
else:
    delta = max(1, (target - totalGas) * parent.BaseFee / target / denom)
    windowsElapsed = timeElapsed / 10          # ap3.WindowLen
    if windowsElapsed > 1: delta *= windowsElapsed
    baseFee -= delta
clamp baseFee to [minBaseFee, maxBaseFee] for the active phase   # (see traps)
```

> **Traps:** (1) `totalGas == target` returns **early & unclamped** — "for legacy
> reasons" baseFee can stay outside the nominal bounds. (2) `delta` floored at
> `common.Big1` (so fee always moves by ≥1 wei when off-target). (3) **Increase
> direction is NOT multiplied by `windowsElapsed`; only the decrease is.**
> (4) Two separate divides `/target` then `/denom` (truncation order matters).
> (5) The first AP3 block / genesis returns `InitialBaseFee` and an empty window.
> (6) Phase selection for bounds & denom keys off `parent.Time` (mostly), while
> activation/return keys off the child `timestamp` — match `IsX(parent.Time)`
> vs `IsX(timestamp)` exactly as in the Go switch.

Per-phase constants (`upgrade/*`, gwei = 1e9 wei):

| Constant | AP3 | AP5 | Etna |
|---|---|---|---|
| `WindowLen` | 10 s | 10 | 10 |
| `TargetGas` (window sum target) | 10 000 000 | 15 000 000 | 15 000 000 |
| `BaseFeeChangeDenominator` | 12 | 36 | 36 |
| `MinBaseFee` | 75 gwei | 25 gwei (ap4) | **1 gwei** (etna) |
| `MaxBaseFee` | 225 gwei | `MaxUint256` (uncapped from AP5) | `MaxUint256` |
| `InitialBaseFee` | 225 gwei (=AP3 MaxBaseFee) | — | — |
| `IntrinsicBlockGas` | 1 000 000 | dynamic (AP4) | — |

(AP4 also sets `MinBaseFee = 25 gwei`, `MaxBaseFee = 1000 gwei`; AP4 still **caps**
to `ap4MaxBaseFee` whereas AP5+ uses `MaxUint256`.)

### 4b. AP4 block gas cost (`ap4.BlockGasCost`, `BlockGasCostWithStep`)

A per-block surcharge that rises when blocks come faster than `TargetBlockRate`
and falls when slower:

```math
\text{cost} = \mathrm{clamp}_{[0,\,10^6]}\!\left(\text{parentCost} \pm \text{step}\cdot|\,2 - \Delta t\,|\right)
```
sign `−` if `Δt > 2` (slower → cheaper), `+` if `Δt < 2` (faster → costlier),
where `Δt = timeElapsed` (seconds since parent), `step = BlockGasCostStep`.

```text
deviation = |TargetBlockRate - timeElapsed|         # AbsDiff, TargetBlockRate=2
change    = step * deviation                          # overflow -> MaxUint64
if timeElapsed > 2: cost = parentCost - change  (underflow -> MinBlockGasCost=0)
else:               cost = parentCost + change  (overflow  -> MaxBlockGasCost)
return clamp(cost, MinBlockGasCost=0, MaxBlockGasCost=1_000_000)
```
`parentCost == nil` (AP3/AP4 boundary, new network) ⇒ returns `MinBlockGasCost = 0`.
In **Granite** `BlockGasCost` returns `0` outright (mechanism retired).

Constants: `TargetBlockRate = 2 s`, `MinBlockGasCost = 0`, `MaxBlockGasCost = 1_000_000`,
`BlockGasCostStep = 50 000` (AP4) → `200 000` (AP5). `ap5.AtomicGasLimit = 100 000`,
`ap5.AtomicTxIntrinsicGas = 10 000`.

`VerifyBlockFee` (block_gas_cost.go): the block's total effective tips, divided by
`baseFee`, must purchase ≥ `requiredBlockGasCost` units of gas:
`Σ_i tipPremium_i · gasUsed_i (+ extraContribution)) / baseFee ≥ requiredBlockGasCost`.

### Rust sketch

```rust
#[derive(Clone, Copy, Default)]
pub struct Window([u64; 10]);
impl Window {
    pub fn add(&mut self, amounts: &[u64]) {
        let last = &mut self.0[9];
        for &a in amounts { *last = last.saturating_add(a); }
    }
    pub fn shift(&mut self, n: u64) {
        if n >= 10 { *self = Window::default(); return; }
        let n = n as usize; let mut w = [0u64; 10];
        w[..10 - n].copy_from_slice(&self.0[n..]);
        *self = Window(w);
    }
    pub fn sum(&self) -> u64 { self.0.iter().fold(0u64, |a, &v| a.saturating_add(v)) }
}

pub fn ap4_block_gas_cost(parent_cost: Option<u64>, step: u64, dt: u64) -> u64 {
    let parent = match parent_cost { None => return 0 /*MinBlockGasCost*/, Some(p) => p };
    const TARGET: u64 = 2; const MAX: u64 = 1_000_000;
    let deviation = if TARGET > dt { TARGET - dt } else { dt - TARGET };
    let change = step.checked_mul(deviation).unwrap_or(u64::MAX);
    let cost = if dt > TARGET {
        parent.checked_sub(change).unwrap_or(0)      // default MinBlockGasCost
    } else {
        parent.checked_add(change).unwrap_or(MAX)    // default MaxBlockGasCost
    };
    cost.min(MAX)                                    // (>=Min is implied; Min==0)
}
```

### Worked examples

**AP3 base fee (AP5 params: target = 15 000 000, denom = 36):**
1. Window sum = target exactly ⇒ `baseFee` unchanged.
2. Window sum = 30 000 000 (2× target), `parent.BaseFee = 100 gwei`:
   `delta = max(1, (30M−15M)·100e9 / 15M / 36)` = `max(1, 15M·100e9 / 15M /36)`
   = `max(1, 100e9/36)` = `2 777 777 777` wei ⇒ `baseFee = 102 777 777 777` wei.
3. Window sum = 0, `parent.BaseFee = 100 gwei`, `timeElapsed = 25 s`:
   `delta = max(1, 15M·100e9/15M/36) = 2 777 777 777`; `windowsElapsed = 25/10 = 2 > 1`
   ⇒ `delta = 5 555 555 554`; `baseFee = 94 444 444 446` wei.

**AP4 block gas cost (step = 50 000):**
1. `parentCost = 100 000`, `dt = 2` (on target): `deviation = 0` ⇒ `cost = 100 000`.
2. `parentCost = 100 000`, `dt = 1` (fast): `cost = 100 000 + 50 000 = 150 000`.
3. `parentCost = 100 000`, `dt = 10` (slow): `change = 50 000·8 = 400 000`;
   `cost = sat_sub(100 000, 400 000) = 0`.

---

## 5. EVM ACP-176 / Fortuna dynamic fee — `vms/evm/acp176/acp176.go`

> **Code:** `vms/evm/acp176/acp176.go`, wired by
> `customheader/{dynamic_fee_state,base_fee}.go`. **Activation:** `Fortuna`
> (seconds granularity), `Granite` (millisecond granularity via
> `AdvanceMilliseconds`). Replaces the AP3 window. Reuses §0 + §1's `gas.State`.

On-chain state `State { gas: gas.State{capacity, excess}, targetExcess: Gas (q) }`,
serialized as 24 bytes (3×u64, big-endian).

### The two-level exponential

The **target itself** is exponential in `targetExcess` (lets governance move the
target smoothly), and the **price** is exponential in gas excess with `K` scaled
by the target:

```math
\begin{aligned}
T &= \text{Target} = \text{CalculatePrice}(P,\; q,\; D) \quad (\approx P\cdot e^{q/D})\\
R &= \text{maxPerSecond} = T \cdot \text{TargetToMax} \\
C &= \text{maxCapacity} = R \cdot \text{TimeToFillCapacity} \\
K &= T \cdot \text{TargetToPriceUpdateConversion} \\
\text{gasPrice} &= \text{CalculatePrice}(M,\; \text{gas.excess},\; K) \quad (\approx M\cdot e^{x/K})
\end{aligned}
```

Constants (`acp176.go`):

| Name | Symbol | Value |
|---|---|---|
| `MinTargetPerSecond` | P | 1 000 000 |
| `MaxTargetExcessDiff` | Q | 1<<15 = 32 768 |
| `MaxTargetChangeRate` | | 1 024 |
| `TargetConversion` | D | `1024 · 32768 = 33 554 432` |
| `MinGasPrice` | M | 1 |
| `TimeToFillCapacity` | | 5 s |
| `TargetToMax` | | 2 |
| `TargetToPriceUpdateConversion` | | 87 (≈60/ln2 ⇒ price doubles ≤~60 s) |
| `TargetToMaxCapacity` | | `2·5 = 10` |
| `maxTargetExcess` | | 1 024 950 627 |

`mulWithUpperBound(a,b)` = checked `a·b`, saturating to `MaxUint64`.

### Operations

- `Target()` = `CalculatePrice(P=1e6, targetExcess, D=33 554 432)`.
- `GasPrice()` = `CalculatePrice(M=1, gas.excess, K = mulUB(Target, 87))`.
- `AdvanceSeconds(s)`: `gas.AdvanceTime(C, R, T, s)` with `R = mulUB(T,2)`,
  `C = mulUB(R,5)`, `T = Target()`. (§1's `AdvanceTime`.)
- `AdvanceMilliseconds(ms)`: per-ms rates `targetPerMS = T/1000`,
  `maxPerMS = targetPerMS·2`, but `C` still computed from the per-second `R`.
- `ConsumeGas(gasUsed, extraGasUsed)`: `gas.ConsumeGas` (each separately; errors on
  insufficient capacity; `extraGasUsed` must fit u64).
- `UpdateTargetExcess(desired)`: move `targetExcess` toward `desired` by at most
  `Q = 32 768` per block (`targetExcess(excess, desired)`); then **rescale gas
  excess** so price is continuous:
  `gas.excess = scaleExcess(gas.excess, newT, oldT) = floor(excess·newT / oldT)`
  (U256; saturates to MaxU64); finally cap `capacity ≤ C(newT)`.
- `DesiredTargetExcess(desiredTarget)`: binary search over `[0, maxTargetExcess)`
  for the least `q` with `Target(q) ≥ desiredTarget` (no float).

> **Traps:** (1) `targetExcess` step is clamped to `±Q` per block — a desired jump
> takes many blocks. (2) `scaleExcess` uses U256 (`excess·newT` can exceed u64) and
> rounds **down** (floor), unlike SAE's ceil (§6). (3) `K = T·87` is itself
> saturating; for huge `T` it caps at `MaxUint64`, after which price is ~flat at M.
> (4) `MaxCapacity` floor: `MinTargetPerSecond·TargetToMaxCapacity = 10 000 000`.

### Rust sketch

```rust
const P: u64 = 1_000_000; const D: u64 = 33_554_432;
const M: u64 = 1; const Q: u64 = 1 << 15;
const T2MAX: u64 = 2; const FILL: u64 = 5; const T2PRICE: u64 = 87;

pub struct Acp176 { pub gas: GasState, pub target_excess: u64 }
fn mul_ub(a: u64, b: u64) -> u64 { a.checked_mul(b).unwrap_or(u64::MAX) }

impl Acp176 {
    pub fn target(&self) -> u64 { calculate_price(P, self.target_excess, D) }
    pub fn gas_price(&self) -> u64 {
        let k = mul_ub(self.target(), T2PRICE);
        calculate_price(M, self.gas.excess, k)
    }
    pub fn advance_seconds(&mut self, s: u64) {
        let t = self.target(); let r = mul_ub(t, T2MAX); let c = mul_ub(r, FILL);
        self.gas = self.gas.advance_time(c, r, t, s);
    }
    pub fn update_target_excess(&mut self, desired: u64) {
        let old_t = self.target();
        let diff = self.target_excess.abs_diff(desired).min(Q);
        self.target_excess = if self.target_excess < desired
            { self.target_excess + diff } else { self.target_excess - diff };
        let new_t = self.target();
        // scaleExcess: floor(excess * new_t / old_t), U256, saturate u64
        let scaled = (U256::from(self.gas.excess) * U256::from(new_t)) / U256::from(old_t);
        self.gas.excess = u64::try_from(scaled).unwrap_or(u64::MAX);
        self.gas.capacity = self.gas.capacity.min(mul_ub(new_t, T2MAX * FILL));
    }
}
```

### Worked examples

1. **Baseline target.** `targetExcess = 0` ⇒ `Target = CalculatePrice(1e6, 0, D) = 1e6`.
   Then `R = 2e6`, `C = 10e6`, `K = 1e6·87 = 87 000 000`.
2. **Price at empty/full excess.** With `K = 87 000 000`:
   `gas.excess = 0` ⇒ `gasPrice = CalculatePrice(1, 0, K) = 1` (= M, the floor).
   `gas.excess = K = 87 000 000` ⇒ `gasPrice = CalculatePrice(1, K, K) = 2`
   (the `x=k` doubling identity).
3. **Target excess step clamp.** From `targetExcess = 0`, `UpdateTargetExcess(1_000_000)`:
   change `= min(1 000 000, 32 768) = 32 768` ⇒ new `targetExcess = 32 768`;
   `newT = CalculatePrice(1e6, 32 768, 33 554 432)`. Since `q/D = 32768/33554432 = 1/1024`,
   `newT ≈ 1e6·e^{1/1024} ≈ 1 000 977`. Then `gas.excess` is rescaled by
   `floor(excess·1 000 977 / 1 000 000)`.

---

## 6. SAE gas-as-time pricing — `vms/saevm/gastime`

> **Code:** `vms/saevm/gastime/{gastime,acp176,config}.go`, `proxytime/proxytime.go`,
> `intmath/intmath.go`. **Spec:** ACP-194 (a "continuous" ACP-176). **Activation:**
> SAE genesis (the saevm). See §11 for full context; this section consolidates the
> formulas with worked examples. Reuses §0's `CalculatePrice` (via the thin
> `calculatePrice(x,k) = CalculatePrice(1, x, k)`).

SAE measures *time* in gas: consuming `target · TargetToRate` gas = 1 second of
proxy-time. State (`gastime.Time`): proxy-time `+ target (T) + excess (x) + config`.

Constants (`config.go`, `gastime.go`):

| Name | Value |
|---|---|
| `TargetToRate` | 2 (rate `R = 2·T`) |
| `MinTarget` | 1 (clamp floor) |
| `MaxTarget` | `MaxUint64 / 2` |
| `DefaultTargetToExcessScaling` | 87 (K = scaling·T) |
| `DefaultMinPrice` (M) | 1 |

### Formulas

- **Excess scaling factor** `K = excessScalingFactor() = BoundedMultiply(TargetToExcessScaling, T, MaxUint64)`.
- **Price** = `max(MinPrice, calculatePrice(excess, K))` = `max(M, CalculatePrice(1, x, K))`.
  (Floor at `MinPrice` because for tiny `K`/large M the exp can't represent M.)
- **`Tick(g)`** (consume `g` gas in a block): advance proxy-time by `g`, and
  ```math
  x \mathrel{+}= \left\lfloor \frac{g\,(R-T)}{R} \right\rfloor,\quad x \le \text{MaxUint64}
  ```
  `intmath.MulDiv(g, R-T, R)` (overflow-free since `(R−T)/R < 1`), bounded-add.
  With `R = 2T`, `(R−T)/R = 1/2`, so excess grows by `⌊g/2⌋` per consumed gas
  unit above none — i.e. half of consumed gas accrues to excess.
- **`FastForwardTo(s, f)`** (time passes, `s` seconds + `f` fractional rate units):
  reduce excess by `s·T + ⌊f·T/R⌋`, each bounded at 0 (matches ACP's `−T·dt`).
- **`AfterBlock(used, target, c)`**: `Tick(used)`; if not static, **rescale excess**
  to the new target via `scaleExcess` (U256, **rounds up**: `⌈oldX·newK / oldK⌉`,
  `K = T·scaling`); set new `target`, `rate = 2·target`, enforce min-excess.
- **`enforceMinExcess()`**: ensure `excess ≥ excessForPrice(MinPrice, K)`, where
  `excessForPrice` **binary-searches** the least `x` with `calculatePrice(x,K) ≥ p`
  (honoring the lower price if integer approximation overshoots). `p ≤ 1 ⇒ 0`.

> **Traps vs EVM ACP-176 (§5):** (1) SAE's `scaleExcess` rounds **up** (adds
> `oldK−1` before dividing) and uses `K = T·scaling`; EVM's rounds **down** and
> uses raw `T`. Do not share the routine. (2) `Tick` uses `MulDiv` (floor) with
> `(R−T)`, not the EVM `AdvanceTime`. (3) `enforceMinExcess` binary search must
> match Go's `lo/hi` bisection and the "honor lower price → return `lo−1`" branch
> bit-for-bit. (4) `BeforeBlock` converts the block's nanosecond component via
> `MulDivCeil(ns, rate, 1e9)` (**ceil**) — sub-second alignment is ceil-rounded.

### Rust sketch

```rust
const TARGET_TO_RATE: u64 = 2;
pub struct GasTime { proxy: ProxyTime, target: u64, excess: u64, scaling: u64, min_price: u64, static_pricing: bool }

impl GasTime {
    fn k(&self) -> u64 { self.scaling.checked_mul(self.target).unwrap_or(u64::MAX) } // BoundedMultiply
    pub fn price(&self) -> u64 { self.min_price.max(calculate_price(1, self.excess, self.k())) }

    pub fn tick(&mut self, g: u64) {
        self.proxy.tick(g);
        if self.static_pricing { return; }
        let r = self.target * TARGET_TO_RATE;           // rate
        // MulDiv floor: g*(r-t)/r  (overflow-free, ratio<1)
        let quo = mul_div_floor(g, r - self.target, r);
        self.excess = self.excess.checked_add(quo).unwrap_or(u64::MAX); // BoundedAdd
    }
    // scaleExcess (ceil): ⌈oldX*newK/oldK⌉, saturate u64  — used in after_block
}
```

### Worked examples (scaling = 87, MinPrice = 1)

1. **Target = 1e6 ⇒ K = 87e6.** `excess = 0` ⇒ `price = max(1, CalculatePrice(1,0,87e6)) = 1`.
   `excess = 87e6` ⇒ `price = max(1, CalculatePrice(1, 87e6, 87e6)) = 2`.
2. **Tick accrual.** Target = 1e6 ⇒ R = 2e6. Consume `g = 1 000 000` gas in a block:
   `Δexcess = ⌊1 000 000·(2e6 − 1e6)/2e6⌋ = ⌊1 000 000·1/2⌋ = 500 000`.
   New excess from 0 = 500 000; price = `CalculatePrice(1, 500 000, 87e6) = 1`
   (still ≈1; needs ~87e6 excess to double).
3. **Fast-forward decay.** Target = 1e6, R = 2e6, `excess = 1 000 000`. Time passes
   `s = 1` second, `f = 0`: `excess −= s·T = 1·1 000 000 = 1 000 000` (bounded 0) ⇒
   excess = 0; then `enforceMinExcess` keeps it ≥ `excessForPrice(1, 87e6) = 0`.

---

## 7. Mechanism → fork → crate

Fork gating uses the shared `Fork`/`UpgradeConfig` model in §03 / §11.

| # | Mechanism | Activating fork | Pre-fork fallback | Go package | Rust crate (per §03 map) |
|---|---|---|---|---|---|
| 1 | ACP-103 dynamic gas fee (P/X) | `Etna` | static §2a | `vms/components/gas`, `vms/platformvm/txs/fee` | `ava-gas`, `ava-platformvm` |
| 2a | P-Chain static tx fee | genesis | — | `vms/platformvm/txs/fee` | `ava-platformvm` |
| 2b | L1 validator continuous fee | `Etna` (ACP-77) | — | `vms/platformvm/validators/fee` | `ava-platformvm` |
| 3 | P-Chain staking reward | genesis | — | `vms/platformvm/reward` | `ava-platformvm` |
| 4a | EVM AP3 base-fee window | `ApricotPhase3` (mods AP5/Etna) | nil base fee | `coreth/.../customheader`, `upgrade/ap3` | `ava-coreth`/`ava-evm` |
| 4b | EVM AP4 block gas cost | `ApricotPhase4` (mods AP5; 0 in Granite) | nil | `coreth/.../upgrade/ap4` | `ava-coreth`/`ava-evm` |
| 5 | EVM ACP-176 dynamic fee | `Fortuna` (ms in `Granite`) | AP3 window §4a | `vms/evm/acp176` | `ava-evm-acp176` |
| 6 | SAE gas-as-time | SAE genesis | — | `vms/saevm/gastime` | `ava-saevm` |

All six share the §0 exponential except #3 (own bignum formula) and #4 (linear
window + linear block cost). #1, #2b, #5, #6 all call `gas::calculate_price`.

---

## 8. Go→Rust mapping

| Go | Rust |
|---|---|
| `gas.Gas`, `gas.Price` (`uint64`) | `u64` newtypes (or transparent `u64`) |
| `gas.Dimensions [4]uint64` | `[u64; 4]` |
| `uint256.Int` (holiman) | `ruint::aliases::U256` (or `primitive-types::U256`) |
| `math/big.Int` (reward) | `ruint` U256 / `num-bigint::BigUint` |
| `safemath.{Add,Sub,Mul}` → `(v, err)` | `checked_{add,sub,mul}` → `Option` |
| `safemath.AbsDiff` | `a.abs_diff(b)` |
| `Gas.AddOverTime/SubOverTime` (saturating) | `saturating_*` / `checked_*().unwrap_or(MAX/0)` |
| `intmath.BoundedAdd/Multiply/Subtract` | `checked_*().unwrap_or(ceil)` / `.min/.max` |
| `intmath.MulDiv/MulDivCeil` | `mul_div_floor/ceil` (U256 intermediate) |
| `time.Duration` (ns, reward) | `u64` nanoseconds (keep ns, do not convert to f64) |

> Newtype `Gas`/`Price` improve safety but make the §0 U256 conversions noisier;
> either is acceptable as long as the arithmetic is identical.

---

## 9. Test plan

**Golden tests (freeze the tables in this doc):**
- §0: the 9-row `CalculatePrice` table (esp. the `MaxUint64 − 11` row — pins
  truncation order) and the clamp row.
- §1: the advance/consume round trips.
- §2b: the full `validators/fee/fee_test.go` table (the `177 321 939` underflow row).
- §3: a `(Δt, stake, supply)` grid frozen from `reward.Calculate`.
- §4: AP3 base-fee deltas (incl. exact-target no-op, increase-vs-decrease windows-
  elapsed asymmetry) and AP4 cost (on/fast/slow + clamp-to-0).
- §5/§6: target/price at `excess ∈ {0, K}`, the `±Q` target-excess clamp, scale-
  excess round direction (down for §5, up for §6).

**Proptest invariants:**
- `CalculatePrice` monotone non-decreasing in `excess`; equals `minPrice` at
  `excess=0`; never panics; result ≤ `MaxUint64`.
- AP3 base fee stays within `[min,max]` bounds **except** the exact-target case;
  off-target ⇒ moves by ≥1.
- AP4 cost ∈ `[0, 1_000_000]`; faster-than-target ⇒ non-decreasing, slower ⇒
  non-increasing.
- ACP-176/SAE: price continuous across `UpdateTargetExcess`/`AfterBlock` (rescale
  keeps price within ±1 of pre-update); `targetExcess` moves ≤ `Q` per block.
- Reward ≤ `remaining`; reward is 0 when `Δt = 0`; monotone non-decreasing in
  `stake` and in `Δt`.

**Differential vs Go (ref §02):** drive a CGO/`go test`-emitted harness that, over
random *block sequences*, advances each mechanism (random gas used, random Δt,
random target moves) and compares Rust vs Go state+price every block. Concretely:
seed a PRNG, generate N=10⁴ blocks, assert byte-identical serialized state
(`State.Bytes()` / window `Bytes()`) and identical price/fee at each step. This is
the only way to catch the order-of-operations and rounding traps flagged above.

## 10. Perf notes

- §0 `CalculatePrice` is hot (called per-tx and per-block, sometimes per-second in
  §2b loops). The series converges in ≤ ~`x/k + O(log)` iterations; for typical
  `x ≈ k` it's ~30–40 iterations of U256 mul/div. Keep it allocation-free
  (`ruint` is stack `[u64;4]`); the Go version explicitly does zero allocations.
- §2b `CostOf`/`SecondsRemaining` loop per-second; the **zero-excess fast path** is
  essential — without it a week-long query is 604 800 `CalculatePrice` calls.
  Replicate it for both correctness *and* performance.
- §3 reward and §5/§6 `scaleExcess` use U256; one multiply chain each — negligible.
- The AP3 window is a fixed `[u64;10]`; `Shift` is a small copy. No bignum on the
  hot path except `parent.BaseFee` (already a big int in geth/libevm).
