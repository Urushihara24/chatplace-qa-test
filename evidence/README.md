# EVIDENCE — доказательная база дефектов

Скриншоты, подтверждающие дефекты и наблюдения из `test-artifacts/TEST_MATRIX.md`.
Имена файлов ниже = реальные имена приложенных изображений = ссылки из реестра (traceability по ID).

> **Security-примечание (Public-репозиторий).** На всех скринах DevTools **зачищены секреты**
> прямоугольником поверх: JWT `x-auth-token: eyJ0…` целиком, Stripe `pk_live_…`,
> при желании `project_id` / `bot_id`. Оставлено только то, что является сутью доказательства
> и не содержит секретов: HTTP-статус, тело ответа, список админов канала, текст варнингов.
> Это соответствие security-чеклисту (*secrets / data in transit*): публичный репозиторий
> не должен содержать токенов, даже собственных.

## Индекс доказательств

| Файл | Привязка | Что показывает | Как воспроизвести | Зачищено |
|---|---|---|---|---|
| `bug01_empty_channels.png` | BUG-01, TC-06, AC-3 | DevTools → Network → `GET /telegram-channels`: вкладка General/Headers — путь запроса и `Status Code 200 OK` | F12 → Network → клик «Бот добавлен» → запрос `telegram-channels` → Headers | `X-Auth-Token` оставлен вне кадра |
| `bug01_empty_response.png` | BUG-01, TC-06, TC-09, AC-3/AC-4 | Вкладка Response/Preview того же запроса: тело `{type:"result", data:[]}` — пустой массив при подтверждённом админстве | тот же запрос → вкладка Response | не требуется (секретов нет) |
| `bug01_bot_is_admin.png` | BUG-01 (контраст) | Telegram → канал → Администраторы: `TestCalc — админ`. Доказывает, что пустой массив — не вина пользователя | Telegram → канал → название → Администраторы | не требуется |
### OBS-01 — дамп Console (текстовое evidence, бинарник не приложен)
Зафиксировано на экране оплаты/подписки (stripe.js, чанк `CNGlEfIp.js` / `paymentblocks.js`):
```
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'saveCard'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'debug'
CNGlEfIp.js:28 Unsupported 'PaymentOptions' field: 'sbpSupport'
VM821 paymentblocks.js:1 Allow attribute will take precedence over 'allowpaymentrequest'.
```
Интерпретация: продукт передаёт в Stripe Payment Element поля, которых текущая версия stripe.js не знает → устаревший SDK / некорректная конфигурация; `allowpaymentrequest` — deprecated-атрибут. Нарушает Definition of Done (*«нет console errors / необработанных ошибок в UI»*). Риск: часть платёжных опций (СБП, сохранение карты) может не применяться. Секретов в дампе нет — `pk_live` в этих строках не фигурирует.

## Как читать контраст по BUG-01
Два первых скрина работают **в паре**: `bug01_bot_is_admin.png` доказывает, что на стороне
платформы бот реально администратор канала (Telegram это подтверждает), а
`bug01_empty_channels.png` показывает, что ChatPlace при этом возвращает пустой массив и
блокирует онбординг без feedback. Расхождение «платформа = да / продукт = нет» и есть суть
дефекта и рассинхрона контракта API↔UI. Машинный аналог этого контраста — пара запросов в
`api/chatplace_keyword_bot.postman_collection.json` (Telegram `getChatMember` vs ChatPlace
`telegram-channels`).

## Статус приложения
- Если изображения загружены в эту папку — таблица выше является их индексом.
- Если изображения **не** приложены — считайте таблицу **спецификацией ожидаемой доказательной
  базы**: какие именно снимки, с каких экранов и с какой зачисткой должны сопровождать дефекты
  при передаче в разработку (требование воспроизводимости из обязательного пакета артефактов).
