# CC Security Audit — cc.zxcv.app

Репозиторий с результатами security-аудита сайта CYBERBOX.

## Файлы

| Файл | Описание |
|---|---|
| [SECURITY_AUDIT.md](SECURITY_AUDIT.md) | Полный аналитический отчёт: уязвимости, код, рекомендации |

## Краткий итог

- **2 CRITICAL** — auth cookies без `httpOnly`/`secure`
- **4 HIGH** — CSRF, нет CSP, нет CAPTCHA, FingerprintJS как security-токен
- **6 MEDIUM** — API disclosure, localStorage, client-only validation, clickjacking
- **4 LOW** — нет security.txt, sitemap, build ID виден, кросс-домен ассеты

## Самое важное исправление

Добавить `httpOnly: true, secure: true, sameSite: "Strict"` к установке cookies `access_token` и `refresh_token`. Одна строка закрывает CRITICAL×2 + HIGH (CSRF).
