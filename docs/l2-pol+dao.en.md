# L2 POL+DAO Specification

This specification defines L2 POL+DAO systems that issue custom L2 tokens (L2PDT - L2 POL DAO Tokens) while establishing L2 Protocol-Owned Liquidity paired with the L1 Native token. L2 POL+DAOs may use various emission mechanisms including UTBC (Unidirectional Token Bonding Curve) or custom minting strategies. Core innovations include declining voting power, progressive participation rewards, and per-block micro-streaming for MEV elimination.

---

## Table of Contents

1. [Core Definitions](#1-core-definitions)
2. [System Invariants](#2-system-invariants)
3. [Governance Mechanics](#3-governance-mechanics)
4. [Treasury Integration](#4-treasury-integration)
5. [L2 POL+DAO Lifecycle](#5-l2-poldao-lifecycle)
6. [BLDR Pattern](#6-bldr-pattern)
7. [Security Model](#7-security-model)
8. [Configuration Parameters](#8-configuration-parameters)
9. [Implementation Requirements](#9-implementation-requirements)

---

## 1. Core Definitions

### Primary Entities

- **Native (L1)** — Parachain base token governed by first-order referenda at L1 level
- **L2 POL+DAO** — Second-level DAO maintaining L2 protocol-owned liquidity paired with L1 Native
- **Ecosystem L2 POL+DAO** — L2 POL+DAO initialized by ecosystem origin, receiving L1 Treasury funding
- **L2PDT** — L2 POL DAO Token issued by an L2 POL+DAO with mandatory POL support
- **L1 POL** — Protocol-Owned Liquidity at L1 (Native) level
- **L2 POL** — Second-level Protocol-Owned Liquidity in L2 POL+DAO
- **BLDR** — Canonical L2PDT builder token for ecosystem payroll and development

### Mechanisms

- **UTBC** — Unidirectional Token Bonding Curve, one possible emission mechanism for L2PDT (mint-only, no redemption)
- **Custom Emission** — Alternative minting mechanisms configured per L2 POL+DAO requirements
- **DripVault** — Per-block streaming contract for continuous treasury operations
- **Declining Power** — Voting weight decreasing from 10x to 1x over the voting period
- **Progressive Rewards** — Redistribution creating 1:2 ratio between passive and active participants
- **Team Shares** — Locked allocations with governance rights but no transfer ability

---

## 2. System Invariants

Every certified L2 POL+DAO MUST enforce these on-chain invariants:

### 2.1 L2 POL Management

LP tokens marked as L2 POL can be managed in two modes:

**Locked Mode:**

- LP tokens require for withdrawal:
  - L2 POL+DAO supermajority approval (≥66%)
  - Mandatory timelock (14 days)
  - L1 Native veto window (7 days)

**Treasury Mode:**

- LP tokens held on L2 DAO Treasury balance
- Governed by DAO voting mechanisms
- Flexibility configured at POL+DAO initialization

### 2.2 Native Floor Liquidity

Initial L2 POL must satisfy minimum requirements configured via L1 POL+DAO pallet parameters (set through L1 referendum):

```
L2_POL_native ≥ configured_minimum_native
```

This ensures XYK mathematics remain valid and prevents dust attacks.

### 2.3 L2 POL+DAO Protection via L1 Treasury

When L1 Treasury holds L2PDT tokens of any L2 POL+DAO:

- L1 Treasury L2PDT holdings receive 10x voting multiplier (unified for all L2 POL+DAOs)
- Provides ecosystem-level protection for both ecosystem-initiated and user-created POL+DAOs
- Security level determined by actual L1 Treasury holdings, not origin type
- Simplified security model without complex cap calculations

### 2.4 Team Share Requirements

If team allocation exists:

- Minimum 5-year vesting from last mint
- Full governance voting rights during vesting
- No transfers until vesting complete
- Counts toward cap protection calculations

### 2.5 Telemetry Transparency

Real-time on-chain emission of:

- L2 POL depth (Native and L2PDT reserves)
- Treasury balances (including locked team shares)
- Voting power multipliers and decay state
- Participation rates and reward distributions

### 2.6 Evolutionary Design

- System parameters and mechanisms are configurable via governance
- Protocol can evolve when better mechanics are discovered
- Changes require consensus through established governance paths
- Evolution by Design: continuous improvement is a core principle

---

## 3. Governance Mechanics

### 3.1 Declining Voting Power

Voting weight decays linearly from 10x to 1x over different periods (applies to both L1 and L2 referendums independently):

```rust
fn calculate_voting_power(
    vote_time: Timestamp,
    vote_start: Timestamp,
    vote_end: Timestamp,
    base_weight: Balance,
    is_ecosystem: bool
) -> Balance {
    let progress = (vote_time - vote_start) / (vote_end - vote_start);
    let multiplier = 10.0 - (9.0 * progress);

    // Apply 10x multiplier for L1 Treasury holdings
    let final_multiplier = if is_l1_treasury {
        multiplier * 10.0  // 100x to 10x range for L1 Treasury
    } else {
        multiplier
    };

    base_weight * final_multiplier
}
```

**Properties:**

- Early commitment receives maximum 10x multiplier (100x for L1 Treasury holdings)
- L2 POL+DAO votes: 7-day period by default
- L1 Native POL+DAO votes: 14-day period when involved
- Linear decay prevents last-minute manipulation
- L1 Treasury gets 10x super-multiplier on any L2PDT holdings (unified model)

### 3.2 Progressive Participation Rewards

Active governance participants receive enhanced rewards:

```rust
fn distribute_rewards(epoch: EpochId) -> Result<(), Error> {
    let base_reward = calculate_base_staking_reward();
    let active_voters = count_active_participants(epoch);
    let passive_stakers = total_stakers - active_voters;

    // Non-participants receive 50% of base
    let passive_reward = base_reward * 0.5;
    let total_passive_payout = passive_reward * passive_stakers;

    // Redistribution pool from penalties
    let penalty_pool = (base_reward * passive_stakers) - total_passive_payout;

    // Active voters share base + redistribution
    let active_reward = base_reward + (penalty_pool / active_voters);

    // Result: ~2x rewards for active participation
    distribute(passive_stakers, passive_reward);
    distribute(active_voters, active_reward);
}
```

### 3.3 Treasury Voting Multiplier

**L1 Treasury Holdings (All L2 POL+DAOs):**

- When L1 Treasury holds L2PDT tokens of any L2 POL+DAO, they receive 10x voting weight
- Applies universally whether L2 POL+DAO was ecosystem-initiated or user-created
- Native holders vote in Track 2 to decide how L1 Treasury deploys its L2PDT votes
- Protection level scales with actual L2PDT amount in L1 Treasury
- Creates economic incentive for L2 POL+DAOs to have L1 Treasury participation

**L2 POL+DAO Governance:**

- L2 POL+DAO uses standard declining voting power (10x to 1x) for all direct holders
- Critical decisions can be overridden by L1 via super user veto
- System incentivizes cooperation through proxy superiority mechanics

### 3.4 Governance Resolution Mechanisms

**Super User Veto Power:**

- Super user can veto any L2 POL+DAO referendum (both user-created and ecosystem-initiated)
- Veto immediately cancels the referendum without further voting
- Provides ultimate safety mechanism for critical security issues

**Two-Track Voting System:**
When L1 Treasury holds L2PDT tokens, L2 POL+DAO uses a dual-track voting mechanism:

**Track 1: Direct L2PDT Voting**

- L2PDT holders vote directly with declining power (10x to 1x)
- Standard L2 POL+DAO governance rules apply
- Produces a winning position (FOR or AGAINST) with total vote strength

**Track 2: Native Proxy Voting**

- Native (L1) holders vote on how L1 Treasury should deploy its L2PDT holdings
- Decision determines whether L1 Treasury L2PDT votes for/against the proposal
- L1 Treasury L2PDT receives constant 10x multiplier

Both tracks run simultaneously for 7 days, then the system determines the outcome:

**Path 1: Consensus (tracks align) - 7 days total**

- Both tracks vote the same way (both FOR or both AGAINST)
- L1 Treasury L2PDT receives 10x multiplier
- Votes sum: Track 1 votes + (L1 Treasury L2PDT × 10)
- Immediate execution after 7 days

**Path 2: Divergence with Proxy Superiority (tracks conflict) - 7 days total**

When tracks vote opposite ways AND Track 2 with 10x is stronger:

- Example: Track 1 votes 8000 AGAINST, Track 2 votes 1000 L2PDT FOR (×10 = 10000)
- Track 2 proxy votes (10000 FOR) exceed Track 1 winner (8000 AGAINST)
- Result: Proxy position wins, Track 1 minority votes ignored
- Final: 10000 FOR wins, immediate execution (proxy superiority rule)

**Path 3: Divergence with Execution Delay (tracks conflict) - 14 days total**

When tracks vote opposite ways BUT Track 1 remains stronger:

- Example: Track 1 votes 8000 AGAINST, Track 2 votes 500 L2PDT FOR (×10 = 5000)
- Track 1 winner (8000 AGAINST) exceeds Track 2 proxy votes (5000 FOR)
- Result: Track 1 position wins but with 7-day execution delay
- Delay provides time for L1 DAO to initiate veto referendum if needed

Critical actions requiring L1 POL+DAO involvement:

- L2 POL withdrawal or modification
- Bonding curve parameter changes
- Treasury spending above threshold
- Governance rule modifications

---

## 4. Treasury Integration

### 4.1 Per-Block Micro-Streaming

Eliminates MEV through continuous micro-transactions:

```rust
struct DripVault {
    total_allocation: Balance,
    blocks_remaining: BlockNumber,
    amount_per_block: Balance,
}

impl DripVault {
    fn initialize(allocation: Balance, duration_blocks: BlockNumber) -> Self {
        Self {
            total_allocation: allocation,
            blocks_remaining: duration_blocks,
            amount_per_block: allocation / duration_blocks,
        }
    }

    fn execute_block(&mut self) -> Balance {
        if self.blocks_remaining > 0 {
            self.blocks_remaining -= 1;
            self.amount_per_block  // Amount too small for profitable MEV
        } else {
            0
        }
    }
}
```

**Configuration Example:**

```
2-year stream: 2 * 365 * 7200 blocks = 5,256,000 blocks
Per-block amount: 10,000 NATIVE / 5,256,000 = 0.0019 NATIVE
MEV profit: price_impact(0.0019) - gas_cost < 0 ✓
```

(amount_per_block, 0)
}

### 4.2 Treasury Participation Modes

1. **One-shot seed** — Immediate swap for strategic position
2. **DripVault stream** — Continuous support over months/years
3. **Revenue recycling** — Percentage of income auto-converts to L2PDT

---

## 5. L2 POL+DAO Lifecycle

### 5.1 Genesis Phase

**Registration requires:**

- Origin type (ecosystem-initiated or user-created)
- Emission mechanism configuration (UTBC parameters, custom minting rules, etc.)
- Share distribution (user, l2_pol, treasury, team)
- Team vesting schedule (minimum 5 years)
- Governance configuration
- Initial Native for L2 POL floor (minimum set via L1 referendum)

**Validation checks:**

- Shares sum to exactly 100%
- L2 POL share ≥ 20%
- Initial Native meets minimum requirement (configured via L1 POL+DAO pallet parameters)
- Ecosystem origin verified if claiming ecosystem funding

### 5.2 Launch Phase

**Activation mint sequence:**

1. Lock initial Native contribution
2. Execute first mint through configured emission mechanism (UTBC or custom)
3. Create XYK pool with L2 POL allocation
4. Configure LP tokens per POL management mode (locked or treasury-held)
5. Distribute treasury and team allocations
6. Activate governance mechanisms

### 5.3 Operational Phase

**Continuous operations:**

- Each mint enforces cap check and routes to L2 POL
- Proposals follow dual approval path for critical actions
- Participation rewards calculate and distribute each epoch
- DripVault executes per-block streaming if configured

### 5.4 Maturation Metrics

L2 POL+DAO considered mature when:

- L2 POL depth > 100,000 NATIVE equivalent
- 6 months of positive cash flow
- Average participation rate > 30%
- No critical proposals vetoed for 3 months

---

## 6. BLDR Pattern

### 6.1 Architecture

BLDR serves as the canonical L2 POL+DAO implementation (using UTBC) for transparent payroll:

```rust
struct BldrConfig {
    team_share: Permill,        // 0% for pure treasury model
    treasury_share: Permill,    // 100% if no team
    invoice_cooldown: BlockNumber,
    max_invoice_size: Balance,
}
```

### 6.2 Invoice Flow

1. **Submission** — Contributor submits invoice with deliverables hash
2. **Voting** — BLDR holders vote on invoice approval
3. **Approval** — If passed, enters execution queue
4. **Payment** — Automated disbursement in BLDR or Native
5. **Tracking** — On-chain reference to work completed

### 6.3 Economic Loop

```
Treasury seeds BLDR → Contributors earn → Some sell for Native →
XYK price dips → Treasury buyback → Relock with multiplier →
Stronger governance → Better decisions → More value → Repeat
```

---

## 7. Security Model

### 7.1 Attack Vectors and Mitigations

| Attack Vector          | Mitigation                         | Cost Formula                               |
| ---------------------- | ---------------------------------- | ------------------------------------------ |
| **Mint-whale capture** | L1 Treasury 10x multiplier         | `cost > l1_treasury_holdings × 10`         |
| **Flash governance**   | Declining voting power             | `cost = full_position × 10 × lock_time`    |
| **L1 Native takeover** | Veto override for critical actions | `cost > native_market_cap × quorum%`       |
| **MEV exploitation**   | Per-block streaming                | `profit < 0` (unprofitable)                |
| **Treasury drain**     | Three-path resolution + thresholds | Requires L1 override or L2 POL+DAO consent |

### 7.2 Economic Security

Minimum attack cost for any L2 POL+DAO:

```
Attack_cost = min(
    l1_treasury_l2pdt × 10,      // If L1 Treasury holds L2PDT
    circulating_l2pdt × price × 5, // Need >50% with 1x vs early 10x voters
    native_market_cap × 0.1,      // L1 Native takeover
    opportunity_cost(10x_lockup)  // Time-weighted capital
)
```

**Key insight:** Protection scales with L1 Treasury investment, not origin type. User-created POL+DAOs gain ecosystem-level protection when L1 Treasury acquires their tokens.

### 7.3 Defense Layers

1. **Mathematical** — Ecosystem L2 POL+DAOs: 10x multiplier ensures Native Treasury dominance
2. **Economic** — Attack cost exceeds potential gain through multiple mechanisms
3. **Temporal** — Three-path resolution provides appropriate response windows
4. **Social** — Transparent operations enable monitoring
5. **Hierarchical** — Ecosystem origin provides additional security layer

---

## 8. Configuration Parameters

### 8.1 Recommended Defaults

```rust
const DEFAULT_CONFIG: L2PolDaoConfig = L2PolDaoConfig {
    // Shares
    l2_pol_share: Permill::from_percent(33),
    treasury_share: Permill::from_percent(33),
    user_share: Permill::from_percent(34),
    team_share: Permill::from_percent(0),  // Or up to 11%

    // Governance
    ecosystem_multiplier: 10,  // For ecosystem L2 POL+DAOs
    treasury_multiplier: 3,    // For regular L2 POL+DAOs
    voting_period: 7 * DAYS,
    native_voting_period: 14 * DAYS,
    veto_override_threshold: 14 * DAYS,

    // Economics
    min_native_floor: 1000 * UNITS,
    min_mint_amount: 100 * UNITS,
    buyback_threshold: Permill::from_percent(95),

    // Streaming
    default_drip_duration: 2 * YEARS,
    blocks_per_drip: 1,
};
```

### 8.2 Governance Schemas

**Ecosystem L2 POL+DAO Schema:**

- Native Treasury receives 10x voting multiplier
- Three-path resolution (L2 POL+DAO-only, consensus, or veto)
- Suitable for ecosystem-funded projects

**Regular L2 POL+DAO Schema:**

- Standard declining voting power (10x to 1x)
- Optional Native POL+DAO involvement
- Suitable for community-initiated projects

**Pure L1-Proxy Schema:**

- All decisions via L1 POL+DAO referendum
- L2PDT purely economic
- Suitable for maximum security requirements

---

## 9. Implementation Requirements

### 9.1 Core Pallets

```rust
trait L2PolDaoPallet {
    // Lifecycle
    fn register_dao(config: L2PolDaoConfig, origin: Origin) -> Result<DaoId, Error>;
    fn activate_dao(dao_id: DaoId, initial_native: Balance) -> Result<(), Error>;

    // Minting (implementation varies by emission mechanism)
    fn mint(dao_id: DaoId, amount: Balance) -> Result<Balance, Error>;
    fn custom_emission(dao_id: DaoId, params: EmissionParams) -> Result<Balance, Error>;

    // Governance
    fn cast_vote(proposal: ProposalId, weight: Balance, conviction: u8);
    fn calculate_voting_power(
        voter: AccountId,
        time: Timestamp,
        is_l1_treasury: bool
    ) -> Balance;
    fn resolve_proposal(proposal: ProposalId) -> ResolutionPath;

    // L2 POL Management
    fn add_liquidity_to_l2_pol(dao_id: DaoId, native: Balance, l2pdt: Balance);
    fn get_l2_pol_reserves(dao_id: DaoId) -> (Balance, Balance);
}
```

### 9.2 Critical Hooks

```rust
trait Hooks {
    fn on_mint(dao_id: DaoId, amount: Balance);
    fn on_vote_cast(voter: AccountId, power: Balance);
    fn on_proposal_executed(proposal: ProposalId);
    fn on_participation_reward(voter: AccountId, reward: Balance);
}
```

### 9.3 Storage

- L2 POL reserves and LP token tracking
- Voting power decay schedules
- Participation history for rewards
- Treasury and team vesting schedules
- DripVault streaming states

---

## Conclusion

L2 POL+DAO architecture enables second-level DAOs with mathematical security guarantees through:

1. **Declining voting power** preventing last-minute governance attacks
2. **Per-block streaming** eliminating MEV entirely
3. **Progressive rewards** incentivizing active participation
4. **L2 POL immutability** ensuring permanent second-level liquidity
5. **Dual-track governance** with execution delay for potential L1 veto intervention
6. **Unified 10x multiplier** for L1 Treasury holdings in any L2 POL+DAO
7. **Evolution by Design** enabling continuous protocol improvement

The system creates sustainable token economies where L2PDT tokens carry governance power with mandatory L2 Protocol-Owned Liquidity support, while treasury participation provides alignment without traditional team allocations, maintaining security through economic incentives rather than bureaucratic structures.

---

**Version:** 1.0.0
**Date:** October 2025
**License:** MIT
