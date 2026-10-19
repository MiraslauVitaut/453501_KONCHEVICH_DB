# Даталогическая модель БД

Проект: **Система бронирования (отели и билеты)** · Автор: Кончевич Мирослав Валерьевич, группа 453501
СУБД: MySQL 8.0.16+ (нужна для работы ограничений `CHECK`)

Модель развивает инфологическую схему из лабораторной №1: уточнены типы данных, ключи и ограничения, а таблицы приведены к третьей нормальной форме (подробности и доказательство — в `normalization.md`).
Диаграмма: `datalogical_diagram.png` (редактируемая версия — `datalogical_diagram.drawio`).

Всего таблиц: **19** (18 сущностей + 1 связующая таблица `RoomAmenities`).

## 1. Соглашения

- Первичный ключ — суррогатный `id INT AUTO_INCREMENT` (в `ActionLogs` — `BIGINT`, т.к. журнал растёт быстрее остальных таблиц). Исключение: `RoomAmenities` — составной ключ из двух внешних ключей.
- Денежные значения — `DECIMAL(10,2)`, даты и время — `DATE` / `DATETIME`, логические — `BOOLEAN`.
- Перечисления (статусы, типы) хранятся как `VARCHAR` с ограничением `CHECK (... IN (...))`.
- Внешние ключи: `ON UPDATE RESTRICT` (суррогатные ключи не меняются), `ON DELETE` — см. раздел 4.
- «Мягкое» удаление пользователей делается через `Users.is_active`, поэтому история бронирований, билетов и журнал действий сохраняются.

## 2. Что изменилось по сравнению с лабораторной №1

| Изменение | Причина |
|---|---|
| Новая таблица **Cities**; `Hotels.city`, `Routes.origin_city`, `Routes.destination_city` заменены на `city_id`, `origin_city_id`, `destination_city_id` | убрано дублирование названий городов (аномалии обновления и разные написания одного города) |
| Новая таблица **Vehicles**; в `Trips` вместо `vehicle_number` и `seats_total` — `vehicle_id` | зависимость «номер транспорта → число мест» нарушала 3НФ (транзитивная зависимость) |
| `Rooms.capacity` перенесена в `RoomTypes.capacity` | вместимость определяется типом номера (транзитивная зависимость `room_type_id → capacity`) |
| `Payments` ссылается на бронь **или** на билет (`booking_id`, `ticket_id`, ровно один заполнен) | в лабораторной №1 у билетов не было оплаты |
| `Rooms.is_available` переименовано в `Rooms.is_active` | флаг означает «номер в эксплуатации»; доступность на даты вычисляется по `BookingRooms` |
| `Carriers.name` стало уникальным | один перевозчик — одна запись |

## 3. Описание таблиц

### 3.1. Roles — Роли

Справочник ролей системы (customer, manager, admin).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| name | VARCHAR(50) |  | NOT NULL, UNIQUE |
| description | VARCHAR(255) |  | NULL |

### 3.2. Users — Пользователи

Учётные записи клиентов, менеджеров и администраторов.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| email | VARCHAR(255) |  | NOT NULL, UNIQUE |
| password_hash | VARCHAR(255) |  | NOT NULL |
| role_id | INT | FK | NOT NULL, FK → Roles.id |
| is_active | BOOLEAN |  | NOT NULL, DEFAULT TRUE |
| created_at | DATETIME |  | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

### 3.3. UserProfiles — Профили пользователей

Персональные данные пользователя (связь 1:1 с Users).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| user_id | INT | FK | NOT NULL, UNIQUE, FK → Users.id |
| full_name | VARCHAR(150) |  | NOT NULL |
| birth_date | DATE |  | NULL |
| passport_number | VARCHAR(20) |  | NULL |
| phone | VARCHAR(20) |  | NULL |
| address | VARCHAR(255) |  | NULL |

### 3.4. ActionLogs — Журнал действий

Журналирование действий пользователей.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | BIGINT | PK | AUTO_INCREMENT |
| user_id | INT | FK | NOT NULL, FK → Users.id |
| action_type | VARCHAR(50) |  | NOT NULL |
| description | TEXT |  | NULL |
| created_at | DATETIME |  | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

### 3.5. Cities — Города

Справочник городов (новая таблица: убирает дублирование названий городов).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| name | VARCHAR(100) |  | NOT NULL |
| country | VARCHAR(100) |  | NOT NULL |

Дополнительно: `UNIQUE (name, country)`.

### 3.6. Hotels — Отели

Объекты размещения.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| city_id | INT | FK | NOT NULL, FK → Cities.id |
| name | VARCHAR(150) |  | NOT NULL |
| address | VARCHAR(255) |  | NOT NULL |
| description | TEXT |  | NULL |
| star_rating | TINYINT |  | NOT NULL, CHECK (star_rating BETWEEN 1 AND 5) |

### 3.7. RoomTypes — Типы номеров

Справочник типов номеров; вместимость определяется типом (перенесена из Rooms).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| name | VARCHAR(100) |  | NOT NULL, UNIQUE |
| description | TEXT |  | NULL |
| capacity | TINYINT |  | NOT NULL, CHECK (capacity > 0) |
| base_price | DECIMAL(10,2) |  | NOT NULL, CHECK (base_price >= 0) |

### 3.8. Rooms — Номера

Конкретные номера отелей.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| hotel_id | INT | FK | NOT NULL, FK → Hotels.id |
| room_type_id | INT | FK | NOT NULL, FK → RoomTypes.id |
| room_number | VARCHAR(10) |  | NOT NULL |
| floor | SMALLINT |  | NULL |
| price_per_night | DECIMAL(10,2) |  | NOT NULL, CHECK (price_per_night >= 0) |
| is_active | BOOLEAN |  | NOT NULL, DEFAULT TRUE |

Дополнительно: `UNIQUE (hotel_id, room_number)`.

### 3.9. Amenities — Удобства

Справочник удобств (Wi-Fi, парковка, бассейн).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| name | VARCHAR(100) |  | NOT NULL, UNIQUE |

### 3.10. RoomAmenities — Удобства номеров (связующая)

Связь М:М между Rooms и Amenities. Собственных данных не несёт.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| room_id | INT | PK, FK | NOT NULL, FK → Rooms.id; часть составного PK |
| amenity_id | INT | PK, FK | NOT NULL, FK → Amenities.id; часть составного PK |

Дополнительно: составной первичный ключ `PRIMARY KEY (room_id, amenity_id)`.

### 3.11. Bookings — Бронирования

Бронирование номеров пользователем.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| user_id | INT | FK | NOT NULL, FK → Users.id |
| status | VARCHAR(20) |  | NOT NULL, DEFAULT 'PENDING', CHECK (status IN ('PENDING','CONFIRMED','CANCELLED','COMPLETED')) |
| created_at | DATETIME |  | NOT NULL, DEFAULT CURRENT_TIMESTAMP |
| total_price | DECIMAL(10,2) |  | NOT NULL, DEFAULT 0, CHECK (total_price >= 0) |

### 3.12. BookingRooms — Позиции бронирования

Какие номера и на какие даты входят в бронирование (М:М Bookings–Rooms с данными).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| booking_id | INT | FK | NOT NULL, FK → Bookings.id |
| room_id | INT | FK | NOT NULL, FK → Rooms.id |
| check_in_date | DATE |  | NOT NULL |
| check_out_date | DATE |  | NOT NULL, CHECK (check_out_date > check_in_date) |
| price_at_booking | DECIMAL(10,2) |  | NOT NULL, CHECK (price_at_booking >= 0) |

Дополнительно: `UNIQUE (booking_id, room_id, check_in_date)`.

### 3.13. Carriers — Перевозчики

Авиакомпании, железная дорога, автобусные компании.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| name | VARCHAR(150) |  | NOT NULL, UNIQUE |
| type | VARCHAR(20) |  | NOT NULL, CHECK (type IN ('AVIA','RAIL','BUS')) |

### 3.14. Vehicles — Транспортные средства

Самолёты, поезда, автобусы (новая таблица: вынесена зависимость номер → число мест).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| registration_number | VARCHAR(20) |  | NOT NULL, UNIQUE |
| model | VARCHAR(100) |  | NULL |
| seats_total | SMALLINT |  | NOT NULL, CHECK (seats_total > 0) |

### 3.15. Routes — Маршруты

Направление перевозки конкретного перевозчика.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| carrier_id | INT | FK | NOT NULL, FK → Carriers.id |
| origin_city_id | INT | FK | NOT NULL, FK → Cities.id |
| destination_city_id | INT | FK | NOT NULL, FK → Cities.id, CHECK (destination_city_id <> origin_city_id) |
| distance_km | INT |  | NULL, CHECK (distance_km > 0) |

Дополнительно: `UNIQUE (carrier_id, origin_city_id, destination_city_id)`.

### 3.16. Trips — Рейсы

Конкретный рейс по маршруту на определённое время.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| route_id | INT | FK | NOT NULL, FK → Routes.id |
| vehicle_id | INT | FK | NOT NULL, FK → Vehicles.id |
| departure_datetime | DATETIME |  | NOT NULL |
| arrival_datetime | DATETIME |  | NOT NULL, CHECK (arrival_datetime > departure_datetime) |
| base_price | DECIMAL(10,2) |  | NOT NULL, CHECK (base_price >= 0) |

Дополнительно: `UNIQUE (vehicle_id, departure_datetime)`.

### 3.17. Tickets — Билеты

Билеты пользователей на рейсы.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| user_id | INT | FK | NOT NULL, FK → Users.id |
| trip_id | INT | FK | NOT NULL, FK → Trips.id |
| seat_number | VARCHAR(10) |  | NULL |
| price | DECIMAL(10,2) |  | NOT NULL, CHECK (price >= 0) |
| status | VARCHAR(20) |  | NOT NULL, DEFAULT 'BOOKED', CHECK (status IN ('BOOKED','CANCELLED','USED')) |
| purchased_at | DATETIME |  | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Дополнительно: `UNIQUE (trip_id, seat_number)`; при отмене билета `seat_number` обнуляется, чтобы место можно было продать повторно (в уникальном индексе допускается несколько NULL).

### 3.18. Payments — Платежи

Оплата бронирования отеля ИЛИ билета (заполнен ровно один из двух внешних ключей).

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| booking_id | INT | FK | NULL, FK → Bookings.id |
| ticket_id | INT | FK | NULL, FK → Tickets.id |
| amount | DECIMAL(10,2) |  | NOT NULL, CHECK (amount > 0) |
| payment_method | VARCHAR(30) |  | NOT NULL, CHECK (payment_method IN ('CARD','CASH','ONLINE')) |
| status | VARCHAR(20) |  | NOT NULL, DEFAULT 'PENDING', CHECK (status IN ('PENDING','SUCCESS','FAILED','REFUNDED')) |
| paid_at | DATETIME |  | NULL |

Дополнительно: `CHECK ((booking_id IS NULL) <> (ticket_id IS NULL))` — заполнен ровно один из двух внешних ключей.

### 3.19. Reviews — Отзывы

Отзывы пользователей об отелях.

| Поле | Тип | Ключ | Ограничения |
|---|---|---|---|
| id | INT | PK | AUTO_INCREMENT |
| user_id | INT | FK | NOT NULL, FK → Users.id |
| hotel_id | INT | FK | NOT NULL, FK → Hotels.id |
| rating | TINYINT |  | NOT NULL, CHECK (rating BETWEEN 1 AND 5) |
| comment | TEXT |  | NULL |
| created_at | DATETIME |  | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Дополнительно: `UNIQUE (user_id, hotel_id)`.

## 4. Внешние ключи и ссылочные действия

| Дочерняя таблица.столбец | Родительская таблица | Связь | Необязательный | ON DELETE | Обоснование |
|---|---|---|---|---|---|
| Users.role_id | Roles | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| UserProfiles.user_id | Users | 1:1 | нет | CASCADE | профиль не существует без пользователя |
| ActionLogs.user_id | Users | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Bookings.user_id | Users | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Reviews.user_id | Users | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Tickets.user_id | Users | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Hotels.city_id | Cities | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Routes.origin_city_id | Cities | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Routes.destination_city_id | Cities | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Rooms.hotel_id | Hotels | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Rooms.room_type_id | RoomTypes | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Reviews.hotel_id | Hotels | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| RoomAmenities.room_id | Rooms | 1:M | нет | CASCADE | связь теряет смысл при удалении номера |
| RoomAmenities.amenity_id | Amenities | 1:M | нет | CASCADE | связь теряет смысл при удалении удобства |
| BookingRooms.booking_id | Bookings | 1:M | нет | CASCADE | позиции бронирования не существуют без самого бронирования |
| BookingRooms.room_id | Rooms | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Payments.booking_id | Bookings | 1:M | да | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Payments.ticket_id | Tickets | 1:M | да | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Routes.carrier_id | Carriers | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Trips.route_id | Routes | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Trips.vehicle_id | Vehicles | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |
| Tickets.trip_id | Trips | 1:M | нет | RESTRICT | не удаляем данные, на которые есть ссылки (сохраняем историю) |

## 5. Правила, которые схема не выражает (будут реализованы триггерами и процедурами в лабораторной №5)

- Номер нельзя забронировать на пересекающиеся даты (`BookingRooms`: нет двух позиций одного номера с пересекающимися периодами в неотменённых бронированиях).
- `Bookings.total_price` пересчитывается при добавлении, изменении и удалении позиций бронирования.
- Отзыв можно оставить только в отеле, где у пользователя было завершённое бронирование.
- Число проданных билетов на рейс не больше `Vehicles.seats_total`; при отмене билета место освобождается (`seat_number = NULL`).
- Перевозчик маршрута рейса должен соответствовать транспорту рейса (если транспорт привязывается к перевозчику).

## 6. Рекомендуемые индексы (создаются в лабораторной №3)

Индексы по первичным, уникальным и внешним ключам MySQL создаёт сам. Дополнительно под функциональные требования:

- `Hotels (city_id, star_rating)` — поиск отелей по городу и звёздности;
- `Rooms (hotel_id, room_type_id, price_per_night)` — подбор номеров по цене и типу;
- `BookingRooms (room_id, check_in_date, check_out_date)` — проверка доступности номера на даты;
- `Routes (origin_city_id, destination_city_id)` и `Trips (route_id, departure_datetime)` — поиск рейсов по направлению и дате;
- `Tickets (user_id, purchased_at)` и `Bookings (user_id, created_at)` — история пользователя;
- `ActionLogs (user_id, created_at)` — просмотр журнала;
- `Reviews (hotel_id)` — отзывы отеля и средняя оценка.
