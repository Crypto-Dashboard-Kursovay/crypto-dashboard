# Phase 9 — Trading Start Reliability, Backtest Resume, Fast Balance Refresh Plan

## Цель

1. Убрать зависание стратегий в статусе `Запускается`, когда Redis Pub/Sub-команда
   потерялась или движок не был подписан в момент публикации.
2. Сохранить активный бэктест при переходе на другие страницы SPA и восстановить
   polling при возврате.
3. Ускорить обновление баланса без агрессивного спама приватных API бирж.

## План реализации

- **Движок**
  - Оставить Redis Pub/Sub быстрым каналом команд.
  - Добавить чтение pending `bot_commands` из PostgreSQL как durable source of truth.
  - Обрабатывать backlog примерно раз в 1 секунду, с Redis-dedup по `command_id`.
  - Помечать `processed_at` после попытки обработки, включая команды, уже обработанные
    через Pub/Sub.
  - Ограничить backlog свежими и статусно-релевантными командами, чтобы после деплоя не
    переиграть исторические строки, у которых раньше никогда не заполнялся `processed_at`.

- **Backend**
  - По engine log `kind=strategy_started` переводить бота в `RUNNING` сразу, не дожидаясь
    heartbeat.
  - Убрать дублирующийся блок heartbeat-проекции `RUNNING/STOPPING -> STOPPED`.

- **Frontend**
  - Сохранять активный `backtest job id` в `localStorage`.
  - При входе на страницу бэктеста восстанавливать сохранённый job или последний
    `queued/running` job из `listBacktests`.
  - Очищать active-key после `completed/failed`, но оставлять результат видимым.
  - Снизить polling `BalanceWidget` до 5 секунд.
  - Дополнительно обновлять баланс по focus/visibilitychange и по WS-сигналу
    `balance_update` из `LogsContext`.

- **Конфигурация**
  - `ENGINE_BALANCE_POLL_INTERVAL_SEC` по умолчанию: `5`.
  - `ENGINE_COMMAND_POLL_INTERVAL_SEC` по умолчанию: `1`.

## Проверка

- `cd trade-engine-crypto && ./.venv/bin/python -m pytest -q`
- `cd backend && uv run pytest -q`
- `cd frontend && pnpm test -- --run Backtesting BalanceWidget`
- `cd frontend && pnpm build`

## Предположения

- Надёжный запуск стратегий означает доставку команды в движок и переход стратегии в
  running; фактическая торговля всё ещё зависит от валидных testnet-ключей, сети биржи,
  доступности символа и появления торгового сигнала.
- 5 секунд для баланса — самый быстрый безопасный default: меньший интервал повышает риск
  rate-limit по приватным endpoint бирж.
- Восстановление бэктеста покрывает навигацию внутри SPA; перезапуск backend worker остаётся
  отдельной задачей.
