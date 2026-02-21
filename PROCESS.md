# Рабочий процесс разработки — CYBERBOX

Документ описывает как должен быть организован процесс разработки продукта:
структуру проекта в Jira, Git-workflow и зрелый CI/CD pipeline с автотестами.

---

## Часть 1. Структура проекта в Jira

### 1.1 Иерархия задач

```
Project: CYBERBOX
│
├── Epic  (квартальная цель)
│     ├── Story  (ценность для пользователя, 1–5 дней)
│     │     ├── Task  (конкретная работа, < 1 дня)
│     │     └── Sub-task  (часть задачи)
│     └── Bug  (привязан к Story или к эпику)
│
└── Technical Debt  (отдельный тип, не попадает в product backlog)
```

**Принципы:**
- Story пишется от лица пользователя: *«Как игрок, я хочу видеть статус вывода, чтобы не беспокоиться»*
- Каждая Story имеет Acceptance Criteria (2–5 пунктов) и Definition of Done
- Bug создаётся с приоритетом P0–P3 и привязкой к среде воспроизведения

---

### 1.2 Эпики проекта

```mermaid
flowchart TD
    subgraph CRIT["🔴 До запуска (Sprint 1–2)"]
        E1["EPIC-1\nSecurity Foundation\n─────────────────\n• httpOnly + secure + sameSite cookies\n• CSP + security headers\n• CAPTCHA на login/register\n• Rate limiting на API"]
        E2["EPIC-2\nEconomics & Compliance\n─────────────────\n• Рефералы → выплаты от GGR\n• Задержка вывода 72h после карты\n• KYC при первом выводе\n• Убрать TEST в юр. документах"]
    end

    subgraph HIGH["🟠 Высокий приоритет (Sprint 3–5)"]
        E3["EPIC-3\nTrust & Transparency\n─────────────────\n• Provably Fair\n• SLA + статусы вывода\n• Онбординг 3 шага\n• Прогресс-бар вейджера"]
        E4["EPIC-4\nPayment Infrastructure\n─────────────────\n• PSP-каскад (основной + резервный)\n• 3DS2 для первого депозита\n• Risk scoring (IP, BIN, device)\n• Chargeback dispute API"]
    end

    subgraph MED["🟡 Средний приоритет (Sprint 6–8)"]
        E5["EPIC-5\nRetention & Engagement\n─────────────────\n• Daily Case Streak\n• Wager Race (недельный)\n• Email-реактивация D3/D7\n• Push-уведомления"]
        E6["EPIC-6\nAnalytics & Monitoring\n─────────────────\n• GGR/NGR dashboard\n• FTD CR воронка\n• PSP health monitoring\n• AML-алерты"]
    end

    subgraph GROWTH["📈 Growth (Sprint 9+)"]
        E7["EPIC-7\nAcquisition Channels\n─────────────────\n• TG Mini App + /api/auth/telegram\n• Google / Telegram OAuth\n• Affiliate панель (GGR-метрики)\n• Реферальная геймификация"]
    end

    CRIT --> HIGH --> MED --> GROWTH
```

---

### 1.3 Jira Workflow

```mermaid
flowchart LR
    BACKLOG["📋 Backlog"]
    REFINED["🔍 Refined\n(оценена, AC готовы)"]
    SPRINT["⚡ In Sprint"]
    DEV["💻 In Progress"]
    REVIEW["👀 Code Review"]
    QA["🧪 QA / Testing"]
    STAGING["🚀 On Staging"]
    DONE["✅ Done"]
    BLOCKED["🚫 Blocked"]

    BACKLOG -->|Grooming| REFINED
    REFINED -->|Sprint Planning| SPRINT
    SPRINT -->|Разработчик берёт| DEV
    DEV -->|PR открыт| REVIEW
    REVIEW -->|Approve| QA
    REVIEW -->|Changes requested| DEV
    QA -->|Pass| STAGING
    QA -->|Fail| DEV
    STAGING -->|PO acceptance| DONE
    DEV -->|Зависимость / блокер| BLOCKED
    BLOCKED -->|Блокер снят| DEV
```

**Правила перехода:**
- `Backlog → Refined`: оценена в SP, написаны AC, нет открытых вопросов
- `In Progress → Code Review`: PR открыт, линт и тесты зелёные, самопроверка пройдена
- `Code Review → QA`: минимум 1 апрув, CI pipeline зелёный
- `On Staging → Done`: PO или тестировщик подтвердил сценарии на стейджинге

---

### 1.4 Definition of Done

```
☐ Код покрыт юнит-тестами (≥ 80% новых строк)
☐ Интеграционные тесты обновлены / добавлены
☐ E2E-тест на критичный путь (если затронут флоу пользователя)
☐ PR-ревью: ≥ 1 апрув от другого разработчика
☐ CI pipeline зелёный (lint + unit + integration + SAST)
☐ Функциональность проверена на staging-среде
☐ Документация обновлена (если API изменился)
☐ PO подтвердил соответствие Acceptance Criteria
```

---

### 1.5 Ритм команды (4 человека)

| Церемония | Когда | Длительность | Участники |
|---|---|---|---|
| Sprint Planning | Пн, начало спринта | 1 ч | Все |
| Daily Standup | Каждый день | 10 мин | Все |
| Backlog Grooming | Ср, середина спринта | 45 мин | PO + Tech Lead |
| Sprint Review | Пт, конец спринта | 30 мин | Все |
| Retrospective | Пт, после Review | 30 мин | Все |

**Спринт:** 1 неделя. Velocity фиксируется с 3-го спринта, не раньше.

---

## Часть 2. Git Workflow & CI/CD Pipeline

### 2.1 Branch Strategy

```mermaid
gitGraph
   commit id: "init"
   branch dev
   checkout dev
   commit id: "dev baseline"

   branch feature/auth-cookies
   checkout feature/auth-cookies
   commit id: "httpOnly cookies"
   commit id: "tests"
   checkout dev
   merge feature/auth-cookies id: "PR #1 merged"

   branch feature/captcha
   checkout feature/captcha
   commit id: "Turnstile integration"
   checkout dev
   merge feature/captcha id: "PR #2 merged"

   branch hotfix/csrf-fix
   checkout hotfix/csrf-fix
   commit id: "sameSite: Strict"
   checkout main
   merge hotfix/csrf-fix id: "hotfix to prod" tag: "v1.0.1"
   checkout dev
   merge hotfix/csrf-fix id: "sync hotfix"

   checkout main
   merge dev id: "Release v1.1" tag: "v1.1.0"
```

**Правила веток:**
- `main` — только из `dev` через PR, после зелёного CI. Protected. Прямые пуши запрещены.
- `dev` — интеграционная ветка. CI обязателен на каждый PR.
- `feature/*` — одна задача / одна Story. Живёт не дольше 3 дней.
- `hotfix/*` — критические фиксы в прод. Мержится в `main` И `dev`.
- `release/*` — при необходимости подготовки релиза (feature freeze).

---

### 2.2 CI/CD Pipeline — полная схема

```mermaid
flowchart TD
    DEV["👨‍💻 Разработчик\npush feature branch"]
    PR["📬 Pull Request\nв dev или main"]

    subgraph CI["⚙️ CI Pipeline (автоматически на каждый PR)"]
        LINT["🔍 Lint & Format\nESLint · Prettier"]
        UNIT["🧪 Unit Tests\nJest — backend + frontend\nПорог: ≥ 80% coverage"]
        INT["🔗 Integration Tests\nAPI-тесты с тестовой БД\nПроверка бизнес-логики"]
        SAST["🛡️ SAST Security Scan\nSemgrep · CodeQL\nПроверка injection, XSS, IDOR"]
        SNYK["📦 Dependency Audit\nSnyk / npm audit\nKnown CVE scan"]
        BUILD["🏗️ Docker Build\nMuliti-stage build\nImage push → registry"]
    end

    subgraph STAGING["🔵 Staging Deploy (автоматически после merge в dev)"]
        DEPLOY_STG["🚀 Deploy to Staging\nDocker Compose / K8s"]
        SMOKE["💨 Smoke Tests (E2E)\nCypress / Playwright\n─────────────────\n✓ Регистрация\n✓ Подтверждение email\n✓ Депозит крипто\n✓ Открытие кейса\n✓ Вывод средств\n✓ Реферальная ссылка"]
        PERF["📊 Performance Check\nLighthouse CI\nBaseline: TTFB < 300ms"]
        HEADERS["🔒 Security Headers Check\nCSP, X-Frame, HSTS\nHSTS, Referrer-Policy"]
    end

    subgraph GATE["🔐 Production Gate"]
        APPROVAL["👍 Manual Approval\nPO / Tech Lead\nв Slack / GitHub"]
    end

    subgraph PROD["🟢 Production Deploy"]
        BLUE_GREEN["🔄 Blue-Green Deploy\nTraffic switch 10% → 50% → 100%"]
        HEALTH["❤️ Health Check\n/api/health · DB ping\nPSP ping"]
        SMOKE_PROD["💨 Prod Smoke Tests\nКритичный путь\nна реальной среде"]
        ROLLBACK["⏪ Auto-Rollback\nпри failed health check\nили ошибке smoke tests"]
    end

    subgraph MONITOR["📡 Мониторинг (always-on)"]
        ALERTS["🚨 Алерты\n─────────────────\n• GGR drop > 20% за час\n• Chargeback Rate > 1%\n• PSP Approval Rate < 70%\n• Error Rate > 2%\n• P95 latency > 1s"]
    end

    DEV --> PR
    PR --> LINT
    LINT --> UNIT
    UNIT --> INT
    INT --> SAST
    SAST --> SNYK
    SNYK --> BUILD

    BUILD -->|"❌ Любой шаг упал → PR заблокирован"| PR
    BUILD -->|"✅ Все зелёные → merge доступен"| DEPLOY_STG

    DEPLOY_STG --> SMOKE
    SMOKE --> PERF
    PERF --> HEADERS
    HEADERS --> APPROVAL

    APPROVAL --> BLUE_GREEN
    BLUE_GREEN --> HEALTH
    HEALTH -->|OK| SMOKE_PROD
    HEALTH -->|FAIL| ROLLBACK
    SMOKE_PROD -->|OK| MONITOR
    SMOKE_PROD -->|FAIL| ROLLBACK
```

---

### 2.3 Стратегия тестирования

```
Пирамида тестов:
                  ▲
                 /E\      E2E (Cypress/Playwright)
                /   \     → Критичные пользовательские сценарии
               /─────\    → Запускаются на staging после каждого деплоя
              /  INT  \   Integration Tests (Jest + Supertest)
             /─────────\  → API-эндпоинты с тестовой БД
            /   UNIT    \ → Бизнес-логика: вейджер, бонусы, рефералы, RTP
           /─────────────\
```

| Тип | Инструмент | Запуск | Порог |
|---|---|---|---|
| Unit | Jest | каждый PR | ≥ 80% coverage новых строк |
| Integration | Jest + Supertest | каждый PR | все тесты зелёные |
| E2E smoke | Cypress / Playwright | после деплоя на staging | 100% критичных сценариев |
| SAST | Semgrep / CodeQL | каждый PR | 0 HIGH/CRITICAL |
| Dependency audit | Snyk | каждый PR + ежедневно | 0 CRITICAL CVE |
| DAST | OWASP ZAP | еженедельно (scheduled) | ручная проверка отчёта |
| Load test | k6 | еженедельно (scheduled) | P95 < 500ms при 500 RPS |

---

### 2.4 Окружения

| Окружение | Назначение | Деплой | База данных |
|---|---|---|---|
| **local** | Разработка | `docker compose up` | Локальная PostgreSQL |
| **dev** | Интеграция после merge в dev | Автоматически | Shared test DB |
| **staging** | QA, PO acceptance, smoke tests | Автоматически после CI | Staging DB (prod-like data) |
| **production** | Реальные пользователи | Manual approval → auto | Production PostgreSQL |

---

### 2.5 Scheduled Jobs (фоновые проверки)

| Job | Расписание | Инструмент | Действие при сбое |
|---|---|---|---|
| Dependency audit | Ежедневно 09:00 | Snyk | Тикет в Jira, уведомление в Slack |
| DAST scan | Еженедельно, пн | OWASP ZAP | Ручная проверка отчёта PM + SecEng |
| Load test | Еженедельно, пт | k6 | Сравнение с baseline, алерт при деградации |
| DB backup check | Ежедневно | pg_dump verify | PagerDuty alert |
| SSL cert expiry | Ежедневно | certbot check | Алерт за 30/7/1 день до истечения |
| AML transaction scan | Ежедневно | Chainalysis API | Автоматическая заморозка + тикет |

---

### 2.6 Мониторинг & Алерты

```mermaid
flowchart LR
    subgraph METRICS["📊 Метрики продукта"]
        M1["GGR / NGR\n(hourly)"]
        M2["FTD CR\n(reg→dep)"]
        M3["Chargeback Rate"]
        M4["PSP Approval Rate"]
        M5["D7 Retention\n(когортно)"]
    end

    subgraph INFRA["🖥️ Инфраструктура"]
        I1["Error Rate\n(Sentry)"]
        I2["P95 Latency\n(Datadog)"]
        I3["Uptime\n(Pingdom)"]
        I4["PSP latency\n(custom probe)"]
    end

    subgraph ALERTS["🚨 Алерты → Slack #alerts"]
        A1["L1 (Дежурный)\nнемедленно"]
        A2["L2 (PO + Tech Lead)\nв рабочее время"]
    end

    M1 -->|"Падение > 20%/ч"| A1
    M3 -->|"> 1%"| A2
    M4 -->|"< 70%"| A1
    I1 -->|"> 2% error rate"| A1
    I2 -->|"> 1s P95"| A2
    I3 -->|"Downtime"| A1
    I4 -->|"PSP latency > 5s"| A1
    M2 -->|"CR < 25% за день"| A2
```

---

*Документ разработан как часть продуктового аудита CYBERBOX. Отражает best practices CI/CD и процессного управления для iGaming-продуктов.*
