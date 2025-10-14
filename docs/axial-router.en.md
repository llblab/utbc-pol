# Axial Router 1.0 Specification

## Abstract

Axial Router is a minimalist multi-token routing system optimized for UTBC ecosystems. Operating exclusively within the parachain's internal liquidity pools, it provides intelligent path discovery and mechanism selection through Native token as the sole routing anchor. The router seamlessly integrates `pallet-asset-conversion` for XYK pools and custom UTBC pallet for unidirectional bonding curves. Key features include separation of path from swap mechanism, TVL-weighted EMA price protection with half-life decay, and systematic fee burning for deflation.

---

## 1. Design Principles

### 1.1 Core Philosophy

1. **Radical Simplicity** - Minimal viable routing with zero bloat
2. **Clean Abstractions** - Path discovery separate from mechanism selection
3. **Economic Purity** - Best price wins, no artificial preferences
4. **Native-Only Anchor** - Native token as the sole routing hub
5. **Essential Protection** - TVL-weighted EMA with half-life decay prevents manipulation
6. **Atomic Execution** - All operations complete in single transaction

### 1.2 System Architecture

```
┌──────────────────────────────────────┐
│           Axial Router 1.0           │
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │     Path Discovery Engine      │  │
│  ├────────────────────────────────┤  │
│  │   Mechanism Selection Layer    │  │
│  ├────────────────────────────────┤  │
│  │  TVL-Weighted EMA w/ Half-life │  │
│  ├────────────────────────────────┤  │
│  │      Fee Burning Manager       │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼                       ▼
   [XYK Pools]            [UTBC Curves]
(pallet-asset-conversion) (pallet-utbc)
```

### 1.3 Liquidity Sources

The router integrates **2 types of liquidity sources**:

1. **XYK Pools** - Classical x×y=k AMM pools via `pallet-asset-conversion`
   - Bidirectional swaps between any token pair
   - Standard 0.3% trading fee

2. **UTBC Curves** - Unidirectional bonding curves via custom `pallet-utbc`
   - Foreign → Native minting only
   - Cannot sell Native back to curve
   - Transfer of reserve asset to POL

---

## 2. Data Structures

```rust
/// Precision constant for all price calculations
const PRECISION: u128 = 1_000_000_000_000;  // 10^12
```

### 2.1 Core Configuration

```rust
pub struct RouterConfig {
    /// Native token as the sole routing anchor
    pub native_asset: AssetId,

    /// Maximum price deviation allowed (in basis points)
    pub max_deviation_bps: u32,  // Default: 2000 (20%)

    /// Router fee for burning (in basis points)
    pub router_fee_bps: u32,  // Default: 30 (0.3%)

    /// Half-life for EMA weight decay (in blocks)
    pub ema_half_life: BlockNumber,  // Default: 100 blocks
}
```

### 2.2 Path and Mechanism Separation

```rust
/// A complete route from source to destination
pub struct Route {
    /// The token path to follow
    pub path: Vec<AssetId>,

    /// The specific hops with chosen mechanisms
    pub hops: Vec<Hop>,

    /// Total expected output
    pub expected_output: Balance,
}

/// A single hop in the route
pub struct Hop {
    /// Source token
    pub from: AssetId,

    /// Destination token
    pub to: AssetId,

    /// The swap mechanism to use
    pub mechanism: SwapMechanism,

    /// Expected output for this hop
    pub expected_output: Balance,
}

/// Available swap mechanisms
pub enum SwapMechanism {
    /// Standard XYK pool swap
    XykPool {
        pool_id: PoolId,
    },

    /// UTBC minting (only Foreign → Native)
    UtbcMint,
}
```

### 2.3 Price Oracle

```rust
/// Per-token EMA tracking with half-life decay
pub struct TokenEma {
    /// EMA price of token in native units
    pub native_price: Balance,

    /// Total value locked across all pools (in native units)
    pub total_tvl: Balance,

    /// Block number of last update
    pub last_update: BlockNumber,
}
```

### 2.4 Interfaces

```rust
/// Interface for pallet-asset-conversion XYK pools
pub trait AssetConversionApi<AccountId, AssetId, Balance> {
    /// Get pool ID for a token pair
    fn get_pool_id(asset_a: AssetId, asset_b: AssetId) -> Option<PoolId>;

    /// Quote price for exact input swap
    fn quote_price_exact_tokens_for_tokens(
        asset_in: AssetId,
        asset_out: AssetId,
        amount_in: Balance,
        include_fee: bool,
    ) -> Option<Balance>;

    /// Execute swap with exact input amount
    fn swap_exact_tokens_for_tokens(
        who: AccountId,
        path: Vec<AssetId>,
        amount_in: Balance,
        min_amount_out: Balance,
        recipient: AccountId,
        keep_alive: bool,
    ) -> DispatchResult;
}

/// Interface for custom UTBC pallet (unidirectional)
pub trait UtbcInterface<AccountId, AssetId, Balance> {
    /// Check if token has bonding curve
    fn has_curve(asset_id: AssetId) -> bool;

    /// Calculate how much Native user receives for Foreign payment
    fn calculate_user_receives(
        foreign_asset: AssetId,
        foreign_amount: Balance
    ) -> Result<Balance, DispatchError>;

    /// Execute mint through bonding curve
    fn mint_with_distribution(
        who: &AccountId,
        foreign_asset: AssetId,
        foreign_amount: Balance,
    ) -> Result<Balance, DispatchError>;
}

/// Interface for fee management
pub trait FeeManagerInterface<AccountId, AssetId, Balance> {
    /// Receive and process fee in specified asset
    fn receive_fee(asset_id: AssetId, amount: Balance) -> DispatchResult;

    /// Query total amount burned
    fn total_burned() -> Balance;
}
```

### 2.5 Fee Management

```rust
/// Handles router fee collection and burning for deflationary pressure
pub struct FeeManager;

impl FeeManager {
    /// Minimum amount worth converting and burning
    const MIN_BURN_AMOUNT: Balance = 1_000;

    /// Receive fee in any asset
    pub fn receive_fee(
        asset_id: AssetId,
        amount: Balance,
    ) -> DispatchResult {
        // Add to fee buffer for this asset
        FeeBuffers::<T>::mutate(asset_id, |buffer| {
            *buffer = buffer.saturating_add(amount);
        });

        // Try to process accumulated fees
        Self::process_fee_buffer(asset_id)
    }

    /// Process accumulated fees for burning
    fn process_fee_buffer(asset_id: AssetId) -> DispatchResult {
        let buffer_amount = FeeBuffers::<T>::get(asset_id);

        if buffer_amount < Self::MIN_BURN_AMOUNT {
            return Ok(());  // Wait for more fees to accumulate
        }

        // If native asset, burn directly
        if asset_id == T::NativeAsset::get() {
            T::Assets::burn(asset_id, buffer_amount)?;
            FeeBuffers::<T>::remove(asset_id);

            TotalBurned::<T>::mutate(|total| {
                *total = total.saturating_add(buffer_amount);
            });

            Self::deposit_event(Event::FeeBurned {
                asset: asset_id,
                amount: buffer_amount,
            });
        } else {
            // Convert to native first via XYK pool
            match T::Pools::swap_to_native(asset_id, buffer_amount) {
                Ok(native_amount) => {
                    FeeBuffers::<T>::remove(asset_id);
                    T::Assets::burn(T::NativeAsset::get(), native_amount)?;

                    TotalBurned::<T>::mutate(|total| {
                        *total = total.saturating_add(native_amount);
                    });

                    Self::deposit_event(Event::FeeBurned {
                        asset: T::NativeAsset::get(),
                        amount: native_amount,
                    });
                }
                Err(_) => {
                    // Keep in buffer if swap fails
                    return Ok(());
                }
            }
        }

        Ok(())
    }

    /// Query total burned amount
    pub fn total_burned() -> Balance {
        TotalBurned::<T>::get()
    }
}
```

---

## 3. Route Discovery

### 3.1 Path Discovery Engine

The router first discovers possible paths, then selects optimal mechanisms for each hop:

```rust
impl<T: Config> Pallet<T> {
    pub fn find_best_route(
        from: AssetId,
        to: AssetId,
        amount: Balance,
    ) -> Result<Route, Error> {
        let native = T::NativeAsset::get();

        // Step 1: Discover possible paths
        let paths = vec![
            vec![from, to],                // Direct path
            vec![from, native, to],        // Through Native
        ];

        // Step 2: Build routes with optimal mechanisms
        let mut best_route = None;
        let mut best_output = 0;

        for path in paths {
            if let Ok(route) = Self::build_route_with_mechanisms(&path, amount) {
                if route.expected_output > best_output {
                    best_output = route.expected_output;
                    best_route = Some(route);
                }
            }
        }

        best_route.ok_or(Error::<T>::NoRouteFound)
    }

    /// Build a route by selecting best mechanism for each hop
    fn build_route_with_mechanisms(
        path: &[AssetId],
        initial_amount: Balance,
    ) -> Result<Route, Error> {
        let mut hops = Vec::new();
        let mut current_amount = initial_amount;

        // For each consecutive pair in path
        for window in path.windows(2) {
            let from = window[0];
            let to = window[1];

            let hop = Self::find_best_hop(from, to, current_amount)?;
            current_amount = hop.expected_output;
            hops.push(hop);
        }

        Ok(Route {
            path: path.to_vec(),
            hops,
            expected_output: current_amount,
        })
    }

    /// Find the best mechanism for a single hop
    fn find_best_hop(
        from: AssetId,
        to: AssetId,
        amount: Balance,
    ) -> Result<Hop, Error> {
        let mut best_mechanism = None;
        let mut best_output = 0;

        // Option 1: XYK Pool
        if let Some(pool_id) = T::AssetConversion::get_pool_id(from, to) {
            if let Some(output) = T::AssetConversion::quote_price_exact_tokens_for_tokens(
                from, to, amount, true
            ) {
                best_mechanism = Some(SwapMechanism::XykPool { pool_id });
                best_output = output;
            }
        }

        // Option 2: UTBC Minting (only for Foreign → Native)
        if Self::is_foreign(from) && to == T::NativeAsset::get() {
            if let Ok(output) = T::UtbcPallet::calculate_user_receives(from, amount) {
                if output > best_output {
                    best_mechanism = Some(SwapMechanism::UtbcMint);
                    best_output = output;
                }
            }
        }

        best_mechanism
            .map(|mechanism| Hop {
                from,
                to,
                mechanism,
                expected_output: best_output,
            })
            .ok_or(Error::<T>::NoRouteFound)
    }

    /// Check if asset is a foreign token (not Native)
    fn is_foreign(asset: AssetId) -> bool {
        asset != T::NativeAsset::get()
    }
}
```

### 3.2 Mechanism Selection Criteria

For each hop, the router compares all available mechanisms and selects the one with highest output. No artificial preferences or biases are applied - pure economic efficiency drives selection.

---

## 4. Price Oracle

### 4.1 Half-Life EMA Implementation

The router maintains per-token exponentially weighted moving average prices using TVL-weighted pricing across all pools, with half-life decay for smooth trust degradation:

```rust
impl<T: Config> Pallet<T> {
    /// Calculate EMA weight based on age using half-life decay
    fn calculate_ema_weight(age_blocks: BlockNumber) -> u128 {
        let half_life = T::EmaHalfLife::get();

        if age_blocks == 0 {
            return PRECISION;
        }

        // Weight = 2^(-age/half_life) * PRECISION
        // For computational efficiency, approximate with linear segments
        let decay_periods = age_blocks / half_life;
        let remainder = age_blocks % half_life;

        // Each period halves the weight
        let base_weight = PRECISION >> decay_periods.min(40);  // Cap at 2^-40

        // Linear interpolation for partial period
        let partial_decay = remainder.saturating_mul(base_weight) / (half_life * 2);
        base_weight.saturating_sub(partial_decay)
    }

    /// Update token EMA using TVL-weighted pricing
    pub fn update_token_ema(
        token_id: AssetId,
    ) -> DispatchResult {
        // Skip native token (always 1:1 with itself)
        if token_id == T::NativeAsset::get() {
            return Ok(());
        }

        // Calculate TVL-weighted average price across all pools
        let (weighted_price, total_tvl) = Self::calculate_tvl_weighted_price(token_id)?;

        if total_tvl == 0 {
            return Ok(());  // No liquidity, skip update
        }

        TokenEmas::<T>::mutate(token_id, |maybe_ema| {
            match maybe_ema {
                Some(ema) => {
                    let age = frame_system::Pallet::<T>::block_number()
                        .saturating_sub(ema.last_update);

                    // Calculate weight based on age (half-life decay)
                    let old_weight = Self::calculate_ema_weight(age);
                    let new_weight = PRECISION;
                    let total_weight = old_weight.saturating_add(new_weight);

                    // Weighted average: (old_price * old_weight + new_price * new_weight) / total
                    ema.native_price = ema.native_price
                        .saturating_mul(old_weight)
                        .saturating_add(weighted_price.saturating_mul(new_weight))
                        .saturating_div(total_weight.max(1));

                    ema.total_tvl = total_tvl;
                    ema.last_update = frame_system::Pallet::<T>::block_number();
                },
                None => {
                    *maybe_ema = Some(TokenEma {
                        native_price: weighted_price,
                        total_tvl,
                        last_update: frame_system::Pallet::<T>::block_number(),
                    });
                }
            }
        });

        Ok(())
    }

    /// Calculate TVL-weighted average price for a token
    fn calculate_tvl_weighted_price(
        token_id: AssetId,
    ) -> Result<(Balance, Balance), DispatchError> {
        let mut weighted_sum = 0u128;
        let mut total_tvl = 0u128;

        // Iterate through all pools containing this token
        for pool_id in T::Pools::get_pools_for_token(token_id) {
            let (reserve_token, reserve_native) = if Self::is_native_pair(pool_id, token_id) {
                // Direct pair with native
                T::Pools::get_reserves(pool_id)?
            } else {
                // Indirect pair - need to calculate native value
                Self::calculate_native_value_in_pool(pool_id, token_id)?
            };

            if reserve_token > 0 {
                // Price = native_reserve / token_reserve
                let price = reserve_native
                    .saturating_mul(PRECISION)
                    .saturating_div(reserve_token);

                // Weight by TVL (2 * native_reserve for XYK pools)
                let pool_tvl = reserve_native.saturating_mul(2);

                weighted_sum = weighted_sum.saturating_add(
                    price.saturating_mul(pool_tvl).saturating_div(PRECISION)
                );
                total_tvl = total_tvl.saturating_add(pool_tvl);
            }
        }

        if total_tvl > 0 {
            let weighted_price = weighted_sum
                .saturating_mul(PRECISION)
                .saturating_div(total_tvl);
            Ok((weighted_price, total_tvl))
        } else {
            Ok((0, 0))
        }
    }

    /// Validate that token prices are within acceptable deviation
    pub fn validate_price_deviation(
        route: &Route,
    ) -> bool {
        let config = RouterConfiguration::<T>::get();

        // Calculate implied exchange rate from route
        let from_amount = route.hops.first().map(|h| h.expected_output).unwrap_or(0);
        let to_amount = route.expected_output;

        if from_amount == 0 {
            return false;
        }

        let implied_rate = to_amount
            .saturating_mul(PRECISION)
            .saturating_div(from_amount);

        // Get EMA prices for tokens
        let from_token = route.path.first().unwrap();
        let to_token = route.path.last().unwrap();

        let from_ema = Self::get_token_ema_price(*from_token);
        let to_ema = Self::get_token_ema_price(*to_token);

        // Calculate expected rate from EMAs
        let expected_rate = from_ema
            .saturating_mul(PRECISION)
            .saturating_div(to_ema.max(1));

        // Check deviation
        let max_deviation = config.max_deviation_bps as u128;
        let lower_bound = expected_rate
            .saturating_mul(10000u128.saturating_sub(max_deviation))
            .saturating_div(10000);
        let upper_bound = expected_rate
            .saturating_mul(10000u128.saturating_add(max_deviation))
            .saturating_div(10000);

        implied_rate >= lower_bound && implied_rate <= upper_bound
    }

    /// Get token EMA price with half-life weighting
    fn get_token_ema_price(token_id: AssetId) -> Balance {
        // Native token is always 1:1 with itself
        if token_id == T::NativeAsset::get() {
            return PRECISION;
        }

        if let Some(ema) = TokenEmas::<T>::get(token_id) {
            let current_block = frame_system::Pallet::<T>::block_number();
            let age = current_block.saturating_sub(ema.last_update);

            // Get weight based on age (half-life decay)
            let ema_weight = Self::calculate_ema_weight(age);

            // If weight is still significant (>12.5%), use EMA
            if ema_weight > PRECISION / 8 {
                return ema.native_price;
            }
        }

        // Fallback to current TVL-weighted price
        Self::calculate_tvl_weighted_price(token_id)
            .map(|(price, _)| price)
            .unwrap_or(PRECISION)
    }

    /// Check if pool is a direct native pair
    fn is_native_pair(pool_id: PoolId, token_id: AssetId) -> bool {
        let (token_a, token_b) = T::Pools::get_pool_assets(pool_id)
            .unwrap_or((AssetId::default(), AssetId::default()));

        let native = T::NativeAsset::get();
        (token_a == native && token_b == token_id) ||
        (token_a == token_id && token_b == native)
    }

    /// Calculate native value for non-native pairs through routing
    fn calculate_native_value_in_pool(
        pool_id: PoolId,
        token_id: AssetId,
    ) -> Result<(Balance, Balance), DispatchError> {
        let (token_a, token_b) = T::Pools::get_pool_assets(pool_id)?;
        let (reserve_a, reserve_b) = T::Pools::get_reserves(pool_id)?;

        // Determine which reserve corresponds to our token
        let (token_reserve, other_token, other_reserve) = if token_a == token_id {
            (reserve_a, token_b, reserve_b)
        } else {
            (reserve_b, token_a, reserve_a)
        };

        // Get the native value of the other token
        let other_native_price = Self::get_token_ema_price(other_token);

        // Calculate native value of the other side of the pool
        let native_value = other_reserve
            .saturating_mul(other_native_price)
            .saturating_div(PRECISION);

        Ok((token_reserve, native_value))
    }
}
```

---

## 5. Swap Execution

### 5.1 Standard Swap

```rust
impl<T: Config> Pallet<T> {
    #[pallet::call_index(0)]
    #[pallet::weight(T::WeightInfo::swap())]
    pub fn swap(
        origin: OriginFor<T>,
        from: AssetId,
        to: AssetId,
        amount_in: Balance,
        min_amount_out: Balance,
        deadline: T::BlockNumber,
    ) -> DispatchResult {
        let who = ensure_signed(origin)?;

        // Validations
        ensure!(!Self::is_paused(), Error::<T>::RouterPaused);
        ensure!(from != to, Error::<T>::IdenticalAssets);
        ensure!(amount_in > 0, Error::<T>::ZeroAmount);
        ensure!(
            frame_system::Pallet::<T>::block_number() <= deadline,
            Error::<T>::DeadlinePassed
        );

        // Collect router fee
        let router_fee = T::RouterFeeBps::get()
            .saturating_mul(amount_in)
            .saturating_div(10000);
        let amount_after_fee = amount_in.saturating_sub(router_fee);

        T::FeeManager::receive_fee(from, router_fee)?;

        // Find best route (path + mechanisms)
        let route = Self::find_best_route(from, to, amount_after_fee)?;

        // Validate price before execution
        ensure!(
            Self::validate_price_deviation(&route),
            Error::<T>::ExcessivePriceDeviation
        );

        // Execute and validate slippage
        let amount_out = Self::execute_route(&who, &route)?;
        ensure!(amount_out >= min_amount_out, Error::<T>::SlippageExceeded);

        // Update token EMAs
        Self::update_token_ema(from)?;
        Self::update_token_ema(to)?;

        Self::deposit_event(Event::SwapExecuted {
            who,
            from,
            to,
            amount_in,
            amount_out,
            route,
        });

        Ok(())
    }
}
```

### 5.2 Route Execution

```rust
impl<T: Config> Pallet<T> {
    fn execute_route(
        who: &T::AccountId,
        route: &Route,
    ) -> Result<Balance, DispatchError> {
        let mut current_balance = T::Assets::balance(route.path[0], who);

        // Execute each hop
        for hop in &route.hops {
            current_balance = Self::execute_hop(who, hop, current_balance)?;
        }

        Ok(current_balance)
    }

    fn execute_hop(
        who: &T::AccountId,
        hop: &Hop,
        amount_in: Balance,
    ) -> Result<Balance, DispatchError> {
        match &hop.mechanism {
            SwapMechanism::XykPool { pool_id } => {
                T::AssetConversion::swap_exact_tokens_for_tokens(
                    who.clone(),
                    vec![hop.from, hop.to],
                    amount_in,
                    1,  // min_amount_out already validated
                    who.clone(),
                    false,  // keep_alive
                )?;
                Ok(T::Assets::balance(hop.to, who))
            },

            SwapMechanism::UtbcMint => {
                // Only valid for Foreign → Native
                ensure!(
                    Self::is_foreign(hop.from) && hop.to == T::NativeAsset::get(),
                    Error::<T>::InvalidMechanism
                );

                T::UtbcPallet::mint_with_distribution(
                    who,
                    hop.from,
                    amount_in,
                )
            },
        }
    }

    /// Quote expected output for a route without executing
    fn quote_route(route: &Route) -> Balance {
        route.expected_output
    }
}
```

---

## 6. Storage

```rust
/// Token EMA tracking with TVL-weighted pricing
#[pallet::storage]
pub type TokenEmas<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    AssetId,
    TokenEma,
    OptionQuery,
>;

/// Fee buffers awaiting conversion/burning
#[pallet::storage]
pub type FeeBuffers<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    AssetId,
    Balance,
    ValueQuery,
>;

/// Total amount of native tokens burned
#[pallet::storage]
pub type TotalBurned<T: Config> = StorageValue<_, Balance, ValueQuery>;

/// Router configuration
#[pallet::storage]
pub type RouterConfiguration<T: Config> = StorageValue<_, RouterConfig, ValueQuery>;
```

---

## 7. Events and Errors

### 7.1 Events

```rust
/// Events emitted by the router
#[pallet::event]
pub enum Event<T: Config = ()> {
    /// Swap successfully executed
    SwapExecuted {
        who: T::AccountId,
        from: AssetId,
        to: AssetId,
        amount_in: Balance,
        amount_out: Balance,
        route: Route,
    },

    /// Token EMA updated
    TokenEmaUpdated {
        token: AssetId,
        native_price: Balance,
        tvl: Balance,
    },

    /// Fee burned
    FeeBurned {
        asset: AssetId,
        amount: Balance,
    },
}
```

### 7.2 Errors

```rust
#[pallet::error]
pub enum Error<T> {
    /// No viable route found between tokens
    NoRouteFound,

    /// Identical source and target assets
    IdenticalAssets,

    /// Amount is zero
    ZeroAmount,

    /// Insufficient liquidity in pools
    InsufficientLiquidity,

    /// Output amount below minimum acceptable
    SlippageExceeded,

    /// Transaction deadline passed
    DeadlinePassed,

    /// Price deviates too much from EMA
    ExcessivePriceDeviation,

    /// Invalid mechanism for given token pair
    InvalidMechanism,

    /// Arithmetic overflow
    Overflow,

    /// Router is paused
    RouterPaused,
}
```

---

## 8. Configuration

### 8.1 Pallet Configuration

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
    /// Runtime events
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;

    /// Asset management interface
    type Assets: Inspect<Self::AccountId, AssetId = AssetId, Balance = Balance> +
                 Transfer<Self::AccountId> +
                 Burn<Self::AccountId>;

    /// Asset conversion pools (pallet-asset-conversion)
    type AssetConversion: AssetConversionApi<
        Self::AccountId,
        AssetId,
        Balance,
    >;

    /// UTBC pallet for bonding curves
    type UtbcPallet: UtbcInterface<Self::AccountId, AssetId, Balance>;

    /// Native asset ID
    #[pallet::constant]
    type NativeAsset: Get<AssetId>;

    /// Maximum price deviation in basis points (default: 2000 = 20%)
    #[pallet::constant]
    type MaxDeviationBps: Get<u32>;

    /// Router fee in basis points (default: 30 = 0.3%)
    #[pallet::constant]
    type RouterFeeBps: Get<u32>;

    /// Half-life for EMA weight decay (default: 100 blocks)
    #[pallet::constant]
    type EmaHalfLife: Get<Self::BlockNumber>;

    /// Fee manager for burning router fees
    type FeeManager: FeeManagerInterface<Self::AccountId, AssetId, Balance>;

    /// Weight information
    type WeightInfo: WeightInfo;
}
```

### 8.2 Genesis Configuration

```rust
#[pallet::genesis_config]
pub struct GenesisConfig {
    pub native_asset: AssetId,
    pub max_deviation_bps: u32,
    pub router_fee_bps: u32,
    pub ema_half_life: BlockNumber,
}

#[pallet::genesis_build]
impl<T: Config> GenesisBuild<T> for GenesisConfig {
    fn build(&self) {
        // Initialize router configuration
        RouterConfiguration::<T>::put(RouterConfig {
            native_asset: self.native_asset,
            max_deviation_bps: self.max_deviation_bps,
            router_fee_bps: self.router_fee_bps,
            ema_half_life: self.ema_half_life,
        });
    }
}
```

---

## 9. Security Considerations

### 9.1 Price Manipulation

- Per-token EMA with TVL-weighted pricing
- Global deviation limit (20% default)
- Half-life decay for smooth trust degradation
- Pre-execution validation prevents sandwich attacks
- TVL weighting prioritizes larger, stable pools

### 9.2 MEV Protection

- EMA oracle prevents price manipulation
- Clean separation of path and mechanism prevents gaming
- Systematic fee burning creates deflation
- Atomic execution ensures completeness

### 9.3 Liquidity Management

- TVL weighting minimizes thin pool impact
- Multiple mechanisms per hop ensure availability
- Native-only anchor simplifies liquidity flow
- UTBC unidirectional design prevents liquidity drain

---

## 10. Testing Requirements

### Core Functionality

- Path discovery and mechanism selection
- TVL-weighted EMA price calculation with half-life
- Atomic execution and slippage protection
- Fee collection and burning
- UTBC minting integration

### Edge Cases

- Zero liquidity pools
- Overflow protection in calculations
- Path cycles prevention
- Half-life boundary conditions
- Foreign vs Native token distinction

### Integration

- Compatibility with pallet-asset-conversion
- UTBC pallet unidirectional interface
- FeeManager flow
- Path-mechanism separation logic

---

## Conclusion

Axial Router 1.0 embodies radical simplicity through clean abstraction layers. By separating path discovery from mechanism selection, it provides flexible routing infrastructure that seamlessly integrates both XYK pools and unidirectional UTBC curves.

The architecture leverages battle-tested `pallet-asset-conversion` for bidirectional swaps while properly handling UTBC's unidirectional nature for Foreign→Native minting. TVL-weighted EMA oracles with half-life decay protect against manipulation, while pure economic selection ensures users always get the best price.

Every design decision serves a purpose: path abstraction enables extensibility, mechanism selection ensures optimality, and systematic fee burning creates sustainable deflation. The result is infrastructure so efficient it becomes invisible—users simply enjoy seamless swaps while the protocol intelligently routes through the most advantageous mechanisms.

---

- **Version**: 1.0.0
- **Date**: October 2025
- **Author**: LLB Lab
- **License**: MIT
