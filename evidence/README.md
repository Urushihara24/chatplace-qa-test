# EVIDENCE — доказательная база дефектов

Скриншоты и текстовые дампы, подтверждающие дефекты и наблюдения из `test-artifacts/TEST_MATRIX.md`.
Имена файлов ниже = реальные имена приложенных изображений = ссылки из реестра (traceability по ID).

> **Security-примечание (Public-репозиторий).** На всех скринах DevTools **зачищены секреты**:
> JWT `x-auth-token: eyJ0…` оставлен вне кадра, Stripe `pk_live_…` не попадает в кадр,
> при желании замазаны `project_id` / `bot_id`. Оставлено только то, что является сутью
> доказательства и не содержит секретов: HTTP-статус, тело ответа, список админов канала,
> текст варнингов/ошибок. Это соответствие security-чеклисту (*secrets / data in transit*):
> публичный репозиторий не должен содержать токенов, даже собственных.

## Индекс доказательств

| Файл | Привязка | Что показывает | Как воспроизвести | Зачищено |
|---|---|---|---|---|
| `bug01_empty_channels.png` | BUG-01, TC-06, AC-3 | DevTools → Network → `GET /telegram-channels`: вкладка General/Headers — путь запроса и `Status Code 200 OK` | F12 → Network → клик «Бот добавлен» → запрос `telegram-channels` → Headers | `X-Auth-Token` оставлен вне кадра |
| `bug01_empty_response.png` | BUG-01, TC-06, TC-09, AC-3/AC-4 | Вкладка Response/Preview того же запроса: тело `{type:"result", data:[]}` — пустой массив при подтверждённом админстве | тот же запрос → вкладка Response | не требуется (секретов нет) |
| `bug01_bot_is_admin.png` | BUG-01 (контраст) | Telegram → канал → Администраторы: `TestCalc — админ`. Доказывает, что пустой массив — не вина пользователя | Telegram → канал → название → Администраторы | не требуется |
| `bug03_cloudpayments_500.png` | BUG-03, TC-18, AC-1 | DevTools → Network/Headers: `POST …/create-cloudpayments` = красный `500 Internal Server Error` (HTTP-статус не соответствует business-исходу) | F12 → Network → подтвердить оплату картой без средств → запрос `create-cloudpayments` → Headers | `X-Auth-Token` вне кадра |
| `bug03_response_body.png` | BUG-03, TC-18, AC-1 | Вкладка Response того же запроса: тело `{type:"error", data:{code:0, message:"Недостаточно денег на карте", level:"business"}}`. Доказывает, что бизнес-слой отработал (level:"business"), а нарушен только HTTP-маппинг; `code:0` = отсутствие типизации ошибки | тот же запрос → вкладка Response | не требуется (секретов нет) |
| `bug03_decline_ui.png` | BUG-03 (контраст), TC-18, AC-1 | UI при том же decline: fallback «Недостаточно денег на карте» + «Повторить». Доказывает, что фронт показал message из тела, а подзаголовок — хардкод; текст копии противоречив | тот же сценарий → вид экрана после отказа | не требуется |

## Как читать контракты
- **BUG-01 (две стороны контракта):** `bug01_bot_is_admin.png` доказывает, что на стороне платформы бот реально администратор канала, а `bug01_empty_channels.png` + `bug01_empty_response.png` показывают, что ChatPlace при этом возвращает `200 OK` с пустым массивом и блокирует онбординг без feedback. Расхождение «платформа = да / продукт = нет» = суть дефекта. Машинный аналог — пара запросов в `api/chatplace_keyword_bot.postman_collection.json`.
- **BUG-03 (нарушение контракта `level → HTTP-код`, три проекции одного ответа):**`bug03_cloudpayments_500.png` (HTTP = 500), `bug03_response_body.png` (тело = level:"business", code:0) и `bug03_decline_ui.png` (UI = message из тела + хардкод-подзаголовок) вместе показывают: бизнес-логика отработала и фронт корректно прочитал message, но HTTP-код не соответствует level ответа, а code:0 не даёт типизировать ошибку. Это не «тупик» и не «сервер лежит» — это точечный дефект маппинга `level → HTTP-код`.

## OBS-01 — дамп Console (текстовое evidence, бинарник не приложен)
Зафиксировано на экране оплаты/подписки (платёжный SDK, чанки `CNGlEfIp.js` / `paymentblocks.js`):
```
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'saveCard'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'debug'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'sbpSupport'
VM821 paymentblocks.js:1 Allow attribute will take precedence over 'allowpaymentrequest'.
```
Интерпретация: продукт передаёт в платёжный виджет поля, которых текущая версия SDK не знает → устаревший SDK / некорректная конфигурация; `allowpaymentrequest` — deprecated-атрибут. Нарушает Definition of Done (*«нет console errors / необработанных ошибок в UI»*). Риск: часть платёжных опций (СБП, сохранение карты) может не применяться. Секретов в дампе нет — `pk_live` в этих строках не фигурирует, поэтому бинарник не обязателен.

## Статус приложения
Скрины BUG-01 и BUG-03 приложены в эту папку; таблица выше — их индекс. OBS-01 приведён текстовым дампом (достаточно для Trivial-наблюдения и безопаснее скрина с риском засветить ключи). Полный HAR-дамп сессии в репозиторий **не кладётся** по соображениям очистки секретов — доступен в зачищенном виде по запросу.
