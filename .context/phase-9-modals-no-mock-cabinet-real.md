# Phase 9 — Унификация модалок + удаление мока + рабочий кабинет

Дата: 2026-05-25
Ветка: `main` (backend submodule + frontend submodule + root)

## Context

Продолжение работы по фазе 8. Три направления:

1. **Модальные окна** — дефолтный MUI Dialog выглядел как простая плашка (квадратные углы, серый, без X). Нужен единый Perplexity-стиль через theme overrides + общая обёртка.
2. **Mock-режим** — бэк живёт в продакшене, mock-инфраструктура (`frontend/src/mock/`, `MockDataContext`, свитч в шапке, ветки `if (isMockEnabled())`) была мёртвым кодом. Удалили полностью.
3. **Личный кабинет** — `POST /api/cabinet/api-keys` отдавал 404 (бэк-эндпоинта нет), псевдо-QR не сканировался. Сделали полный бэк (FastAPI router + Alembic-миграция + bcrypt + TOTP) и настоящий `<QRCodeSVG>` от `qrcode.react`.

Логи продового `crypto-backend` показывали активный трафик по `/api/bots`, `/api/auth/me`, `/api/positions`, и т.д. — никаких запросов на `/api/cabinet/*` не доходило (404 на стороне фронта). Это подтвердило, что эндпоинтов просто нет.

## Что сделано — Backend

### 1. Зависимость pyotp

[backend/pyproject.toml:25](backend/pyproject.toml#L25) — добавил `"pyotp>=2.9"` в `dependencies`. `bcrypt>=4.0.1,<4.1` уже был в зависимостях, отдельно ставить не пришлось.

### 2. Миграция 0007

Новый файл [backend/alembic/versions/0007_user_api_keys_and_2fa.py](backend/alembic/versions/0007_user_api_keys_and_2fa.py):
- Создаёт таблицу `user_api_keys` (`id`, `user_id` FK→`users.id` CASCADE, `label`, `key_prefix` UNIQUE INDEX, `key_hash`, `created_at`).
- Добавляет колонки `users.two_fa_enabled BOOLEAN DEFAULT false NOT NULL` и `users.two_fa_secret VARCHAR(64) NULL`.
- Реверс — корректный (drop columns, drop table).

[backend/alembic/env.py:19](backend/alembic/env.py#L19) — импорт `models.user_api_key` (для регистрации таблицы в metadata).

### 3. Модели

[backend/src/models/user.py](backend/src/models/user.py): добавлены поля `two_fa_enabled: bool` и `two_fa_secret: str | None`. Импорт `Boolean` подтянул.

Новый [backend/src/models/user_api_key.py](backend/src/models/user_api_key.py): `UserApiKey` по паттерну `ExchangeCredential`. Полный секрет в БД не хранится — только `key_prefix` (12 символов) для быстрого поиска и `key_hash` (bcrypt).

### 4. Схемы Pydantic

Новый [backend/src/api/schemas/cabinet.py](backend/src/api/schemas/cabinet.py): `ApiKeyCreateIn`, `ApiKeyOut`, `ApiKeyCreateOut` (наследует `ApiKeyOut` + `key` для одноразового показа), `TwoFaStateOut`, `TwoFaSetupOut`, `TwoFaVerifyIn` (с regex-валидацией `^\d{6}$`), `TestConnectionOut`.

### 5. Репозиторий и сервис ключей

Новый [backend/src/repositories/api_key_repo.py](backend/src/repositories/api_key_repo.py): `list_for_user`, `get`, `get_by_prefix`, `create`, `delete`. Сортировка по `created_at desc`.

Новый [backend/src/services/api_key_service.py](backend/src/services/api_key_service.py):
- `_generate_key()` → `secret = "cd_" + secrets.token_hex(16)` (35 символов), `prefix = secret[:12]`, `hash = bcrypt.hashpw(...)`.
- `ApiKeyService.generate(user_id, label)` → возвращает `(UserApiKey, secret)`. Дефолтный label «Ключ от 25.05.2026».
- `ApiKeyService.delete_for_user(user_id, key_id)` — проверяет принадлежность; бросает `ApiKeyError`, если чужой.
- `ApiKeyService.verify(secret)` — O(1) поиск по `key_prefix`, затем `bcrypt.checkpw`. Используется внешним эндпоинтом `/api/external/test`.

### 6. Сервис 2FA

Новый [backend/src/services/two_fa_service.py](backend/src/services/two_fa_service.py):
- `setup(user)` → `pyotp.random_base32()`, сохраняет в Redis с ключом `2fa_setup:{user_id}` TTL=300с, возвращает `otpauth://totp/Crypto Dashboard:{email}?secret=…&issuer=Crypto%20Dashboard`. Секрет в БД пока **не** пишется — иначе незавершённое сканирование снесло бы старую 2FA.
- `verify(user, code)` → читает секрет из Redis, проверяет через `pyotp.TOTP(secret).verify(code, valid_window=1)` (±30 сек дрейфа часов), при успехе записывает в `user.two_fa_secret` + ставит флаг, удаляет ключ из Redis.
- `disable(user)` → обнуляет оба поля.

[backend/src/repositories/user_repo.py:35-37](backend/src/repositories/user_repo.py#L35) — добавлен публичный `flush()` для использования из сервиса (вместо прокола абстракции и обращения к `_session`).

### 7. Роутер кабинета

Новый [backend/src/api/routers/cabinet.py](backend/src/api/routers/cabinet.py): `/api/cabinet/*` с `CurrentUser` на каждом endpoint'е:
- `GET /api/cabinet/api-keys` → list
- `POST /api/cabinet/api-keys` → создать (201, returns `ApiKeyCreateOut` с полным ключом)
- `DELETE /api/cabinet/api-keys/{key_id}` → 204; 404 если чужой
- `GET /api/cabinet/2fa` → `{enabled}`
- `POST /api/cabinet/2fa/setup` → `{otpauth_uri}`
- `POST /api/cabinet/2fa/verify` → 204; 400 если код неверный / сессия истекла
- `DELETE /api/cabinet/2fa` → 204

`RedisDep = Annotated[Redis, Depends(get_redis)]` определён локально в роутере (импорт `from redis.asyncio import Redis`).

### 8. Внешний роутер

Новый [backend/src/api/routers/external.py](backend/src/api/routers/external.py): `POST /api/external/test`:
- Читает `Authorization: Bearer <api_key>` и `X-Client-Id: <uuid>` из заголовков.
- Парсит UUID Client ID, верифицирует ключ через `ApiKeyService.verify`, проверяет `item.user_id == client_uuid`.
- При успехе → `{"message": "This is a test API connection implementation"}` (фиксированная английская строка, как заказывал пользователь).
- При любой неудаче → 401.

### 9. Регистрация роутеров

[backend/src/main.py:11-26](backend/src/main.py#L11) — импорт `cabinet, external` в существующий блок.
[backend/src/main.py:144-145](backend/src/main.py#L144) — `app.include_router(cabinet.router)` и `app.include_router(external.router)`.

## Что сделано — Frontend

### 1. Удаление мок-инфраструктуры

Удалены файлы:
- `frontend/src/mock/config.ts`
- `frontend/src/mock/store.ts`
- `frontend/src/mock/generators.ts`
- `frontend/src/app/MockDataContext.tsx`
- `frontend/src/test/mockStore.test.ts`

Снят гейт `if (isMockEnabled())` из 10 api-файлов: `backtest.ts`, `balances.ts`, `bots.ts`, `candles.ts`, `credentials.ts`, `exchanges.ts`, `health.ts`, `positions.ts`, `trades.ts`, `cabinet.ts`. Каждый теперь напрямую делает `apiFetch(...)`.

В [frontend/src/app/components/Layout.tsx](frontend/src/app/components/Layout.tsx) убран `<MockDataProvider>` и импорт.

В [frontend/src/app/components/layout/Header.tsx](frontend/src/app/components/layout/Header.tsx) убран весь блок `<FormControlLabel>` со свитчером «мок-данные», импорты `Switch`, `FormControlLabel`, `useMockData`, и переменные `mockEnabled / setMockEnabled`. Иконка профиля и Logout остались.

Финальная проверка: `grep -rn "MockData\|mockStore\|isMockEnabled" frontend/src/` — пусто.

### 2. Тема: глобальные overrides для модалок

[frontend/src/app/theme.ts](frontend/src/app/theme.ts) — добавлены overrides:
- `MuiDialog.paper`: `borderRadius: 20`, `bgcolor: rgba(20,20,22,0.85)`, `border: 1px solid rgba(255,255,255,0.08)`, `backdropFilter: blur(24px)`, `boxShadow: 0 24px 64px rgba(0,0,0,0.5)`.
- `MuiBackdrop.root`: `bgcolor: rgba(0,0,0,0.6)` + `backdropFilter: blur(4px)`. `invisible` оставлен прозрачным.
- `MuiDialogTitle/Content/Actions`: единые отступы 24/28px и `gap: 8` у Actions.

### 3. PerplexityDialog

Новый [frontend/src/app/components/PerplexityDialog.tsx](frontend/src/app/components/PerplexityDialog.tsx): обёртка над `<Dialog>`, принимает `open, onClose, title, subtitle, maxWidth, fullWidth, children, actions, disableClose`. Render:
- Абсолютно-позиционированный `<IconButton>` с `<Close>` в правом верхнем углу.
- `<DialogTitle>` с `pr: 6` под X, subtitle через `<Stack>`.
- `disableClose` блокирует ESC и backdrop-click (нужно во время API-вызовов).

### 4. Strategies — Dialog → PerplexityDialog

[frontend/src/app/pages/Strategies.tsx](frontend/src/app/pages/Strategies.tsx) — диалог «Редактировать стратегию» переписан на `<PerplexityDialog>`. Импорты `Dialog, DialogTitle, DialogContent, DialogActions` удалены. Передаётся `title="Редактировать стратегию"`, `subtitle="${strategy_class} · ${symbol} · ${timeframe}"`, `disableClose={savingEdit}`, кнопки в `actions={…}`.

### 5. Cabinet — полная переписка

[frontend/src/app/pages/Cabinet.tsx](frontend/src/app/pages/Cabinet.tsx):
- Удалены `pseudoQr()` и связанная CSS-сетка.
- Импортирован `QRCodeSVG` из `qrcode.react`.
- Новый поток 2FA:
  - При вкл → `setupTwoFa()` → получает `otpauth_uri` → открывает `<PerplexityDialog>`.
  - В диалоге `<QRCodeSVG value={otpauthUri} size={200} bgColor="#ffffff" fgColor="#0a0a0a" />` — настоящий QR, сканируется Google Authenticator / 1Password / Microsoft Authenticator.
  - Поле ввода 6 цифр → «Подтвердить» → `verifyTwoFa(code)` → toast «2FA включена».
  - При выкл → `disableTwoFa()` сразу без диалога.
- `generateApiKey()` → теперь `POST /api/cabinet/api-keys` → возвращает `ApiKeyCreateOut` с полным секретом.
- `revealedKey` хранит весь `ApiKeyCreateOut` — для копирования и для теста.
- `listApiKeys()` теперь возвращает `ApiKeyOut[]` без секрета — только `key_prefix`. Отображается `{key_prefix}…` в моноширинном шрифте.
- «Тест подключения» доступен **только для только что созданного ключа** (когда `revealedKey?.id === k.id`) — у остальных ключей в БД хранится только bcrypt-хеш, плейн-секрет утрачен. На прочих ключах кнопка `disabled` с пояснительным тултипом. Это явное архитектурное решение, отражающее security-стандарт «secret-only-once» (как GitHub PAT).
- Все диалоги Cabinet (один — 2FA setup) переехали на `<PerplexityDialog>`.

### 6. API-слой кабинета

[frontend/src/api/cabinet.ts](frontend/src/api/cabinet.ts) — переписан целиком без `isMockEnabled`:
- `listApiKeys()`, `generateApiKey(label?)`, `deleteApiKey(id)`, `getTwoFa()`, `setupTwoFa()`, `verifyTwoFa(code)`, `disableTwoFa()` — все через `apiFetch`.
- `testConnection(apiKey, clientId)` — **не** использует `apiFetch`, потому что `apiFetch` принудительно проставляет JWT в `Authorization` и затирает наш Bearer-ключ. Делаем чистый `fetch` к `${API_ORIGIN}/api/external/test` с `Authorization: Bearer ${apiKey}` и `X-Client-Id: ${clientId}`, бросаем `ApiHttpError` на не-2xx.

### 7. Типы

[frontend/src/api/types.ts:111-128](frontend/src/api/types.ts#L111) — обновлены:
- `ApiKeyOut` теперь имеет `key_prefix` (не `key`).
- Добавлен `ApiKeyCreateOut extends ApiKeyOut` с `key: string`.
- Добавлен `TwoFaSetupOut { otpauth_uri }`.

### 8. Зависимости

[frontend/package.json](frontend/package.json) — добавлено `qrcode.react: 4.2.0` через `pnpm add -w qrcode.react`.

## Файлы

**Backend — новые (8):**
- `backend/src/models/user_api_key.py`
- `backend/src/api/schemas/cabinet.py`
- `backend/src/repositories/api_key_repo.py`
- `backend/src/services/api_key_service.py`
- `backend/src/services/two_fa_service.py`
- `backend/src/api/routers/cabinet.py`
- `backend/src/api/routers/external.py`
- `backend/alembic/versions/0007_user_api_keys_and_2fa.py`

**Backend — изменены (5):**
- `backend/pyproject.toml` (+pyotp)
- `backend/src/models/user.py` (+2FA columns)
- `backend/src/main.py` (+routers)
- `backend/src/api/deps.py` — нет изменений (cabinet router сам определяет RedisDep)
- `backend/src/repositories/user_repo.py` (+flush helper)
- `backend/alembic/env.py` (+user_api_key import)

**Frontend — новые (1):**
- `frontend/src/app/components/PerplexityDialog.tsx`

**Frontend — изменены (15):**
- `frontend/src/app/theme.ts` (+Dialog overrides)
- `frontend/src/api/cabinet.ts` (переписан полностью)
- `frontend/src/api/types.ts` (ApiKeyOut, ApiKeyCreateOut, TwoFaSetupOut)
- `frontend/src/api/backtest.ts`, `balances.ts`, `bots.ts`, `candles.ts`, `credentials.ts`, `exchanges.ts`, `health.ts`, `positions.ts`, `trades.ts` (убран mock-гейт)
- `frontend/src/app/pages/Strategies.tsx` (Dialog → PerplexityDialog)
- `frontend/src/app/pages/Cabinet.tsx` (новый 2FA-поток, реальный QR, тест по полному ключу)
- `frontend/src/app/components/Layout.tsx` (убрал MockDataProvider)
- `frontend/src/app/components/layout/Header.tsx` (убрал mock-свитч)
- `frontend/package.json` (+qrcode.react)

**Frontend — удалены (5):**
- `frontend/src/mock/config.ts`
- `frontend/src/mock/store.ts`
- `frontend/src/mock/generators.ts`
- `frontend/src/app/MockDataContext.tsx`
- `frontend/src/test/mockStore.test.ts`

## Verification

**Автоматические проверки прошли:**
```bash
cd frontend && pnpm build  # ✓ 4.60s, 1112.23 kB / 330.80 kB gzip
cd frontend && pnpm test   # ✓ 94/94 tests passed (8 файлов; -4 теста удалённого mockStore.test.ts)
python3 -c "import ast; [ast.parse(open(p).read()) for p in <all touched backend files>]"  # ✓ all ok
```

**Бэк-тесты на dev-машине не запускались** — для них нужны Postgres + Redis. Миграция и интеграция проверится при деплое (`docker compose run --rm migrate` в deploy.yml).

**Ручная верификация (после деплоя):**

| # | Что проверить | Где |
|---|--------------|-----|
| 1 | В шапке нет свитчера «мок-данные» | везде |
| 2 | Все диалоги имеют округлые углы 20px, glass-фон, X в правом верхнем углу, размытый бэкдроп | `/strategies` → клик по карточке; `/cabinet` → 2FA |
| 3 | `/cabinet`: «Сгенерировать API ключ» создаёт ключ, Alert показывает полный `cd_xxxx…` | `/cabinet` |
| 4 | После reload список ключей подгружается с сервера (только `key_prefix`, без полного ключа) | `/cabinet` |
| 5 | «Тест подключения» сразу после создания → Alert с **«This is a test API connection implementation»** | `/cabinet` |
| 6 | Включение 2FA → реальный QR, сканируется Google Authenticator/1Password | `/cabinet` → 2FA |
| 7 | Ввод правильного 6-значного кода → 2FA включается; неправильный → ошибка | `/cabinet` |
| 8 | Удаление API ключа работает | `/cabinet` |
| 9 | В DevTools Network: `POST /api/cabinet/api-keys 201`, `GET /api/cabinet/api-keys 200`, `POST /api/cabinet/2fa/setup 200`, `POST /api/cabinet/2fa/verify 204`, `POST /api/external/test 200` | `/cabinet` |
| 10 | `docker logs crypto-backend` показывает запросы к `/api/cabinet/*` | сервер |

## Деплой (для пользователя)

```bash
# 1. Frontend submodule
cd frontend
git add -A
git commit -m "feat(cabinet): real 2FA QR + bcrypt API keys; unify modals; drop mock mode"
git push origin main
cd ..

# 2. Backend submodule
cd backend
git add -A
git commit -m "feat(cabinet): /api/cabinet/* + /api/external/test, pyotp 2FA, bcrypt-hashed API keys, migration 0007"
git push origin main
cd ..

# 3. Root repo — bump submodule pointers
git add frontend backend
git commit -m "bump frontend + backend: phase 9 (modals, no-mock, real cabinet)"
git push origin main
```

После push GH Actions [.github/workflows/deploy.yml](.github/workflows/deploy.yml):
1. Соберёт три образа.
2. Прогонит `docker compose run --rm migrate` — применит миграцию `0007`.
3. Перезапустит `crypto-backend`, `crypto-frontend`, `crypto-engine`.

Проверка через ~5-10 минут:
```bash
ssh -p 2244 vadim_denisovich@31.200.229.59 'docker logs --tail 30 crypto-backend | grep cabinet'
```

Если миграция упадёт — `ssh … 'cd /opt/<deploy-path> && docker compose run --rm migrate'` для диагностики.

## Заметки

- **Security-decision: bcrypt-хеш с показом секрета один раз.** Полный ключ показывается в `<Alert>` сразу после генерации; после reload его нельзя получить повторно. Это стандарт (GitHub PAT, AWS access keys) — он же обусловливает поведение кнопки «Тест подключения»: тест работает только для свежесгенерированного ключа в той же сессии.
- **2FA-поток**: секрет на бэке сохраняется только после успешной верификации первого кода. До этого живёт 5 мин в Redis под `2fa_setup:{user_id}`. Если пользователь закрыл вкладку, не подтвердив, — старая 2FA (если была включена) не пострадает.
- **Тест API ключа отправляется plain `fetch`**, а не через `apiFetch`, потому что последний принудительно подставляет JWT-токен в `Authorization`, перекрывая Bearer с API-ключом. Это была нерешённая проблема в первоначальной версии cabinet.ts.
- **94 теста проходят** (было 98 — 4 теста из `mockStore.test.ts` удалены вместе с самим стором).
- **Бандл вырос на +2KB gzipped** (с 329 до 331 KB) из-за `qrcode.react`. В приемлемых пределах.
