# Руководство по TurtleLogs API

Данное руководство содержит описание публичных GET-методов API проекта TurtleLogs для получения игровых данных, информации о персонажах, гильдиях и статистике рейдов.

## 1. Аутентификация и заголовки

Большинство методов API требуют аутентификации или специальных заголовков для корректной работы.

### API Токены
Для доступа к защищенным методам используется **API Token**.

*   **Где получить:** В личном кабинете на сайте в разделе **API Tokens**.
    1. Укажите название (например, "My App").
    2. Выберите дату окончания действия.
    3. После генерации сохраните токен (SHA3 хэш).
*   **Передача:** Заголовок `X-Authorization: <ваш_токен>`.

### Дополнительные заголовки
*   `X-Language`: Код языка (например, `en`, `ru`). По умолчанию используется ID 1 (обычно английский).
*   `X-Expansion`: ID дополнения (0 — Vanilla, 1 — TBC, 2 — WotLK). Обязателен для многих методов в Data API.

### Уровни доступа
1.  **Публичный:** Токен не требуется.
2.  **Пользовательский:** Требуется заголовок `X-Authorization`.
3.  **Владелец сервера (ServerOwner):** Требуется токен пользователя, владеющего игровым сервером.

---

## 2. Группы методов

### <a name="data-api"></a> Data API (Справочные данные)
Базовый путь: `/API/data`

Методы для получения общей игровой информации. В основном публичные.

| Метод | Описание | Доступ |
|:---|:---|:---|
| `GET /server` | Список всех игровых миров | Публичный |
| `GET /server/<id>` | Детальная информация о сервере | Публичный |
| `GET /item/<exp_id>/<id>` | Данные о предмете | Публичный |
| `GET /spell/<exp_id>/<id>` | Данные о заклинании | Публичный |
| `GET /race/localized` | Список рас на языке из `X-Language` | Публичный |
| `GET /hero_class/localized` | Список классов на языке из `X-Language` | Публичный |
| `GET /map` | Список всех игровых карт | Публичный |

**Пример (Список серверов):**
```bash
curl -X GET "https://www.turtlogs.com/API/data/server"
```

### <a name="armory-api"></a> Armory API (Оружейная)
Базовый путь: `/API/armory`

| Метод | Описание | Доступ |
|:---|:---|:---|
| `GET /character/<id>` | Основные данные персонажа | Публичный |
| `GET /character/by_name/<name>` | Поиск по имени персонажа | Публичный |
| `GET /character/by_uid/<uid>` | Получение по внутреннему UID | **ServerOwner** |
| `GET /character_viewer/<server>/<name>` | Экипировка и таланты | Публичный |
| `GET /guild/<id>` | Информация о гильдии | Публичный |
| `GET /guild_view/<server>/<name>` | Детальный вид гильдии | Публичный |
| `GET /guild_roster/<guild_id>` | Список участников гильдии | Публичный |

**Пример (Данные персонажа):**
```bash
curl -X GET "https://www.turtlogs.com/API/armory/character_viewer/TurtleWoW/Thrall" \
     -H "X-Language: ru"
```

### <a name="tooltip-api"></a> Tooltip API (Подсказки)
Базовый путь: `/API/tooltip`

Используется для получения HTML/JSON данных для всплывающих подсказок.

*   `GET /item/<exp_id>/<id>` — Тултип предмета.
*   `GET /spell/<exp_id>/<id>` — Тултип заклинания.
*   `GET /character/<id>` — Тултип персонажа (краткая сводка).

**Пример (Тултип предмета):**
```bash
curl -X GET "https://www.turtlogs.com/API/tooltip/item/0/19019" \
     -H "X-Language: ru"
```

### <a name="instance-api"></a> Instance API (Логи и Рейтинги)
Базовый путь: `/API/instance`

| Метод | Описание | Доступ |
|:---|:---|:---|
| `GET /export/meta/<id>` | Мета-данные рейда (карта, время) | Публичный |
| `GET /export/participants/<id>` | Участники рейда | Публичный |
| `GET /export/attempts/<id>` | Список попыток на боссах | Публичный |
| `GET /ranking/dps` | Рейтинг лучших игроков по урону | Публичный |
| `GET /speed_run` | Рейтинг гильдий по скорости прохождения | Публичный |
| `GET /speed_kill` | Рейтинг по скорости убийства боссов | Публичный |

**Пример (Рейтинг DPS):**
```bash
curl -X GET "https://www.turtlogs.com/API/instance/ranking/dps/by_season/1"
```

---

## 7. Техническая реализация (для разработчиков)

Ниже указаны файлы и участки кода, отвечающие за логику авторизации и проверку ключей:

### Бэкенд (Rust)
*   **Логика проверки токена:** `Backend/src/modules/account/tools/token.rs` — метод `validate_token`. Здесь происходит хеширование входящего ключа и сверка с базой данных.
*   **Гарды (Guards):**
    *   `Authenticate` (обязательный токен): `Backend/src/modules/account/guard/authenticate.rs`.
    *   `CurrentUser` (опциональный токен): `Backend/src/modules/account/guard/current_user.rs`.
    *   `ServerOwner` (проверка прав владельца): `Backend/src/modules/account/guard/server_owner.rs`.
*   **Генерация ключа:** `Backend/src/modules/account/tools/token.rs` — метод `create_token`. Использует `sha3::hash` от почты, пароля и случайной соли.

### Фронтенд (TypeScript/Angular)
*   **Интерфейс управления токенами:** `Webclient/src/app/module/account/module/api_tokens/component/api_tokens/`
*   **Сервис для работы с API токенов:** `Webclient/src/app/module/account/module/api_tokens/service/api_tokens.ts`

---
*Примечание: Если метод возвращает 401 Unauthorized, убедитесь, что вы передаете корректный `X-Authorization` заголовок и у вас достаточно прав (например, вы владелец сервера для соответствующих методов).*
