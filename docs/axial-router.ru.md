# Спецификация Axial Router 1.0

## Аннотация

Axial Router — это минималистичная система маршрутизации для множественных токенов, оптимизированная для экосистем UTBC. Работая исключительно с внутренними пулами ликвидности парачейна, она обеспечивает интеллектуальный поиск путей и выбор механизмов через Native токен как единственный маршрутный якорь. Роутер бесшовно интегрирует `pallet-asset-conversion` для XYK пулов и специализированный UTBC паллет для однонаправленных кривых связывания. Ключевые возможности включают разделение пути от механизма обмена, защиту цен через TVL-взвешенные EMA с полупериодом распада и систематическое сжигание комиссий для дефляции.

---

## 1. Принципы проектирования

### 1.1 Основная философия

1. **Радикальная простота** — Минимально жизнеспособная маршрутизация без излишеств
2. **Чистые абстракции** — Поиск пути отделён от выбора механизма
3. **Экономическая чистота** — Лучшая цена побеждает, никаких искусственных предпочтений
4. **Только Native якорь** — Native токен как единственный узел маршрутизации
5. **Необходимая защита** — TVL-взвешенные EMA с полупериодом распада предотвращают манипуляции
6. **Атомарное исполнение** — Все операции завершаются в одной транзакции

### 1.2 Системная архитектура

```
┌──────────────────────────────────────┐
│           Axial Router 1.0           │
├──────────────────────────────────────┤
│  ┌────────────────────────────────┐  │
│  │      Движок поиска путей       │  │
│  ├────────────────────────────────┤  │
│  │    Слой выбора механизмов      │  │
│  ├────────────────────────────────┤  │
│  │ TVL-взвешенные EMA с half-life │  │
│  ├────────────────────────────────┤  │
│  │   Менеджер сжигания комиссий   │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼                       ▼
   [XYK Пулы]             [UTBC Кривые]
(pallet-asset-conversion) (pallet-utbc)
```

### 1.3 Источники ликвидности

Роутер интегрирует **2 типа источников ликвидности**:

1. **XYK пулы** — Классические x×y=k AMM пулы через `pallet-asset-conversion`
   - Двунаправленные обмены между любой парой токенов
   - Стандартная торговая комиссия 0.3%

2. **UTBC кривые** — Однонаправленные кривые связывания через специализированный `pallet-utbc`
   - Только минтинг Foreign → Native
   - Нельзя продать Native обратно в кривую
   - Передача резервного актива в POL

---

## 2. Структуры данных

```rust
/// Константа точности для всех ценовых расчётов
const PRECISION: u128 = 1_000_000_000_000;  // 10^12
```

### 2.1 Основная конфигурация

```rust
pub struct RouterConfig {
    /// Native токен как единственный якорь маршрутизации
    pub native_asset: AssetId,

    /// Максимально допустимое отклонение цены (в базисных пунктах)
    pub max_deviation_bps: u32,  // По умолчанию: 2000 (20%)

    /// Комиссия роутера для сжигания (в базисных пунктах)
    pub router_fee_bps: u32,  // По умолчанию: 30 (0.3%)

    /// Полупериод для затухания веса EMA (в блоках)
    pub ema_half_life: BlockNumber,  // По умолчанию: 100 блоков
}
```

### 2.2 Разделение пути и механизма

```rust
/// Полный маршрут от источника к назначению
pub struct Route {
    /// Путь по токенам
    pub path: Vec<AssetId>,

    /// Конкретные переходы с выбранными механизмами
    pub hops: Vec<Hop>,

    /// Общий ожидаемый выход
    pub expected_output: Balance,
}

/// Один переход в маршруте
pub struct Hop {
    /// Исходный токен
    pub from: AssetId,

    /// Целевой токен
    pub to: AssetId,

    /// Механизм обмена для использования
    pub mechanism: SwapMechanism,

    /// Ожидаемый выход для этого перехода
    pub expected_output: Balance,
}

/// Доступные механизмы обмена
pub enum SwapMechanism {
    /// Стандартный обмен через XYK пул
    XykPool {
        pool_id: PoolId,
    },

    /// UTBC минтинг (только Foreign → Native)
    UtbcMint,
}
```

### 2.3 Ценовой оракул

```rust
/// Отслеживание EMA по токенам с затуханием по полупериоду
pub struct TokenEma {
    /// EMA цена токена в нативных единицах
    pub native_price: Balance,

    /// Общая заблокированная стоимость по всем пулам (в нативных единицах)
    pub total_tvl: Balance,

    /// Номер блока последнего обновления
    pub last_update: BlockNumber,
}
```

### 2.4 Интерфейсы

```rust
/// Интерфейс для pallet-asset-conversion XYK пулов
pub trait AssetConversionApi<AccountId, AssetId, Balance> {
    /// Получить ID пула для пары токенов
    fn get_pool_id(asset_a: AssetId, asset_b: AssetId) -> Option<PoolId>;

    /// Котировка цены для обмена с точным входом
    fn quote_price_exact_tokens_for_tokens(
        asset_in: AssetId,
        asset_out: AssetId,
        amount_in: Balance,
        include_fee: bool,
    ) -> Option<Balance>;

    /// Выполнить обмен с точной входной суммой
    fn swap_exact_tokens_for_tokens(
        who: AccountId,
        path: Vec<AssetId>,
        amount_in: Balance,
        min_amount_out: Balance,
        recipient: AccountId,
        keep_alive: bool,
    ) -> DispatchResult;
}

/// Интерфейс для специализированного UTBC паллета (однонаправленный)
pub trait UtbcInterface<AccountId, AssetId, Balance> {
    /// Проверить наличие кривой связывания для токена
    fn has_curve(asset_id: AssetId) -> bool;

    /// Рассчитать, сколько Native получит пользователь за Foreign платёж
    fn calculate_user_receives(
        foreign_asset: AssetId,
        foreign_amount: Balance
    ) -> Result<Balance, DispatchError>;

    /// Выполнить минтинг через кривую связывания
    fn mint_with_distribution(
        who: &AccountId,
        foreign_asset: AssetId,
        foreign_amount: Balance,
    ) -> Result<Balance, DispatchError>;
}

/// Интерфейс для управления комиссиями
pub trait FeeManagerInterface<AccountId, AssetId, Balance> {
    /// Получить и обработать комиссию в указанном активе
    fn receive_fee(asset_id: AssetId, amount: Balance) -> DispatchResult;

    /// Запросить общую сожжённую сумму
    fn total_burned() -> Balance;
}
```

### 2.5 Управление комиссиями

```rust
/// Обработка сбора и сжигания комиссий роутера для дефляционного давления
pub struct FeeManager;

impl FeeManager {
    /// Минимальная сумма для конвертации и сжигания
    const MIN_BURN_AMOUNT: Balance = 1_000;

    /// Получить комиссию в любом активе
    pub fn receive_fee(
        asset_id: AssetId,
        amount: Balance,
    ) -> DispatchResult {
        // Добавить в буфер комиссий для этого актива
        FeeBuffers::<T>::mutate(asset_id, |buffer| {
            *buffer = buffer.saturating_add(amount);
        });

        // Попытаться обработать накопленные комиссии
        Self::process_fee_buffer(asset_id)
    }

    /// Обработать накопленные комиссии для сжигания
    fn process_fee_buffer(asset_id: AssetId) -> DispatchResult {
        let buffer_amount = FeeBuffers::<T>::get(asset_id);

        if buffer_amount < Self::MIN_BURN_AMOUNT {
            return Ok(());  // Ждать накопления большего объёма
        }

        // Если нативный актив, сжечь напрямую
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
            // Сначала конвертировать в нативный через XYK пул
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
                    // Оставить в буфере при неудачном обмене
                    return Ok(());
                }
            }
        }

        Ok(())
    }

    /// Запросить общую сожжённую сумму
    pub fn total_burned() -> Balance {
        TotalBurned::<T>::get()
    }
}
```

---

## 3. Поиск маршрутов

### 3.1 Движок поиска путей

Роутер сначала находит возможные пути, затем выбирает оптимальные механизмы для каждого перехода:

```rust
impl<T: Config> Pallet<T> {
    pub fn find_best_route(
        from: AssetId,
        to: AssetId,
        amount: Balance,
    ) -> Result<Route, Error> {
        let native = T::NativeAsset::get();

        // Шаг 1: Найти возможные пути
        let paths = vec![
            vec![from, to],                // Прямой путь
            vec![from, native, to],        // Через Native
        ];

        // Шаг 2: Построить маршруты с оптимальными механизмами
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

    /// Построить маршрут, выбирая лучший механизм для каждого перехода
    fn build_route_with_mechanisms(
        path: &[AssetId],
        initial_amount: Balance,
    ) -> Result<Route, Error> {
        let mut hops = Vec::new();
        let mut current_amount = initial_amount;

        // Для каждой последовательной пары в пути
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

    /// Найти лучший механизм для одного перехода
    fn find_best_hop(
        from: AssetId,
        to: AssetId,
        amount: Balance,
    ) -> Result<Hop, Error> {
        let mut best_mechanism = None;
        let mut best_output = 0;

        // Вариант 1: XYK пул
        if let Some(pool_id) = T::AssetConversion::get_pool_id(from, to) {
            if let Some(output) = T::AssetConversion::quote_price_exact_tokens_for_tokens(
                from, to, amount, true
            ) {
                best_mechanism = Some(SwapMechanism::XykPool { pool_id });
                best_output = output;
            }
        }

        // Вариант 2: UTBC минтинг (только для Foreign → Native)
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

    /// Проверить, является ли актив внешним токеном (не Native)
    fn is_foreign(asset: AssetId) -> bool {
        asset != T::NativeAsset::get()
    }
}
```

### 3.2 Критерии выбора механизма

Для каждого перехода роутер сравнивает все доступные механизмы и выбирает тот, который даёт наибольший выход. Никакие искусственные предпочтения или смещения не применяются - чистая экономическая эффективность определяет выбор.

---

## 4. Ценовой оракул

### 4.1 Реализация EMA с полупериодом распада

Роутер поддерживает экспоненциально взвешенные скользящие средние цены для каждого токена, используя TVL-взвешенное ценообразование по всем пулам с затуханием по полупериоду для плавной деградации доверия:

```rust
impl<T: Config> Pallet<T> {
    /// Рассчитать вес EMA на основе возраста с затуханием по полупериоду
    fn calculate_ema_weight(age_blocks: BlockNumber) -> u128 {
        let half_life = T::EmaHalfLife::get();

        if age_blocks == 0 {
            return PRECISION;
        }

        // Вес = 2^(-возраст/полупериод) * PRECISION
        // Для вычислительной эффективности аппроксимируем линейными сегментами
        let decay_periods = age_blocks / half_life;
        let remainder = age_blocks % half_life;

        // Каждый период уменьшает вес вдвое
        let base_weight = PRECISION >> decay_periods.min(40);  // Ограничение на 2^-40

        // Линейная интерполяция для частичного периода
        let partial_decay = remainder.saturating_mul(base_weight) / (half_life * 2);
        base_weight.saturating_sub(partial_decay)
    }

    /// Обновить EMA токена с TVL-взвешенным ценообразованием
    pub fn update_token_ema(
        token_id: AssetId,
    ) -> DispatchResult {
        // Пропустить нативный токен (всегда 1:1 сам с собой)
        if token_id == T::NativeAsset::get() {
            return Ok(());
        }

        // Рассчитать TVL-взвешенную среднюю цену по всем пулам
        let (weighted_price, total_tvl) = Self::calculate_tvl_weighted_price(token_id)?;

        if total_tvl == 0 {
            return Ok(());  // Нет ликвидности, пропустить обновление
        }

        TokenEmas::<T>::mutate(token_id, |maybe_ema| {
            match maybe_ema {
                Some(ema) => {
                    let age = frame_system::Pallet::<T>::block_number()
                        .saturating_sub(ema.last_update);

                    // Рассчитать вес на основе возраста (затухание по полупериоду)
                    let old_weight = Self::calculate_ema_weight(age);
                    let new_weight = PRECISION;
                    let total_weight = old_weight.saturating_add(new_weight);

                    // Взвешенное среднее: (старая_цена * старый_вес + новая_цена * новый_вес) / общий_вес
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

    /// Рассчитать TVL-взвешенную среднюю цену для токена
    fn calculate_tvl_weighted_price(
        token_id: AssetId,
    ) -> Result<(Balance, Balance), DispatchError> {
        let mut weighted_sum = 0u128;
        let mut total_tvl = 0u128;

        // Итерация по всем пулам, содержащим этот токен
        for pool_id in T::Pools::get_pools_for_token(token_id) {
            let (reserve_token, reserve_native) = if Self::is_native_pair(pool_id, token_id) {
                // Прямая пара с нативным
                T::Pools::get_reserves(pool_id)?
            } else {
                // Непрямая пара - нужно рассчитать нативную стоимость
                Self::calculate_native_value_in_pool(pool_id, token_id)?
            };

            if reserve_token > 0 {
                // Цена = native_reserve / token_reserve
                let price = reserve_native
                    .saturating_mul(PRECISION)
                    .saturating_div(reserve_token);

                // Вес по TVL (2 * native_reserve для XYK пулов)
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

    /// Валидация допустимого отклонения цен токенов
    pub fn validate_price_deviation(
        route: &Route,
    ) -> bool {
        let config = RouterConfiguration::<T>::get();

        // Рассчитать подразумеваемый курс обмена из маршрута
        let from_amount = route.hops.first().map(|h| h.expected_output).unwrap_or(0);
        let to_amount = route.expected_output;

        if from_amount == 0 {
            return false;
        }

        let implied_rate = to_amount
            .saturating_mul(PRECISION)
            .saturating_div(from_amount);

        // Получить EMA цены для токенов
        let from_token = route.path.first().unwrap();
        let to_token = route.path.last().unwrap();

        let from_ema = Self::get_token_ema_price(*from_token);
        let to_ema = Self::get_token_ema_price(*to_token);

        // Рассчитать ожидаемый курс из EMA
        let expected_rate = from_ema
            .saturating_mul(PRECISION)
            .saturating_div(to_ema.max(1));

        // Проверить отклонение
        let max_deviation = config.max_deviation_bps as u128;
        let lower_bound = expected_rate
            .saturating_mul(10000u128.saturating_sub(max_deviation))
            .saturating_div(10000);
        let upper_bound = expected_rate
            .saturating_mul(10000u128.saturating_add(max_deviation))
            .saturating_div(10000);

        implied_rate >= lower_bound && implied_rate <= upper_bound
    }

    /// Получить EMA цену токена с взвешиванием по полупериоду
    fn get_token_ema_price(token_id: AssetId) -> Balance {
        // Нативный токен всегда 1:1 сам с собой
        if token_id == T::NativeAsset::get() {
            return PRECISION;
        }

        if let Some(ema) = TokenEmas::<T>::get(token_id) {
            let current_block = frame_system::Pallet::<T>::block_number();
            let age = current_block.saturating_sub(ema.last_update);

            // Получить вес на основе возраста (затухание по полупериоду)
            let ema_weight = Self::calculate_ema_weight(age);

            // Если вес всё ещё значителен (>12.5%), использовать EMA
            if ema_weight > PRECISION / 8 {
                return ema.native_price;
            }
        }

        // Откат к текущей TVL-взвешенной цене
        Self::calculate_tvl_weighted_price(token_id)
            .map(|(price, _)| price)
            .unwrap_or(PRECISION)
    }

    /// Проверить, является ли пул прямой нативной парой
    fn is_native_pair(pool_id: PoolId, token_id: AssetId) -> bool {
        let (token_a, token_b) = T::Pools::get_pool_assets(pool_id)
            .unwrap_or((AssetId::default(), AssetId::default()));

        let native = T::NativeAsset::get();
        (token_a == native && token_b == token_id) ||
        (token_a == token_id && token_b == native)
    }

    /// Рассчитать нативную стоимость для не-нативных пар через маршрутизацию
    fn calculate_native_value_in_pool(
        pool_id: PoolId,
        token_id: AssetId,
    ) -> Result<(Balance, Balance), DispatchError> {
        let (token_a, token_b) = T::Pools::get_pool_assets(pool_id)?;
        let (reserve_a, reserve_b) = T::Pools::get_reserves(pool_id)?;

        // Определить, какой резерв соответствует нашему токену
        let (token_reserve, other_token, other_reserve) = if token_a == token_id {
            (reserve_a, token_b, reserve_b)
        } else {
            (reserve_b, token_a, reserve_a)
        };

        // Получить нативную стоимость другого токена
        let other_native_price = Self::get_token_ema_price(other_token);

        // Рассчитать нативную стоимость другой стороны пула
        let native_value = other_reserve
            .saturating_mul(other_native_price)
            .saturating_div(PRECISION);

        Ok((token_reserve, native_value))
    }
}
```

---

## 5. Исполнение обменов

### 5.1 Стандартный обмен

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

        // Валидации
        ensure!(!Self::is_paused(), Error::<T>::RouterPaused);
        ensure!(from != to, Error::<T>::IdenticalAssets);
        ensure!(amount_in > 0, Error::<T>::ZeroAmount);
        ensure!(
            frame_system::Pallet::<T>::block_number() <= deadline,
            Error::<T>::DeadlinePassed
        );

        // Сбор комиссии роутера
        let router_fee = T::RouterFeeBps::get()
            .saturating_mul(amount_in)
            .saturating_div(10000);
        let amount_after_fee = amount_in.saturating_sub(router_fee);

        T::FeeManager::receive_fee(from, router_fee)?;

        // Найти лучший маршрут (путь + механизмы)
        let route = Self::find_best_route(from, to, amount_after_fee)?;

        // Валидировать цену перед исполнением
        ensure!(
            Self::validate_price_deviation(&route),
            Error::<T>::ExcessivePriceDeviation
        );

        // Выполнить и валидировать проскальзывание
        let amount_out = Self::execute_route(&who, &route)?;
        ensure!(amount_out >= min_amount_out, Error::<T>::SlippageExceeded);

        // Обновить EMA токенов
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

### 5.2 Исполнение маршрута

```rust
impl<T: Config> Pallet<T> {
    fn execute_route(
        who: &T::AccountId,
        route: &Route,
    ) -> Result<Balance, DispatchError> {
        let mut current_balance = T::Assets::balance(route.path[0], who);

        // Исполнить каждый переход
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
                    1,  // min_amount_out уже валидирован
                    who.clone(),
                    false,  // keep_alive
                )?;
                Ok(T::Assets::balance(hop.to, who))
            },

            SwapMechanism::UtbcMint => {
                // Действителен только для Foreign → Native
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

    /// Котировка ожидаемого выхода для маршрута без исполнения
    fn quote_route(route: &Route) -> Balance {
        route.expected_output
    }
}
```

---

## 6. Хранилище

```rust
/// Отслеживание EMA токенов с TVL-взвешенным ценообразованием
#[pallet::storage]
pub type TokenEmas<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    AssetId,
    TokenEma,
    OptionQuery,
>;

/// Буферы комиссий, ожидающие конвертации/сжигания
#[pallet::storage]
pub type FeeBuffers<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    AssetId,
    Balance,
    ValueQuery,
>;

/// Общее количество сожжённых нативных токенов
#[pallet::storage]
pub type TotalBurned<T: Config> = StorageValue<_, Balance, ValueQuery>;

/// Конфигурация роутера
#[pallet::storage]
pub type RouterConfiguration<T: Config> = StorageValue<_, RouterConfig, ValueQuery>;
```

---

## 7. События и ошибки

### 7.1 События

```rust
/// События, генерируемые роутером
#[pallet::event]
pub enum Event<T: Config = ()> {
    /// Обмен успешно выполнен
    SwapExecuted {
        who: T::AccountId,
        from: AssetId,
        to: AssetId,
        amount_in: Balance,
        amount_out: Balance,
        route: Route,
    },

    /// EMA токена обновлена
    TokenEmaUpdated {
        token: AssetId,
        native_price: Balance,
        tvl: Balance,
    },

    /// Комиссия сожжена
    FeeBurned {
        asset: AssetId,
        amount: Balance,
    },
}
```

### 7.2 Ошибки

```rust
#[pallet::error]
pub enum Error<T> {
    /// Жизнеспособный маршрут между токенами не найден
    NoRouteFound,

    /// Одинаковые исходный и целевой активы
    IdenticalAssets,

    /// Нулевая сумма
    ZeroAmount,

    /// Недостаточная ликвидность в пулах
    InsufficientLiquidity,

    /// Выходная сумма ниже минимально допустимой
    SlippageExceeded,

    /// Дедлайн транзакции истёк
    DeadlinePassed,

    /// Цена слишком сильно отклоняется от EMA
    ExcessivePriceDeviation,

    /// Недопустимый механизм для данной пары токенов
    InvalidMechanism,

    /// Арифметическое переполнение
    Overflow,

    /// Роутер приостановлен
    RouterPaused,
}
```

---

## 8. Конфигурация

### 8.1 Конфигурация паллета

```rust
#[pallet::config]
pub trait Config: frame_system::Config {
    /// События рантайма
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;

    /// Интерфейс управления активами
    type Assets: Inspect<Self::AccountId, AssetId = AssetId, Balance = Balance> +
                 Transfer<Self::AccountId> +
                 Burn<Self::AccountId>;

    /// Пулы конвертации активов (pallet-asset-conversion)
    type AssetConversion: AssetConversionApi<
        Self::AccountId,
        AssetId,
        Balance,
    >;

    /// UTBC паллет для кривых связывания
    type UtbcPallet: UtbcInterface<Self::AccountId, AssetId, Balance>;

    /// ID нативного актива
    #[pallet::constant]
    type NativeAsset: Get<AssetId>;

    /// Максимальное отклонение цены в базисных пунктах (по умолчанию: 2000 = 20%)
    #[pallet::constant]
    type MaxDeviationBps: Get<u32>;

    /// Комиссия роутера в базисных пунктах (по умолчанию: 30 = 0.3%)
    #[pallet::constant]
    type RouterFeeBps: Get<u32>;

    /// Полупериод для затухания веса EMA (по умолчанию: 100 блоков)
    #[pallet::constant]
    type EmaHalfLife: Get<Self::BlockNumber>;

    /// Менеджер комиссий для сжигания комиссий роутера
    type FeeManager: FeeManagerInterface<Self::AccountId, AssetId, Balance>;

    /// Информация о весах
    type WeightInfo: WeightInfo;
}
```

### 8.2 Конфигурация генезиса

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
        // Инициализация конфигурации роутера
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

## 9. Соображения безопасности

### 9.1 Манипуляция ценами

- EMA по токенам с TVL-взвешенным ценообразованием
- Глобальный лимит отклонения (20% по умолчанию)
- Затухание по полупериоду для плавной деградации доверия
- Валидация перед исполнением предотвращает сэндвич-атаки
- TVL-взвешивание приоритезирует крупные, стабильные пулы

### 9.2 Защита от MEV

- EMA оракул предотвращает манипуляции ценами
- Чистое разделение пути и механизма предотвращает игры
- Систематическое сжигание комиссий создаёт дефляцию
- Атомарное исполнение гарантирует завершённость

### 9.3 Управление ликвидностью

- TVL-взвешивание минимизирует влияние тонких пулов
- Множественные механизмы на переход обеспечивают доступность
- Якорь только через Native упрощает поток ликвидности
- Однонаправленный дизайн UTBC предотвращает утечку ликвидности

---

## 10. Требования к тестированию

### Основная функциональность

- Поиск путей и выбор механизмов
- TVL-взвешенный расчёт EMA цен с полупериодом
- Атомарное исполнение и защита от проскальзывания
- Сбор и сжигание комиссий
- Интеграция UTBC минтинга

### Граничные случаи

- Пулы с нулевой ликвидностью
- Защита от переполнения в вычислениях
- Предотвращение циклов в путях
- Граничные условия полупериода
- Различие между Foreign и Native токенами

### Интеграция

- Совместимость с pallet-asset-conversion
- Однонаправленный интерфейс UTBC паллета
- Поток FeeManager
- Логика разделения путь-механизм

---

## Заключение

Axial Router 1.0 воплощает радикальную простоту через чистые слои абстракции. Разделяя поиск путей от выбора механизмов, он предоставляет гибкую инфраструктуру маршрутизации, которая бесшовно интегрирует как XYK пулы, так и однонаправленные UTBC кривые.

Архитектура использует проверенный в боях `pallet-asset-conversion` для двунаправленных обменов, правильно обрабатывая однонаправленную природу UTBC для минтинга Foreign→Native. TVL-взвешенные EMA оракулы с затуханием по полупериоду защищают от манипуляций, а чисто экономический выбор гарантирует, что пользователи всегда получают лучшую цену.

Каждое проектное решение служит цели: абстракция пути обеспечивает расширяемость, выбор механизма гарантирует оптимальность, а систематическое сжигание комиссий создаёт устойчивую дефляцию. Результат — инфраструктура настолько эффективная, что становится невидимой: пользователи просто наслаждаются бесшовными обменами, пока протокол интеллектуально маршрутизирует через наиболее выгодные механизмы.

---

- **Версия**: 1.0.0
- **Дата**: Октябрь 2025
- **Автор**: LLB Lab
- **Лицензия**: MIT
