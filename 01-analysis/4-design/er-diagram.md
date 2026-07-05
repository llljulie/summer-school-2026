# ER-диаграмма — «Скалодром Вертикаль»

> **Область:** каноническая модель данных клиентского приложения и клиентского API
> **Источник истины:** контракт API (R-015); бэкенд — black-box, маппинг внутренних моделей вне скоупа 
> **Связанные артефакты:** [brief-climbing.md](./brief-climbing.md), [functional-requirements.md](./functional-requirements.md), [questions.md](./questions.md)

---

## Диаграмма сущностей и связей

```mermaid
erDiagram
    CLIMBING_GYM ||--o{ TRAINING_SLOT : "публикует"
    INSTRUCTOR ||--o{ TRAINING_SLOT : "ведёт"
    TRAINING_FORMAT ||--o{ TRAINING_SLOT : "имеет формат"
    TRAINING_ZONE ||--o{ TRAINING_SLOT : "проводится в зоне"

    CLIENT ||--o{ BOOKING : "оформляет"
    TRAINING_SLOT ||--o{ BOOKING : "содержит брони"
    BOOKING ||--o| INSTRUCTOR_RATING : "может иметь оценку"
    CLIENT ||--o{ INSTRUCTOR_RATING : "ставит"

    INSTRUCTOR ||--o{ INSTRUCTOR_RATING : "получает"
    INSTRUCTOR ||--|| INSTRUCTOR_STATS : "агрегирует"

    CLIENT ||--|| LOYALTY_ACCOUNT : "имеет"
    LOYALTY_ACCOUNT ||--o{ LOYALTY_DISCOUNT_HISTORY : "фиксирует скидки"

    CLIENT ||--|| PENALTY_ACCOUNT : "имеет"
    PENALTY_ACCOUNT ||--o{ PENALTY_EVENT : "содержит события"

    CLIENT ||--o{ DEVICE_TOKEN : "регистрирует"
    CLIENT ||--o{ NOTIFICATION_LOG : "получает"

    BOOKING ||--o{ NOTIFICATION_LOG : "инициирует"
    TRAINING_SLOT ||--o{ SLOT_CANCELLATION : "может быть отменён"

    BOOKING }o--|| EQUIPMENT_CHOICE : "включает выбор"
    TRAINING_SLOT ||--o| RENTAL_STOCK : "имеет прокатный фонд"

    CLIMBING_GYM {
        uuid id PK
        string name
        string address
        string phone
        string working_hours
    }

    INSTRUCTOR {
        uuid id PK
        string full_name
        boolean is_active
    }

    INSTRUCTOR_STATS {
        uuid instructor_id PK_FK
        decimal avg_rating "1.00–5.00"
        int ratings_count
    }

    TRAINING_FORMAT {
        uuid id PK
        string code "beginner_bouldering | advanced_rope"
        string title
        int max_capacity "8 для новичковых, 16 для остальных"
        int duration_minutes "90"
    }

    TRAINING_ZONE {
        uuid id PK
        string code "bouldering | rope"
        string title
    }

    TRAINING_SLOT {
        uuid id PK
        uuid gym_id FK
        uuid instructor_id FK
        uuid format_id FK
        uuid zone_id FK
        datetime starts_at
        datetime ends_at
        int capacity
        int booked_count
        int free_places "вычисляемое"
        decimal base_price
        decimal rental_price
        boolean rental_available
        string slot_status "open | full | cancelled_by_club"
        string cancellation_reason "nullable"
    }

    RENTAL_STOCK {
        uuid slot_id PK_FK
        int total_units
        int available_units
        boolean rental_available "available_units > 0"
    }

    CLIENT {
        uuid id PK
        string phone E164 "уникальный"
        string display_name "nullable"
        datetime created_at
        datetime last_login_at
    }

    BOOKING {
        uuid id PK
        uuid client_id FK
        uuid slot_id FK
        string equipment_type "own | rental"
        decimal training_price
        decimal rental_price_applied
        decimal loyalty_discount
        decimal total_price
        string status "active | cancelled_by_client | cancelled_by_club | completed | no_show"
        boolean attended "nullable до завершения"
        boolean penalty_applied "при отмене ≤10 мин"
        datetime booked_at
        datetime cancelled_at "nullable"
        string cancellation_reason "nullable, для cancelled_by_club"
    }

    EQUIPMENT_CHOICE {
        string type PK "own | rental"
        string label
    }

    INSTRUCTOR_RATING {
        uuid id PK
        uuid booking_id FK "уникальный"
        uuid client_id FK
        uuid instructor_id FK
        int score "1–5"
        datetime rated_at
        datetime expires_at "rated_at + 24h окно ввода"
    }

    LOYALTY_ACCOUNT {
        uuid client_id PK_FK
        int completed_visits_count
        int visits_to_next_discount "0–9"
        int discount_percent "50 при 10-м посещении"
        datetime updated_at
    }

    LOYALTY_DISCOUNT_HISTORY {
        uuid id PK
        uuid client_id FK
        uuid booking_id FK
        int visit_number
        decimal discount_amount
        datetime applied_at
    }

    PENALTY_ACCOUNT {
        uuid client_id PK_FK
        int penalty_points "0–3+"
        int visits_to_reset "5 успешных подряд"
        boolean is_booking_blocked
        datetime blocked_until "nullable"
    }

    PENALTY_EVENT {
        uuid id PK
        uuid client_id FK
        uuid booking_id FK
        string event_type "late_cancel | reset_after_visits"
        int points_delta
        datetime created_at
    }

    DEVICE_TOKEN {
        uuid id PK
        uuid client_id FK
        string fcm_token
        string platform "android"
        boolean push_enabled
        datetime registered_at
    }

    NOTIFICATION_LOG {
        uuid id PK
        uuid client_id FK
        uuid booking_id FK "nullable"
        string type "booking_confirmed | reminder_24h | reminder_2h | club_cancelled | rate_instructor"
        string channel "push | sms"
        string payload
        datetime scheduled_at
        datetime sent_at "nullable"
        string delivery_status "sent | failed | skipped"
    }

    SLOT_CANCELLATION {
        uuid id PK
        uuid slot_id FK
        string reason "maintenance | instructor_illness | force_majeure"
        string reason_text
        uuid cancelled_by_admin_id "вне клиентского API"
        datetime cancelled_at
    }
```

---

## Легенда кардинальностей

| Обозначение | Значение |
|-------------|----------|
| `\|\|--o{` | Один ко многим (1:N), обязательный «один» |
| `\|\|--\| \|` | Один к одному (1:1) |
| `\}o--\|\|` | Многие к одному (N:1), опциональный «многие» |
| `\|\|--o\|` | Один к нулю или одному (1:0..1) |

---

## Описание сущностей

### CLIMBING_GYM — Скалодром

Стационарные данные площадки. Отдаются в API (адрес — в карточке слота и SMS-напоминаниях).

| Атрибут | Обязательность | Примечание |
|---------|----------------|------------|
| `address` | да | FR-005, FR-017, FR-018 |
| Остальные | опционально | Для справочного блока в приложении |

---

### INSTRUCTOR — Инструктор

Ведёт групповые тренировки. Не редактируется клиентским приложением.

| Связь | Описание |
|-------|----------|
| → TRAINING_SLOT | Один инструктор — много слотов |
| → INSTRUCTOR_STATS | Агрегат рейтинга (денормализация для списка слотов) |
| → INSTRUCTOR_RATING | Много оценок от клиентов |

---

### INSTRUCTOR_STATS — Статистика инструктора

Денормализованная сущность для отображения в карточке слота (FR-005, FR-019).

| Атрибут | Правило |
|---------|---------|
| `avg_rating` | Среднее по INSTRUCTOR_RATING.score |
| `ratings_count` | Количество оценок; при 0 — в UI показывать «Новый» или «—» (TBD) |

---

### TRAINING_FORMAT — Формат тренировки

Бизнес-правило из брифа: новичковые — max 8 мест, остальные — max 16.

Примеры: «Болдеринг с инструктажем (новички)», «Трассы с верёвкой (опытные)».

---

### TRAINING_ZONE — Зона скалодрома

Физическая зона: болдеринг, трассы. Может быть закрыта на профилактику → SLOT_CANCELLATION.

---

### TRAINING_SLOT — Слот (тренировка)

Центральная сущность расписания. Источник — бэкенд (read-only для клиента).

| Атрибут | Источник / правило |
|---------|-------------------|
| `free_places` | `capacity - booked_count`; лимит из TRAINING_FORMAT |
| `rental_available` | Из RENTAL_STOCK или поле API (questions.md §199) |
| `slot_status = cancelled_by_club` | Повторная запись запрещена (FR-016, R-008) |
| Горизонт по умолчанию | 7 дней (FR-002, R-027) |

---

### RENTAL_STOCK — Прокатный фонд на слот

Отражает доступность проката на конкретный слот.

| Сценарий | Поведение в приложении |
|----------|------------------------|
| `rental_available = false` | Кнопка «Прокат» неактивна; запись только «со своим» (FR-007, FR-024) |
| `rental_available = true` | Цена проката добавляется к итогу (questions.md §4) |

---

### CLIENT — Клиент

Регистрация по телефону + SMS (FR-001). Уникальный `phone` в формате E.164.

---

### BOOKING — Бронирование

Связь CLIENT ↔ TRAINING_SLOT. Атомарность создания — на бэкенде (BR-002, NFR-004).

**Статусы (`status`):**

| Значение | Описание | FR / UC |
|----------|----------|---------|
| `active` | Предстоящая запись | FR-012 |
| `cancelled_by_client` | Отменена клиентом | FR-010, UC-005 |
| `cancelled_by_club` | Отменена скалодромом | FR-014, UC-006 |
| `completed` | Тренировка завершена, клиент пришёл | FR-012 |
| `no_show` | Неявка | FR-012 (кто выставляет — бэкенд) |

**Ценообразование:**

```
total_price = training_price + rental_price_applied - loyalty_discount
```

---

### EQUIPMENT_CHOICE — Выбор снаряжения

Справочник значений для поля `BOOKING.equipment_type`. Обязателен при каждой записи (FR-007).

| type | label |
|------|-------|
| `own` | Со своим снаряжением |
| `rental` | Нужен прокат |

---

### INSTRUCTOR_RATING — Оценка инструктора

| Правило | Ссылка |
|---------|--------|
| Окно ввода: 24 ч после окончания слота | FR-019 |
| Только при `booking.attended = true` | UC-008 |
| Одна оценка на бронь (`booking_id` уникален) | FR-019 |
| Push-приглашение через 1 ч после окончания | questions.md §7 |

---

### LOYALTY_ACCOUNT — Счёт лояльности

Механика MVP (questions.md §11, FR-020):

| Правило | Значение |
|---------|----------|
| Скидка | 50% на каждое 10-е **завершённое** посещение (`attended = true`) |
| Счётчик в профиле | `visits_to_next_discount` = «До скидки осталось X» |
| После скидки | Счётчик продолжается (11-е → 1-е к следующей скидке) |
| При записи | Скидка применяется до подтверждения, не суммируется с другими |

> **TBD:** точный момент инкремента `completed_visits_count` — после `completed` или при бронировании 10-й. Рекомендация: после `attended = true`.

---

### LOYALTY_DISCOUNT_HISTORY — История скидок

Журнал применённых скидок (Экран 11 в `screen.md`). Опционально для MVP, если бэкенд отдаёт историю.

---

### PENALTY_ACCOUNT — Штрафной счёт

| Правило | Значение |
|---------|----------|
| Начисление | При отмене ≤ 10 мин до начала (FR-010, FR-011) |
| Блокировка | 3 штрафных балла → запись заблокирована на 24 ч |
| Сброс | 5 успешных посещений подряд без штрафов |
| Отмена под блокировкой | Существующие брони отменять можно (FR-010) |

---

### PENALTY_EVENT — Событие штрафа

Аудит начислений и сбросов. Управляется бэкендом; клиент только отображает (FR-028).

---

### DEVICE_TOKEN — Push-токен устройства

FCM-токен Android-устройства (FR-034, NFR-011). Передаётся на бэкенд при входе / смене токена.

---

### NOTIFICATION_LOG — Журнал уведомлений

| type | channel | Когда |
|------|---------|-------|
| `booking_confirmed` | push | После успешной записи (FR-021) |
| `reminder_24h` | sms | За 24 ч (FR-017) |
| `reminder_2h` | sms | За 2 ч (FR-018) |
| `club_cancelled` | push | Отмена скалодромом (FR-015) |
| `rate_instructor` | push | Через 1 ч после окончания (questions.md §7) |

Критические (`club_cancelled`) доставляются независимо от настройки push в приложении (FR-029), с fallback-модалкой (FR-015).

---

### SLOT_CANCELLATION — Отмена слота скалодромом

Инициируется администратором во внешней системе. Каскадно обновляет все связанные BOOKING → `cancelled_by_club` (UC-006).

---

## Ключевые связи (сводка)

```
Скалодром 1──N Слот N──1 Инструктор
Слот N──1 Формат, Зона
Клиент 1──N Бронирование N──1 Слот
Бронирование 1──0..1 Оценка
Клиент 1──1 Лояльность, 1──1 Штрафной счёт
Слот 1──0..1 Прокатный фонд
Слот 1──0..N Отмена скалодромом (история)
```

---

## Сущности вне клиентского API (только на бэкенде)

| Сущность / операция | Примечание |
|---------------------|------------|
| Администратор | Создаёт расписание, отменяет слоты (R-028) |
| SMS-шлюз | Отправка напоминаний (NFR-011) |
| FCM | Доставка push |
| Планировщик (cron) | Триггеры 24 ч / 2 ч / 1 ч |
| Внутренняя модель транзакций | Атомарность бронирования (R-004) |

---

## Индексы и ограничения (рекомендации для API/БД)

| Сущность | Ограничение |
|----------|-------------|
| CLIENT.phone | UNIQUE |
| BOOKING (client_id, slot_id) | UNIQUE при status = active |
| INSTRUCTOR_RATING.booking_id | UNIQUE |
| TRAINING_SLOT (starts_at, slot_status) | INDEX для выборки 7 дней |
| BOOKING (client_id, status) | INDEX для «Мои записи» |
| PENALTY_ACCOUNT (client_id) | CHECK penalty_points >= 0 |

---

## Открытые вопросы модели (TBD)

1. **Посещение для лояльности:** инкремент при `completed` + `attended = true` или при создании 10-й брони?
2. **INSTRUCTOR_STATS:** обновление в реальном времени или при запросе слотов?
3. **LOYALTY_DISCOUNT_HISTORY:** обязательна в MVP или только текущий счётчик?
4. **Поле API:** `rental_available` на слоте vs отдельная сущность RENTAL_STOCK?
5. **no_show:** автоматическая простановка бэкендом по таймауту после `ends_at`?

---

> **Версия:** 1.0  
> **Дата:** 05.07.2026  
> **Статус:** черновик для согласования с контрактом API
