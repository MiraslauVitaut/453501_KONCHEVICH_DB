# Сущности БД

Проект: **Система бронирования (отели и билеты)** · СУБД: MySQL
Всего сущностей: **16** (минимум по заданию — 10). Присутствуют все виды связей: 1:1, 1:М, М:М.
Схема — см. `er_diagram.png` (ненормализованная инфологическая модель).

---

## 1. Users — пользователи

Ядро системы: учётная запись клиента или сотрудника.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| email | VARCHAR(255) | NOT NULL, UNIQUE |
| password_hash | VARCHAR(255) | NOT NULL |
| role_id | INT | NOT NULL, FK → Roles.id |
| is_active | BOOLEAN | NOT NULL, DEFAULT TRUE |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Связи: М:1 к Roles; 1:1 к UserProfiles; 1:М к ActionLogs, Bookings, Reviews, Tickets.

---

## 2. Roles — роли

Справочник ролей системы.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(50) | NOT NULL, UNIQUE (customer / manager / admin) |
| description | VARCHAR(255) | NULL |

Связи: 1:М к Users.

---

## 3. UserProfiles — профили пользователей

Расширенные персональные данные, вынесены из Users отдельно.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| user_id | INT | NOT NULL, UNIQUE, FK → Users.id |
| full_name | VARCHAR(150) | NOT NULL |
| birth_date | DATE | NULL |
| passport_number | VARCHAR(20) | NULL |
| phone | VARCHAR(20) | NULL |
| address | VARCHAR(255) | NULL |

Связи: **1:1** к Users (уникальность user_id обеспечивает связь один-к-одному).

---

## 4. ActionLogs — журнал действий

Реализует обязательное требование журналирования действий пользователя.

| Поле | Тип | Ограничения |
|---|---|---|
| id | BIGINT | PK, AUTO_INCREMENT |
| user_id | INT | NOT NULL, FK → Users.id |
| action_type | VARCHAR(50) | NOT NULL (LOGIN, BOOKING_CREATE, BOOKING_CANCEL, PROFILE_UPDATE, …) |
| description | TEXT | NULL |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Связи: М:1 к Users.

---

## 5. Hotels — отели

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(150) | NOT NULL |
| city | VARCHAR(100) | NOT NULL |
| address | VARCHAR(255) | NOT NULL |
| description | TEXT | NULL |
| star_rating | TINYINT | NOT NULL, CHECK (star_rating BETWEEN 1 AND 5) |

Связи: 1:М к Rooms; 1:М к Reviews.

---

## 6. RoomTypes — типы номеров

Справочник (одноместный, двухместный, люкс и т.д.).

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(100) | NOT NULL, UNIQUE |
| description | TEXT | NULL |
| base_price | DECIMAL(10,2) | NOT NULL |

Связи: 1:М к Rooms.

---

## 7. Rooms — номера

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| hotel_id | INT | NOT NULL, FK → Hotels.id |
| room_type_id | INT | NOT NULL, FK → RoomTypes.id |
| room_number | VARCHAR(10) | NOT NULL |
| floor | INT | NULL |
| capacity | TINYINT | NOT NULL |
| price_per_night | DECIMAL(10,2) | NOT NULL |
| is_available | BOOLEAN | NOT NULL, DEFAULT TRUE |

Ограничение: UNIQUE (hotel_id, room_number).

Связи: М:1 к Hotels; М:1 к RoomTypes; **М:М** к Amenities (через RoomAmenities); **М:М** к Bookings (через BookingRooms).

---

## 8. Amenities — удобства

Справочник (Wi-Fi, парковка, бассейн, кондиционер и т.д.).

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(100) | NOT NULL, UNIQUE |

Связи: **М:М** к Rooms (через RoomAmenities).

---

## 8a. RoomAmenities — связующая таблица (не самостоятельная сущность)

Реализует связь М:М между Rooms и Amenities. Не считается отдельной сущностью, т.к. не несёт дополнительной информации, кроме самой связи.

| Поле | Тип | Ограничения |
|---|---|---|
| room_id | INT | PK (составной), FK → Rooms.id |
| amenity_id | INT | PK (составной), FK → Amenities.id |

---

## 9. Bookings — бронирования

Факт бронирования, оформленный пользователем (может включать несколько номеров).

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| user_id | INT | NOT NULL, FK → Users.id |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'PENDING' (PENDING / CONFIRMED / CANCELLED / COMPLETED) |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP |
| total_price | DECIMAL(10,2) | NOT NULL, DEFAULT 0 |

Связи: М:1 к Users; 1:М к Payments; **М:М** к Rooms (через BookingRooms).

---

## 10. BookingRooms — позиции бронирования

Связывает Bookings и Rooms; хранит даты и цену на момент бронирования — поэтому считается самостоятельной сущностью, а не просто связующей таблицей.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| booking_id | INT | NOT NULL, FK → Bookings.id |
| room_id | INT | NOT NULL, FK → Rooms.id |
| check_in_date | DATE | NOT NULL |
| check_out_date | DATE | NOT NULL, CHECK (check_out_date > check_in_date) |
| price_at_booking | DECIMAL(10,2) | NOT NULL |

Связи: М:1 к Bookings; М:1 к Rooms (в совокупности реализует **М:М** между Bookings и Rooms).

---

## 11. Payments — платежи

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| booking_id | INT | NOT NULL, FK → Bookings.id |
| amount | DECIMAL(10,2) | NOT NULL |
| payment_method | VARCHAR(30) | NOT NULL (CARD / CASH / ONLINE) |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'PENDING' (PENDING / SUCCESS / FAILED / REFUNDED) |
| paid_at | DATETIME | NULL |

Связи: М:1 к Bookings (одно бронирование может иметь несколько платежей — например, предоплата и доплата).

---

## 12. Reviews — отзывы

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| user_id | INT | NOT NULL, FK → Users.id |
| hotel_id | INT | NOT NULL, FK → Hotels.id |
| rating | TINYINT | NOT NULL, CHECK (rating BETWEEN 1 AND 5) |
| comment | TEXT | NULL |
| created_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Ограничение: UNIQUE (user_id, hotel_id) — один отзыв от пользователя на отель.

Связи: М:1 к Users; М:1 к Hotels.

---

## 13. Carriers — перевозчики

Авиакомпании, ж/д или автобусные компании.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| name | VARCHAR(150) | NOT NULL |
| type | VARCHAR(20) | NOT NULL (AVIA / RAIL / BUS) |

Связи: 1:М к Routes.

---

## 14. Routes — маршруты

Направление перевозки (откуда — куда) конкретного перевозчика.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| carrier_id | INT | NOT NULL, FK → Carriers.id |
| origin_city | VARCHAR(100) | NOT NULL |
| destination_city | VARCHAR(100) | NOT NULL |
| distance_km | INT | NULL |

Связи: М:1 к Carriers; 1:М к Trips.

---

## 15. Trips — рейсы

Конкретный рейс по маршруту на определённые дату и время.

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| route_id | INT | NOT NULL, FK → Routes.id |
| departure_datetime | DATETIME | NOT NULL |
| arrival_datetime | DATETIME | NOT NULL, CHECK (arrival_datetime > departure_datetime) |
| vehicle_number | VARCHAR(20) | NULL |
| base_price | DECIMAL(10,2) | NOT NULL |
| seats_total | INT | NOT NULL |

Связи: М:1 к Routes; 1:М к Tickets.

---

## 16. Tickets — билеты

| Поле | Тип | Ограничения |
|---|---|---|
| id | INT | PK, AUTO_INCREMENT |
| user_id | INT | NOT NULL, FK → Users.id |
| trip_id | INT | NOT NULL, FK → Trips.id |
| seat_number | VARCHAR(10) | NULL |
| price | DECIMAL(10,2) | NOT NULL |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'BOOKED' (BOOKED / CANCELLED / USED) |
| purchased_at | DATETIME | NOT NULL, DEFAULT CURRENT_TIMESTAMP |

Связи: М:1 к Users; М:1 к Trips.

---

## Сводка по видам связей

- **1:1** — Users ↔ UserProfiles
- **1:М** — Roles→Users, Users→ActionLogs, Users→Bookings, Users→Reviews, Users→Tickets, Hotels→Rooms, Hotels→Reviews, RoomTypes→Rooms, Bookings→Payments, Carriers→Routes, Routes→Trips, Trips→Tickets, Bookings→BookingRooms, Rooms→BookingRooms, Rooms→RoomAmenities, Amenities→RoomAmenities
- **М:М** — Rooms ↔ Amenities (реализована парой 1:М через RoomAmenities), Bookings ↔ Rooms (реализована парой 1:М через BookingRooms)

Итого: 17 связей 1:М/1:1 между сущностями (включая связи, образующие обе М:М-пары), плюс 2 полноценные М:М-связи на концептуальном уровне.
