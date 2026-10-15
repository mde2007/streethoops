# StreetHoops: проектирование REST API (пункт 1)

Базовый URL: `https://api.streethoops.example.com/v1`
Формат: JSON (`application/json`), загрузка фото: `multipart/form-data`.
Аутентификация: `Authorization: Bearer <JWT>`.
Роли: **Гость** (без токена), **Игрок** (`player`), **Админ** (`admin`).

Сущности (по ER-диаграмме): `USER`, `Court`, `Check_IN`, `Court_leaderboard`, `Trainer`, `Trainer_Rating`.
Чат площадки исключён из ТЗ, поэтому ресурса сообщений нет.

## 1. Аутентификация

| Метод | URL | Доступ | Тело запроса | Ответ |
|---|---|---|---|---|
| POST | `/auth/register` | Гость | `name`, `email` и/или `phone`, `password` | 201 `UserPublic`; 400, 409 (email/телефон заняты) |
| POST | `/auth/login` | Гость | `login` (email или телефон), `password` | 200 `{access_token, token_type, expires_in, user}`; 400, 401 |

## 2. Пользователь

| Метод | URL | Доступ | Тело запроса | Ответ |
|---|---|---|---|---|
| GET | `/users/me` | Игрок | нет | 200 `UserPublic`; 401 |
| PATCH | `/users/me` | Игрок | `name`, `email`, `phone`, `password` (любые) | 200 `UserPublic`; 400, 401, 409 |
| GET | `/users/me/checkins?date=&page=&limit=` | Игрок | нет | 200 список отметок; 401 |

## 3. Площадки (ФТ-01, User Story 2)

| Метод | URL | Доступ | Тело запроса / параметры | Ответ |
|---|---|---|---|---|
| GET | `/courts` | Гость | query: `surface_type` (asphalt/rubber/parquet), `is_paid`, `has_lighting`, `lat`, `lng`, `radius_m`, `moderation_status` (только admin), `page`, `limit` | 200 список `CourtPublic` (пустой список = «ничего не найдено»); 400 |
| POST | `/courts` | Игрок | `address`, `latitude`, `longitude`, `surface_type`, `has_lighting`, `is_paid` | 201 `CourtPublic` со статусом `pending`; 400, 401 |
| GET | `/courts/{courtId}` | Гость | нет | 200 `CourtPublic` (+ `expected_players_today`); 404 |
| PATCH | `/courts/{courtId}` | Автор / Админ | любые поля из POST | 200; 400, 401, 403, 404 |
| DELETE | `/courts/{courtId}` | Админ | нет | 204; 401, 403, 404 |
| POST | `/courts/{courtId}/photo` | Автор / Админ | form-data: `file` (jpg/png, до 5 МБ) | 200 `CourtPublic`; 400, 401, 403, 404 |
| PATCH | `/courts/{courtId}/moderation` | Админ | `moderation_status` (pending/approved/rejected) | 200; 400, 401, 403, 404 |

## 4. Отметки «Буду сегодня» / «Не смогу» (ФТ-02, User Story 1)

| Метод | URL | Доступ | Тело запроса | Ответ |
|---|---|---|---|---|
| GET | `/courts/{courtId}/checkins?date=` | Гость | нет | 200 `{court_id, checkin_date, expected_players}`; 404 |
| POST | `/courts/{courtId}/checkins` | Игрок | `checkin_date` (необязательно, по умолчанию сегодня) | 201 `CheckinPublic` со статусом «Буду сегодня»; 401, 404, 409 (уже отмечен) |
| PATCH | `/checkins/{checkinId}` | Владелец | `status`: «Буду сегодня» / «Не смогу» | 200 `CheckinPublic`; 400, 401, 403, 404 |

Гость нажимает «Буду сегодня» и получает 401, клиент открывает окно авторизации (use case 1.3).

## 5. Рейтинг игроков площадки

| Метод | URL | Доступ | Тело запроса | Ответ |
|---|---|---|---|---|
| GET | `/courts/{courtId}/leaderboard?page=&limit=` | Гость | нет | 200 список `{user_id, user_name, visits_count, rank, updated_at}`; 404 |

Таблица пересчитывается на сервере по отметкам, клиент её не изменяет.

## 6. Тренеры (новая вкладка «Тренировки»)

| Метод | URL | Доступ | Тело запроса / параметры | Ответ |
|---|---|---|---|---|
| GET | `/trainers` | Гость | query: `specialization`, `min_rating`, `max_price`, `sort` (rating_desc, price_asc, price_desc, experience_desc), `page`, `limit` | 200 список `TrainerPublic` (только `is_active=true`); 400 |
| POST | `/trainers` | Игрок | `display_name`, `about`, `specialization`, `experience_years`, `price_per_hour`, `contact_info`, `photo_url` | 201 `TrainerPublic`; 400, 401, 409 (профиль уже есть) |
| GET | `/trainers/{trainerId}` | Гость | нет | 200 `TrainerPublic` (+ `average_rating`); 404 |
| PATCH | `/trainers/{trainerId}` | Владелец / Админ | любые поля из POST, `is_active` | 200; 400, 401, 403, 404 |
| DELETE | `/trainers/{trainerId}` | Владелец / Админ | нет | 204; 401, 403, 404 |
| POST | `/trainers/{trainerId}/photo` | Владелец | form-data: `file` | 200 `TrainerPublic`; 400, 401, 403, 404 |

## 7. Оценки тренеров

| Метод | URL | Доступ | Тело запроса | Ответ |
|---|---|---|---|---|
| GET | `/trainers/{trainerId}/ratings?page=&limit=` | Гость | нет | 200 список оценок; 404 |
| POST | `/trainers/{trainerId}/ratings` | Игрок | `score` (1..5), `comment` | 201 `TrainerRatingPublic`; 400, 401, 403 (свой профиль), 404, 409 (уже оценил) |
| DELETE | `/trainers/{trainerId}/ratings/{ratingId}` | Автор / Админ | нет | 204; 401, 403, 404 |

После создания или удаления оценки сервер пересчитывает `average_rating` тренера.

## Единый формат ошибки

```json
{ "code": "validation_error", "message": "Поле score должно быть от 1 до 5" }
```

Коды: 400 (валидация), 401 (нет/неверный токен), 403 (нет прав), 404 (не найдено), 409 (дубликат), 500 (ошибка сервера).
