# Phase 8 — Совокупный баланс по биржам

## Цель

Главная страница должна показывать не баланс одного running-бота, а совокупный баланс
пользователя по всем подключённым биржам. Старый путь данных был неполным: engine
публиковал `engine.balance_update` только для активных стратегий, а backend складывал
сырые валютные количества (`USDT + BTC`) без пересчёта в USDT.

## Архитектура изменений

- **Engine** теперь опрашивает все сохранённые `exchange_credentials` независимо от
  running-ботов. `StateManager` получает `CredentialRepository` и фабрику exchange-adapter,
  держит отдельный кеш adapter'ов для баланса и закрывает adapter при удалении credential.
- **Позиции не смешаны с балансом:** `positions_update` остался привязан к running-ботам,
  потому что текущий `PositionManager` и exchange positions всё ещё связаны с live-стратегиями.
- **Binance Spot Testnet:** `CCXTExchangeAdapter.get_balance()` для Binance использует
  raw `privateGetAccount` (`/api/v3/account`) и парсит `balances[].free/locked`. Это обходит
  `fetch_balance()`, который в ccxt вызывает `load_markets()` и может зависать/падать на
  Binance testnet.
- **Backend summary:** `/api/balances/summary` оставляет `currencies` в исходных валютах, но
  верхние `total_equity`, `free_total`, `used_total` считает как USDT-стоимость. `USDT`
  оценивается как `1`, разрешённые базовые активы (`BTC/SOL/XRP/BNB/ETH`) оцениваются через
  публичный `fetch_ticker`, результат кешируется в Redis на 60 секунд.
- **Latest-снапшоты:** `BalanceRepository.latest_for_credential()` теперь делает latest row
  per `(credential_id, currency)` через SQL window function.

## Тронутые файлы

- `trade-engine-crypto/src/infrastructure/state_manager.py`
- `trade-engine-crypto/src/infrastructure/db_repositories.py`
- `trade-engine-crypto/src/infrastructure/ccxt_exchange_adapter.py`
- `trade-engine-crypto/src/engine_main.py`
- `backend/src/api/routers/balances.py`
- `backend/src/repositories/balance_repo.py`
- `backend/src/services/balance_valuation.py`
- `trade-engine-crypto/tests/infrastructure/test_state_manager.py`
- `trade-engine-crypto/tests/infrastructure/test_ccxt_exchange_adapter.py`
- `backend/tests/integration/test_balances_summary.py`

## Как тестировать

```bash
cd trade-engine-crypto
./.venv/bin/python -m pytest tests/infrastructure/test_ccxt_exchange_adapter.py tests/infrastructure/test_state_manager.py -q

cd ../backend
uv --cache-dir /private/tmp/uv-cache run pytest tests/integration/test_balances_summary.py -q

cd ../frontend
pnpm test -- --run BalanceWidget
pnpm build
```

Backend integration-тестам нужен настоящий PostgreSQL с ролью/БД из `INTEGRATION_DB_URL`
(`postgresql+asyncpg://test:test@localhost:5432/test_db` по умолчанию) и Redis.

## Проверка поведения вручную

1. Запустить PostgreSQL, Redis, backend и engine.
2. Добавить Binance Testnet credential в Settings.
3. Не запускать бота: через один balance interval engine должен опубликовать
   `engine.balance_update`, а `/api/balances/summary` должен вернуть ненулевые `currencies`.
4. Добавить второй credential другой биржи: dashboard должен показать сумму USDT-стоимости
   обеих бирж.
5. Проверить `/api/engine-health`: `last_balance_success.credentials_polled` должен быть
   равен числу подключённых credentials.
