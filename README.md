**Общий обзор**

Transaction Service API предоставляет интерфейс для управления кошельками, транзакциями, типами кошельков и платежными запросами. Сервис позволяет осуществлять операции создания, изменения и получения информации о кошельках, а также проводить транзакции с учетом различных типов комиссий и валют.

**Функциональные Требования:**

- Управление кошельками: создание новых кошельков, изменение их данных и получение актуальной информации.
- Управление типами кошельков: поддержка различных типов кошельков с уникальными настройками валюты и статуса.
- Платежные запросы: создание и отслеживание запросов на оплату с возможностью указания суммы, валюты и дополнительных параметров.
- Транзакции: проведение операций между кошельками, включая учет комиссий и конвертацию валют, при необходимости.

**Функциональные Требования:**

- Java 21
- Spring Web
- Spring Data JPA
- Postgres
- Flyway
- TestContainers
- Junit 5
- Mockito
- Docker

**Нефункциональные Требования:**

- Необходимо внедрить шаридрование БД
- Необходимо проумать грамотную стратегию (принцип) шардирования.

Модель данных
```plaintext
create table wallet_types (
uid uuid default uuid_generate_v4() primary key,
created_at timestamp default now() not null,
modified_at timestamp,
name varchar(32) not null,
currency_code varchar(3) not null,
status varchar(18) not null,
archived_at timestamp,
user_type varchar(15),
creator varchar(255),
modifier varchar(255)
);
create table wallets (
uid uuid default uuid_generate_v4() primary key,
created_at timestamp default now() not null,
modified_at timestamp,
name varchar(32) not null,
wallet_type_uid uuid not null constraint fk_wallets_wallet_types references wallet_types,
user_uid uuid not null,
status varchar(30) not null,
balance decimal default 0.0 not null,
archived_at timestamp,
);

create table payment_requests (
uid uuid default uuid_generate_v4() not null primary key,
created_at timestamp default now() not null,
modified_at timestamp,
user_uid uuid not null,
wallet_uid uuid not null references wallets (wallet_uid),
amount decimal default 0.0 not null,
status varchar,
comment varchar(256),
payment_method_id bigint
);

create table transactions (
uid uuid default uuid_generate_v4() not null primary key,
created_at timestamp default now() not null,
modified_at timestamp,
user_uid uuid not null,
wallet_uid uuid not null references wallets (uid),
wallet_name varchar(32) not null,
amount decimal default 0.0 not null,
type varchar(32) not null,
state varchar(32) not null,
payment_request_uid uuid not null references payment_requests on delete cascade
);
create table top_up_requests (
uid uuid default uuid_generate_v4() not null primary key,
created_at timestamp default now() not null,
provider varchar not null,
payment_request_uid uuid not null references payment_requests on delete cascade
);
create table withdrawal_requests (
uid uuid default uuid_generate_v4() not null primary key,
created_at timestamp default now() not null,
payment_request_uid uuid not null references payment_requests on delete cascade
);
create table transfer_requests (
uid uuid default uuid_generate_v4() not null primary key,
created_at timestamp default now() not null,
system_rate varchar not null,
payment_request_uid_from uuid not null references payment_requests on delete cascade,
payment_request_uid_to uuid not null references payment_requests on delete cascade
);
```

**Open API спецификация:**
```plaintext
openapi: 3.0.3
info:
title: Transaction Service API
description: >
Transaction Service API предоставляет интерфейс для управления кошельками, транзакциями, типами кошельков и платежными запросами.
Сервис поддерживает создание и подтверждение транзакций, а также поиск по фильтрам и получение данных по кошелькам пользователя.
version: 1.0.0
servers:
- url: /api/v1

paths:
/transactions:
get:
summary: Поиск транзакций по фильтрам
operationId: searchTransactions
parameters:
- in: query
name: user_uid
schema:
type: string
description: UID пользователя для фильтрации транзакций
- in: query
name: wallet_uid
schema:
type: string
description: UID кошелька для фильтрации транзакций
- in: query
name: type
schema:
type: string
enum: [TOPUP, WITHDRAWAL, TRANSFER]
description: Тип транзакции для фильтрации
- in: query
name: state
schema:
type: string
description: Состояние транзакции для фильтрации
- in: query
name: date_from
schema:
type: string
format: date
description: Начальная дата для фильтрации транзакций
- in: query
name: date_to
schema:
type: string
format: date
description: Конечная дата для фильтрации транзакций
responses:
'200':
description: Список транзакций по указанным фильтрам
content:
application/json:
schema:
type: array
items:
$ref: '#/components/schemas/TransactionResponse'
'400':
description: Неверный запрос

/wallets/user/{user_uid}:
get:
summary: Получение данных по кошелькам пользователя
operationId: getUserWallets
parameters:
- in: path
name: user_uid
required: true
schema:
type: string
description: UID пользователя для получения кошельков
responses:
'200':
description: Список кошельков пользователя
content:
application/json:
schema:
type: array
items:
$ref: '#/components/schemas/WalletResponse'
'404':
description: Пользователь не найден

/wallets/user/{user_uid}/currency/{currency}:
get:
summary: Получение данных по кошельку пользователя по валюте
operationId: getWalletByUserIdAndCurrency
parameters:
- in: path
name: user_uid
required: true
schema:
type: string
description: UID пользователя
- in: path
name: currency
required: true
schema:
type: string
maxLength: 3
description: Код валюты кошелька
responses:
'200':
description: Данные по кошельку пользователя
content:
application/json:
schema:
$ref: '#/components/schemas/WalletResponse'
'404':
description: Кошелек с указанной валютой не найден

/transactions/{uid}/status:
get:
summary: Получение текущего состояния транзакции
operationId: getTransactionStatus
parameters:
- in: path
name: uid
required: true
schema:
type: string
description: UID транзакции
responses:
'200':
description: Текущее состояние транзакции
content:
application/json:
schema:
$ref: '#/components/schemas/TransactionStatusResponse'
'404':
description: Транзакция не найдена

components:
schemas:
WalletResponse:
type: object
properties:
uid:
type: string
name:
type: string
wallet_type_uid:
type: string
user_uid:
type: string
status:
type: string
balance:
type: number
format: decimal
currency_code:
type: string
maxLength: 3
created_at:
type: string
format: date-time

    TransactionResponse:
      type: object
      properties:
        uid:
          type: string
        user_uid:
          type: string
        wallet_uid:
          type: string
        amount:
          type: number
          format: decimal
        type:
          type: string
        state:
          type: string
        created_at:
          type: string
          format: date-time

    TransactionStatusResponse:
      type: object
      properties:
        uid:
          type: string
        state:
          type: string
        updated_at:
          type: string
          format: date-time
      description: >
        Ответ с текущим состоянием транзакции и датой последнего обновления.
```

**Тестирование:**
- JUnit 5: Для юнит-тестов.
- Mockito: Для мокирования зависимостей.
- TestContainers: Для тестирования взаимодействия с БД используются тест-контейнеры.
