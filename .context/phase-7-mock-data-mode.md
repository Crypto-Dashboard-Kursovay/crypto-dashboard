# Phase 7 — Демо-режим «мок-данные»

> Только frontend-submodule. Цель: показывать UI «вживую» без поднятого бэкенда, движка и
> реальных бирж (демо/защита курсовой). Тумблер в хедере, включённый по умолчанию.

## TL;DR

- В хедере рядом с почтой — `Switch` **«мок-данные»**, по умолчанию **ON**.
- Когда включён, фронт **сам фабрикует** баланс, открытые позиции, стратегии, сделки, логи,
  подключённые биржи и бэктест. Никаких сетевых запросов — действия пользователя **не**
  дёргают движок/биржи.
- Реализовано **перехватом на уровне `src/api/*.ts`**: `if (isMockEnabled()) return mockStore.*`.
  Виджеты/страницы не изменялись (кроме диалога редактирования стратегий).
- Тесты: **97 зелёных** (88 прежних + 9 новых на mock-стор), `pnpm build` ок.

---

## Архитектура

Каждый виджет/страница вызывает тонкую функцию из `src/api/*.ts` (через `apiFetch`) и
опрашивает данные по таймеру (5–15 с). Мок внедрён **перед** `fetch`:

```
Header [Switch «мок-данные»] → MockDataContext.enabled ⇄ localStorage("crypto.mock.enabled")
                                       │
        api/*.ts:  if (isMockEnabled()) return mockStore.xxx()      ← перехват
                                       │
MockDataProvider.tick() (setInterval 4с): mockStore.tick()
   → случайное блуждание цен → PnL позиций → equity баланса
   → иногда новая сделка (от случайной активной стратегии)
   → массив строк логов → LogsContext.pushLog()
```

Состояние сессии живёт в **singleton-сторе** (`src/mock/store.ts`) — обычный TS-модуль, не
React, чтобы api-функции читали его синхронно из любого места. React-контекст
(`MockDataContext`) отвечает только за тумблер и за «тик».

### Модель сессии (генерится один раз на загрузку страницы)
- **Биржи/ключи:** по одному фиктивному `CredentialOut` на каждую из Binance/Bybit/OKX/MEXC
  ⇒ в Настройках «все биржи подключены».
- **Стратегии:** **5–15** случайных `BotOut` (класс из 7, пара из allowlist
  BTC/SOL/XRP/BNB/ETH-USDT, ТФ, дефолт-параметры), ~78 % в статусе `running`.
- **Позиции:** по одной на каждый `running`-бот (нотионал 250–3500 USDT).
- **Баланс:** `total_equity = freeCash + Σ(entry·size) + Σ PnL`; `open_pnl = Σ PnL`;
  `position_count` = число позиций. PnL колеблется в обе стороны (рандом ±0.4 %/тик).
- **Сделки:** предзаполнены ~20–45 за последние 6 ч, далее прирастают на тике (cap 500).
- **Логи:** тик периодически шлёт INFO (оценка свечи / снапшот баланса / новая сделка) и
  изредка WARNING.
- **Бэктест:** `runBacktest` сразу возвращает `completed`-job со сгенерированными метриками
  и equity-кривой.

---

## Новые файлы

| Файл | Назначение |
|---|---|
| `frontend/src/mock/config.ts` | Флаг `isMockEnabled()`/`setMockEnabled()` (ключ `crypto.mock.enabled`; нет ключа ⇒ ON в приложении, OFF под Vitest), список бирж, allowlist пар + базовые цены, дефолты параметров стратегий. |
| `frontend/src/mock/generators.ts` | Чистые хелперы: `uuid`, `randInt/randFloat`, `pickRandom`, `chance`, `decStr`, `generateCandles` (random walk OHLCV). |
| `frontend/src/mock/store.ts` | Singleton `mockStore`: инициализация сессии, геттеры под все DTO, мутации (create/start/stop/update/delete bot, add/remove credential, backtest), `tick()`. |
| `frontend/src/app/MockDataContext.tsx` | `MockDataProvider` + `useMockData()`: состояние `enabled`, таймер `tick()`, проброс логов в `useLogs().pushLog`. |
| `frontend/src/test/mockStore.test.ts` | 9 тестов: сессия 5–15 ботов, позиции↔активные, баланс, candles, тик, мутации, бэктест. |

## Изменённые файлы

| Файл | Изменение |
|---|---|
| `frontend/src/api/bots.ts` | guard на listBots/getBot/createBot/startBot/stopBot/updateBotParams/deleteBot |
| `frontend/src/api/trades.ts` | guard на listTrades |
| `frontend/src/api/balances.ts` | guard на listBalances/fetchBalanceSummary |
| `frontend/src/api/positions.ts` | guard на listPositions |
| `frontend/src/api/candles.ts` | guard на fetchCandles |
| `frontend/src/api/credentials.ts` | guard на listCredentials/createCredential/deleteCredential |
| `frontend/src/api/exchanges.ts` | guard на listSupportedExchanges/listExchangeSymbols |
| `frontend/src/api/health.ts` | guard на fetchHealth |
| `frontend/src/api/backtest.ts` | guard на runBacktest/getBacktest/listBacktests/deleteBacktest |
| `frontend/src/app/LogsContext.tsx` | + `pushLog` в значении контекста (для инъекции мок-логов) |
| `frontend/src/app/components/Layout.tsx` | `<MockDataProvider>` внутри `<LogsProvider>` |
| `frontend/src/app/components/layout/Header.tsx` | `Switch` «мок-данные» рядом с почтой |
| `frontend/src/app/pages/Strategies.tsx` | кнопка «Редактировать» + диалог изменения параметров (`updateBotParams`) |
| `frontend/src/test/setup.ts` | выставляет `crypto.mock.enabled=false` для тестов |

---

## Важные детали
- **Default ON только в приложении.** Под Vitest `config.ts` смотрит на `process.env.VITEST`
  и по умолчанию даёт OFF (даже если тест очистил localStorage в токен-тестах) — чтобы
  существующие тесты api/виджетов шли реальным путём через мокнутый `fetch`.
- **Per-session.** Стратегии генерятся заново при каждой загрузке (in-memory singleton, не
  персистится). Изменения пользователя живут до перезагрузки.
- **Тумблер** вступает в силу на следующем опросе виджета (≤15 с); reload не делаем.
- **WS** в `LogsContext` в моке может висеть в `closed` (бэка нет) — это ок, мок-логи идут
  через `pushLog`.
- Decimal-поля возвращаются **строками** (как реальные pydantic-схемы).

## Как проверить
1. `cd frontend && pnpm install && pnpm dev` → `http://localhost:5173`, залогиниться.
2. В хедере тумблер «мок-данные» включён. Dashboard: баланс колеблется, позиции = числу
   активных стратегий, график рисуется, сделки прирастают.
3. Стратегии: 5–15 карточек; Start/Stop/Delete; «Редактировать» меняет параметры; «Создать
   стратегию» добавляет фиктивную. Сделки/Логи наполняются. Настройки: все биржи подключены.
   Бэктест: запуск выдаёт результат с метриками и equity-кривой.
4. DevTools → Network: действия в мок-режиме **не** порождают запросов к `/api/*`.
5. `pnpm test` → 97 зелёных. `pnpm build` → ок.
