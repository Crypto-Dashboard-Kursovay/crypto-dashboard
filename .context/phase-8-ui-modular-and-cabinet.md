# Phase 8 — UI-доработки + личный кабинет

Дата: 2026-05-25
Ветка: `main` (фронтенд-submodule)

## Context

Пакетная доработка фронтенда `crypto-dashboard` по результатам ревью UI + новая фича «Личный кабинет».

UI-правки:
1. Тултип на графике котировок отображался дефолтным белым прямоугольником recharts — нечитаем поверх тёмной темы.
2. Свитчер таймфреймов и селектор пары стояли рядом, но визуально не сочетались.
3. На карточке стратегии иконка-карандаш дублировала естественное ожидание «клик по карточке = редактировать».
4. В форме бэктеста блок «Параметры стратегии» был обёрнут в чёрный контейнер, создавая двойную рамку поверх Card.

Новая фича — личный кабинет:
- Иконка профиля в шапке вместо e-mail, по клику открывается отдельная страница `/cabinet`.
- E-mail, Client ID с copy-on-hover, свитчер 2FA с QR-диалогом, генерация и список API-ключей, кнопка «Тест подключения» → возвращает английский ответ-заглушку будущего бэка.

Цель: убрать UX-шероховатости и заложить контракт для программного доступа к будущему API.

## Что сделано

### 1. Тултип графика

[frontend/src/app/components/dashboard/ChartWidget.tsx:203-221](frontend/src/app/components/dashboard/ChartWidget.tsx#L203-L221)

К `<Tooltip>` из recharts добавлены `contentStyle`, `labelStyle`, `itemStyle`, `cursor`. Фон `rgba(20,20,22,0.95)`, alpha-бордер `rgba(255,255,255,0.08)`, тень `0 10px 40px rgba(0,0,0,0.4)`, backdrop-blur 20px. Label — серый `#94a3b8`, value — светлый `#f8fafc`. Cursor — пунктирная линия 30% непрозрачности.

### 2. Свитчер таймфреймов и Select пары — единый glass-стиль

[frontend/src/app/components/dashboard/ChartWidget.tsx:103-149](frontend/src/app/components/dashboard/ChartWidget.tsx#L103-L149)

**Select пары:** на `sx` Select-а явно проставлены `bgcolor: "rgba(20,20,22,0.6)"`, `backdropFilter: "blur(20px)"` и alpha-цвет `notchedOutline` (`0.12` / `0.25` на hover) — поверх дефолтов темы. Это синхронизирует селектор с соседним свитчером.

**Стек таймфреймов:** заменена обёртка `bgcolor="background.default" borderRadius={1}` на полноценный glass-морф: `rgba(20,20,22,0.6)` + `border: "1px solid rgba(255,255,255,0.12)"` + `borderRadius: 3` (= 12px) + `backdropFilter: blur(20px)`. Каждая кнопка-таймфрейм получила `borderRadius: 2` чтобы активная белая пилюля лучше вписывалась.

### 3. Карточка стратегии: клик по карточке → редактирование

[frontend/src/app/pages/Strategies.tsx:20-29, 253-345](frontend/src/app/pages/Strategies.tsx#L20-L29)

- Удалён импорт `EditOutlined`.
- Удалён `IconButton` с карандашом из строки действий.
- Header + блок «Пара/Таймфрейм/Параметры» обёрнуты в кликабельный `<Box>` с:
  - `role="button"`, `tabIndex={0}`, `aria-label="Редактировать ${bot.strategy_class}"`
  - `onClick={() => onEdit(bot)}`
  - `onKeyDown` для Enter / Space (доступность с клавиатуры)
  - `cursor: "pointer"`, hover-эффект `opacity: 0.85`
  - `:focus-visible` outline для tab-навигации
- Кнопки Start / Stop / Delete остались вне кликабельной зоны — никакого риска поймать клик при попытке остановить бота.
- Существующий `onEdit` handler и Dialog из строк 341–387 переиспользованы без изменений.

### 4. «Параметры стратегии» в форме бэктеста

[frontend/src/app/pages/Backtesting.tsx:403-428](frontend/src/app/pages/Backtesting.tsx#L403-L428)

Чёрный `<Box>` (с `bgcolor: "background.default"` + `border` + `borderRadius`) заменён на простой `<Box sx={{ mt: 1.5 }}>`. Заголовок и поля теперь визуально однородны с дате/депозитом выше — разделитель только верхний отступ.

### 5. Личный кабинет

#### 5.1. Иконка профиля в шапке

[frontend/src/app/components/layout/Header.tsx:11-18, 95-104](frontend/src/app/components/layout/Header.tsx#L95-L104)

E-mail-текст заменён на `<IconButton component={RouterLink} to="/cabinet">` с иконкой `AccountCircleOutlined` и тултипом «Личный кабинет». Импортированы `AccountCircleOutlined` и `Link as RouterLink`. Mock-свитч и Logout оставлены на местах.

#### 5.2. Новый роут `/cabinet`

[frontend/src/app/routes.tsx:11, 32](frontend/src/app/routes.tsx#L11)

Добавлен импорт `Cabinet` и запись `{ path: "cabinet", Component: Cabinet }` в защищённый layout.

#### 5.3. Страница `Cabinet.tsx`

Новый файл: [frontend/src/app/pages/Cabinet.tsx](frontend/src/app/pages/Cabinet.tsx)

Три карточки с glass-морфом (наследует от темы):

**Профиль:**
- E-mail заголовком h3.
- Client ID моноширинным шрифтом. При hover на строке проявляется иконка «копировать» (CSS-анимация opacity). Клик → `navigator.clipboard.writeText(user.id)` + toast «Client ID скопирован».

**2FA:**
- Лейаут с `LockOutlined`-иконкой, заголовком и подзаголовком («Включена…» / «Отключена — рекомендуем включить»).
- `<Switch color="success">` справа.
- При включении открывается `<Dialog maxWidth="xs">` с псевдо-QR-кодом 21×21 (детерминированный на основе FNV-1a хеша от `user.id` — без новых зависимостей) и `<TextField inputMode="numeric">`. Кнопка «Подтвердить» доступна только при 6 цифрах. При успехе — `cabinetApi.setTwoFa(true)` + toast.
- При выключении — диалога нет, сразу `setTwoFa(false)` + toast.

**API ключи:**
- Заголовок + кнопка «Сгенерировать API ключ» (`<Add/>` иконка).
- После генерации показывается `<Alert severity="success">` с полным ключом и кнопкой копирования (один раз — секрет).
- Список ключей: каждый ключ — `<Box border>` с `<Chip>API</Chip>`, лейблом, маской ключа (`crypto_xxxx…yyyy`), датой создания.
- Справа от ключа — иконки «Тест подключения» (`<PlayArrow>`) и «Удалить» (`<DeleteOutline>`).
- «Тест подключения» вызывает `cabinetApi.testConnection(key, clientId)` и показывает результат в `<Alert>` под строкой: **«This is a test API connection implementation»** (английский, как заказывал пользователь).
- Пустое состояние: «Ещё нет ни одного ключа. Сгенерируйте первый — он появится здесь.»

#### 5.4. API-слой `frontend/src/api/cabinet.ts`

Новый файл: [frontend/src/api/cabinet.ts](frontend/src/api/cabinet.ts)

Тонкие обёртки `listApiKeys` / `generateApiKey` / `deleteApiKey` / `getTwoFa` / `setTwoFa` / `testConnection`. Каждая проверяет `isMockEnabled()`:
- В mock-режиме → делегат к `mockStore.*`
- В реальном режиме → `apiFetch<…>("/api/cabinet/…")` (бэкенда пока нет — это TODO для следующей фазы, см. комментарий в начале файла).
- `testConnection` всегда возвращает фиксированное `{ message: "This is a test API connection implementation" }` с 350ms-задержкой — это контракт временной реализации, который не зависит от сети и не требует бэка прямо сейчас.

#### 5.5. Расширение MockStore

[frontend/src/mock/store.ts](frontend/src/mock/store.ts) — добавлены:
- Поля `apiKeys: ApiKeyOut[] = []`, `twoFaEnabled = false` (строки 105–106).
- Поля `apiKeys` и `twoFaEnabled` в `PersistedSession` (опциональные — обратная совместимость с уже сохранёнными сессиями).
- Чтение из `tryRestore()` (с дефолтами `?? [] / ?? false`).
- Запись в `persist()`.
- Публичные методы в конце класса (после `makeBacktestResult`):
  - `listApiKeys()` — сорт по `created_at desc`
  - `generateApiKey(label?)` — генерит `crypto_${uuid}${uuid}` без дефисов, дефолтный label «Ключ от 25.05.2026»
  - `deleteApiKey(id)`
  - `getTwoFa()` / `setTwoFa(b)`

Все мутирующие методы вызывают `persist()` — ключи и 2FA переживают перезагрузку страницы.

#### 5.6. Новые типы

[frontend/src/api/types.ts:111-122](frontend/src/api/types.ts#L111)

Добавлены `ApiKeyOut { id, label, key, created_at }` и `TestConnectionOut { message }`.

#### 5.7. Toast-уведомления через sonner

[frontend/src/app/components/Layout.tsx:5, 144](frontend/src/app/components/Layout.tsx#L5)

Подключён `<Toaster richColors position="bottom-right" theme="dark" />` в корне Layout (sonner уже был в `package.json`). В Cabinet используются `toast.success(...)` / `toast.error(...)` для подтверждений и ошибок.

## Файлы

**Изменены (8):**
- `frontend/src/app/components/dashboard/ChartWidget.tsx` — тултип, свитчер, Select пары
- `frontend/src/app/pages/Strategies.tsx` — убрали карандаш, кликабельная карточка
- `frontend/src/app/pages/Backtesting.tsx` — сняли чёрную подложку
- `frontend/src/app/components/layout/Header.tsx` — иконка профиля вместо e-mail
- `frontend/src/app/components/Layout.tsx` — `<Toaster/>`
- `frontend/src/app/routes.tsx` — роут `/cabinet`
- `frontend/src/api/types.ts` — `ApiKeyOut`, `TestConnectionOut`
- `frontend/src/mock/store.ts` — поля + методы для apiKeys и 2FA

**Созданы (2):**
- `frontend/src/api/cabinet.ts` — API-слой
- `frontend/src/app/pages/Cabinet.tsx` — страница

## Verification

Запущены автоматические проверки:

```bash
cd frontend
pnpm build  # ✓ vite build OK, 4.35s, 1110.07 kB / 329.26 kB gzip
pnpm test   # ✓ 98/98 passed (8 файлов), 4.66s
```

Ручная проверка в dev-режиме (рекомендуемый чек-лист):

| # | Что проверить | Где |
|---|--------------|-----|
| 1 | Тултип на графике читаемый, тёмный с alpha-обводкой | `/` — Dashboard, наведение на линию |
| 2 | Свитчер таймфреймов и Select пары выглядят как братья (glass-морф) | `/` — над графиком |
| 3 | Карандаш на карточке стратегии отсутствует; клик по информационной части открывает диалог редактирования; клики по Start/Stop/Delete диалог не открывают | `/strategies` |
| 4 | Tab-навигация на карточке: focus-ring появляется, Enter/Space открывают диалог | `/strategies` |
| 5 | В блоке «Параметры стратегии» формы бэктеста нет чёрной подложки и второй рамки | `/backtesting` |
| 6 | В шапке вместо e-mail — иконка профиля с тултипом «Личный кабинет» | везде |
| 7 | Клик по иконке → `/cabinet`, отображается e-mail и Client ID | `/cabinet` |
| 8 | Hover на строке Client ID показывает иконку «копировать»; клик → тост «Client ID скопирован» | `/cabinet` |
| 9 | Свитчер 2FA → диалог с QR + поле ввода 6 цифр → «Подтвердить» → тост «2FA включена» | `/cabinet` |
| 10 | Кнопка «Сгенерировать API ключ» → Alert с полным ключом + кнопка копирования; ключ появляется в списке ниже | `/cabinet` |
| 11 | «Тест подключения» рядом с ключом → ниже строки появляется Alert с **«This is a test API connection implementation»** | `/cabinet` |
| 12 | Перезагрузка страницы — 2FA и ключи сохраняются (через MockStore.persist) | `/cabinet` |
| 13 | Real-mode (mock-свитч в шапке выключен) — apiFetch к `/api/cabinet/*` ожидаемо вернёт 404 до реализации бэка; `testConnection` всё равно возвращает английскую заглушку | mock toggle |

## TODO для последующего бэк-этапа

- Реализовать на бэке эндпоинты `/api/cabinet/api-keys` (GET/POST/DELETE) и `/api/cabinet/2fa` (GET/PUT).
- Реализовать `/api/external/*` — приём запросов по связке API-ключ + Client ID. Текущая заглушка с фиксированной английской строкой задаёт ожидаемый формат ответа: `{ "message": "This is a test API connection implementation" }`.
- 2FA: заменить mock-QR на реальный TOTP (например, `otpauth-uri` + библиотека `qrcode.react`) и валидацию кода через `pyotp` на бэке.
