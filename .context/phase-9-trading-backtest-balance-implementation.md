# Phase 9 — Trading Start Reliability, Backtest Resume, Fast Balance Refresh

## Что сделано

### 1. Стратегии больше не зависят только от Redis Pub/Sub

- В engine добавлена mirror-модель `bot_commands` и репозиторий pending-команд.
- `CommandListener` теперь:
  - продолжает слушать `engine.commands.start|stop|update` через Redis Pub/Sub;
  - примерно раз в секунду читает свежие pending-команды из PostgreSQL;
  - использует Redis-dedup `engine:commands:processed:{command_id}` для идемпотентности;
  - помечает `processed_at` после обработки или после обнаружения дубля.
- Backlog ограничен свежими и статусно-релевантными строками:
  - `start`: bot `starting/running`;
  - `stop`: bot `stopping/stopped`;
  - `update`: bot `starting/running`.
- Это защищает прод от replay старых команд, потому что до этой фазы движок вообще не
  заполнял `bot_commands.processed_at`.

### 2. Backend быстрее подтверждает запуск

- `EventProjector` теперь по `engine.log` с `kind=strategy_started` сразу переводит бота в
  `RUNNING`.
- Дублирующийся блок heartbeat-проекции `RUNNING/STOPPING -> STOPPED` удалён.

### 3. Бэктест переживает переходы между страницами

- `Backtesting.tsx` сохраняет активный job id в `localStorage`:
  `crypto.backtest.activeJobId.v1`.
- При входе на страницу восстанавливается сохранённый job, а если ключа нет — последний
  `queued/running` job из `listBacktests(10)`.
- Polling продолжается после возврата на страницу.
- После `completed/failed` active-key очищается, но результат остаётся на экране.

### 4. Баланс обновляется быстрее

- Default `ENGINE_BALANCE_POLL_INTERVAL_SEC` изменён с `15` на `5`.
- В `docker-compose.yml` и `docker-compose.dev.yml` добавлены defaults:
  - `ENGINE_BALANCE_POLL_INTERVAL_SEC=5`;
  - `ENGINE_COMMAND_POLL_INTERVAL_SEC=1`.
- `BalanceWidget` теперь обновляется:
  - при монтировании;
  - каждые 5 секунд;
  - при `window focus`;
  - при `visibilitychange` обратно в visible;
  - сразу после WS-сигнала `balance_update` из `LogsContext`.
- `LogsContext` экспортирует optional-сигнал `lastBalanceUpdateAt`, не ломая старый
  `useLogs()` contract.

## Основные файлы

- Engine:
  - `trade-engine-crypto/src/infrastructure/command_listener.py`
  - `trade-engine-crypto/src/infrastructure/db_models.py`
  - `trade-engine-crypto/src/infrastructure/db_repositories.py`
  - `trade-engine-crypto/src/engine_main.py`
  - `trade-engine-crypto/src/infrastructure/settings.py`
- Backend:
  - `backend/src/services/event_projector.py`
- Frontend:
  - `frontend/src/app/pages/Backtesting.tsx`
  - `frontend/src/app/components/dashboard/BalanceWidget.tsx`
  - `frontend/src/app/LogsContext.tsx`
- Infra/docs:
  - `docker-compose.yml`
  - `docker-compose.dev.yml`
  - `AGENTS.md`

## Тесты

Пройдено:

- `cd trade-engine-crypto && ./.venv/bin/python -m pytest -q`
  - `163 passed`
  - первый sandbox-прогон упал на metrics-тестах из-за запрета bind к `127.0.0.1:0`;
    повторный прогон с разрешением локального bind прошёл.
- `cd backend && uv --cache-dir ../.uv-cache run pytest tests/unit -q`
  - `31 passed`
- `cd backend && uv --cache-dir ../.uv-cache run python -m compileall src tests`
  - ok
- `cd frontend && pnpm test -- --run Backtesting BalanceWidget`
  - `13 passed`
- `cd frontend && pnpm test`
  - `104 passed`
- `cd frontend && pnpm build`
  - ok, остался только стандартный Vite warning про chunk >500 kB.

Ограничение проверки:

- `cd backend && uv --cache-dir ../.uv-cache run pytest -q` не проходит на локальной машине,
  потому что integration fixtures подключаются к `postgresql://test:test@localhost/test_db`,
  а локальная роль Postgres `test` отсутствует:
  `asyncpg.exceptions.InvalidAuthorizationSpecificationError: role "test" does not exist`.
  Unit-тесты backend и компиляция проходят.

## Как проверить вручную

1. Поднять backend, engine, Redis и Postgres.
2. Создать/подключить testnet credential.
3. Запустить стратегию из UI.
4. Ожидаемое поведение:
   - бот быстро переходит из `Запускается` в `Активна`;
   - если Redis-команда была пропущена, engine подхватит pending command из `bot_commands`;
   - `bot_commands.processed_at` заполнится после обработки.
5. Запустить бэктест, перейти на другую страницу, вернуться в бэктестинг.
   - Ожидаемо: активный job восстановлен, индикатор/результат продолжаются.
6. Открыть dashboard.
   - Ожидаемо: баланс обновляется каждые 5 секунд и дополнительно сразу после WS
     `balance_update`.
