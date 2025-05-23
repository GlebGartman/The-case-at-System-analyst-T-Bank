# ЗАДАНИЕ

## Архитектура интеграционного взаимодействия Т-Банка и службы такси

### Модель с подтверждением через персональные ссылки и последующей обработкой поездок

---

## Общая идея

Интеграция основана на процессе подтверждения сотрудником своей принадлежности к корпоративному аккаунту,  
после чего все совершённые поездки передаются в банк в формате **JSON через API**.

---

## Пошаговый процесс интеграции

### Шаг 1. Банк передает список сотрудников партнёру

Внутри Т-Банка существует **централизованный справочник сотрудников**,  
который формируется и регулярно актуализируется на основе данных из HR-системы нашего банка.

Этот справочник хранится в таблице `employees_T_Bank`,  
которая содержит основную информацию о каждом сотруднике, включая его персональные и организационные данные.  
Приведённый состав полей является примером и при необходимости может быть дополнен дополнительной информацией о сотрудниках.


#### Код для создания таблицы

```sql
CREATE TABLE employees_T_Bank (
    employee_id SERIAL PRIMARY KEY,
    full_name VARCHAR(255) NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    work_email VARCHAR(255) NOT NULL, -- Стандарты email до 254 символов
    birth_date DATE NOT NULL,
    department VARCHAR(255) NOT NULL

    -- Дополнительная информация о сотруднике:
    -- - должность (position)
    -- - дата найма (hire_date)
    -- - статус занятости (employment_status)
    -- - руководитель (manager_id)
    -- - локация (location)
    -- - уровень доступа или роли в системе
);
```
##### Поля таблицы сотрудников

| Поле         | Описание                                                   |
|------------- |------------------------------------------------------------|
| employee_id   | Уникальный идентификатор сотрудника в Т-Банке                | <!-- spacing -->
|              |                                                             |
| full_name      | Полное ФИО сотрудника                                        | <!-- spacing -->
|              |                                                             |
| phone_number  | Рабочий номер телефона                                       | <!-- spacing -->
|              |                                                             |
| work_email   | Корпоративный электронный адрес сотрудника                   | <!-- spacing -->
|              |                                                             |
| birth_date   | Дата рождения                                                | <!-- spacing -->
|              |                                                             |
| department   | Наименование подразделения                                   |


---

## Назначение справочника

1. Является единственным источником актуальных данных о сотрудниках для внутренних и внешних сервисов.
2. Используется для:
   - Контроля прав доступа сотрудников.
   - Формирования списков участников внутренних и внешних программ, включая допуск к использованию корпоративного такси.

---

На основе данных из этого справочника банк отбирает сотрудников, которым разрешено пользоваться корпоративным такси.

### В выборке указываются:

- `employee_id` — как уникальный идентификатор в банке.
- `full_name` — для удобства идентификации.

---

### Пример подготовленного списка

| employee_id | full_name              |
|------------|------------------------|
| 101        | Иванов Иван Иванович    |
| 102        | Петров Пётр Петрович    |

Банк передаёт список в виде **JSON через API**.

### Пример структуры передаваемых данных в формате JSON:

```json
[
    {
        "employee_id": 101,
        "full_name": "Иванов Иван Иванович"
    },
    {
        "employee_id": 102,
        "full_name": "Петров Петр Петрович"
    }
]
```

### Назначение передачи

Этот список используется партнёром для:

1. Генерации персональных ссылок для подключения сотрудников к корпоративному счёту.
2. Формирования списка участников корпоративного тарифа Т-Банка.

---

## Шаг 2. Генерация персональных ссылок, подтверждение сотрудниками подключения и формирование записей о связке

После того как банк отправил партнёру список сотрудников с уникальными идентификаторами `employee_id` и ФИО (Шаг 1),  
партнёр формирует для каждого сотрудника персональную ссылку для подключения к корпоративному счёту Т-Банка.  
Эта ссылка содержит `employee_id`, который позволяет системе такси в дальнейшем связать пользователя с корпоративным счётом Т-Банка.

### Пример персональной ссылки для сотрудника с `employee_id = 101`:
https://taxi.com/business/join?tbank_corp&employee_id=101


Сотрудник переходит по полученной ссылке в приложении такси,  
находясь в своём личном аккаунте, зарегистрированном на номер телефона или email.  
В момент перехода система такси автоматически определяет текущий активный аккаунт пользователя,  
получая его уникальный идентификатор — `user_id` (внутренний ключ пользователя в системе такси).

Пользователю отображается приглашение привязать текущий аккаунт к корпоративному счёту Т-Банка.  
После подтверждения подключения система такси фиксирует связку между:

- `employee_id` — полученным из ссылки;
- `user_id` — определённым по текущему активному аккаунту пользователя.

Также система такси автоматически подтягивает ФИО сотрудника,  
ранее полученное от банка вместе с `employee_id`, и сохраняет его вместе со связкой.

---
### В результате формируется запись в таблице связей, которая содержит:

- `employee_id` — идентификатор сотрудника в системе банка, взятый из ссылки;
- `full_name` — ФИО сотрудника, полученное из списка банка;
- `user_id` — уникальный идентификатор пользователя в системе такси, определённый по активной сессии.

Таким образом, при подключении сотрудника создаётся полная и однозначная связка  
между сотрудником Т-Банка и аккаунтом в системе такси,  
что позволяет в дальнейшем правильно учитывать поездки, совершённые по корпоративному счёту.

После формирования связок партнёр (служба такси) передаёт этот
список в Т-Банк для фиксации и дальнейшей обработки поездок.
Формат передачи данных - JSON, передаваемый через API.

### Пример структуры передаваемых данных в формате JSON:

```json

[
 {
 "employee_id": 101,
 "full_name": "Иванов Иван Иванович",
 "user_id": "acc_ivanov"
 },
 {
 "employee_id": 102,
 "full_name": "Петров Пётр Петрович",
 "user_id": "acc_petrov"
 }
]
```

Банк принимает эти данные и сохраняет их в собственной таблице
**employee_taxi_link_T_Bank**,
что позволяет надёжно связать все последующие поездки с конкретными
сотрудниками.
#### Код для создания таблицы
```sql
CREATE TABLE employee_taxi_link_T_Bank (
employee_id BIGINT UNSIGNED NOT NULL REFERENCES
employees_T_Bank(employee_id),
 -- Уникальный ID сотрудника в системе Т-Банка, взятый из персональной
ссылки
 full_name VARCHAR(255) NOT NULL,
 -- ФИО сотрудника, автоматически подтягивается из ранее переданного
банком списка
 user_id VARCHAR(255) UNIQUE NOT NULL,
 -- Уникальный ID пользователя в системе такси, определяется автоматически
по активной сессии в приложении такси
 PRIMARY KEY (employee_id)
 -- Один сотрудник может быть связан только с одним аккаунтом в такси
)
```

---
## Шаг 3. Создание таблицы для хранения корпоративных поездок в Т-Банке и описание связей
Для приёма и хранения информации о поездках сотрудников,
совершаемых за счёт Т-Банка, в базе данных банка создаётся третья
таблица - **corporate_taxi_trips_T_Bank**.
#### Код для создания таблицы
```sql
CREATE TABLE corporate_taxi_trips_T_Bank (
 trip_id SERIAL PRIMARY KEY,
 -- Уникальный идентификатор поездки внутри банка (служебный ключ).
 user_id VARCHAR(255) NOT NULL REFERENCES
employee_taxi_link_Bank(user_id),
 -- Уникальный ID пользователя в системе такси
 trip_date_time TIMESTAMP NOT NULL,
 -- Дата и время начала поездки (важно для проверки рабочего времени,
отчетности и аналитики).
 origin_address VARCHAR(500) NOT NULL,
 -- Адрес начала поездки.
 destination_address VARCHAR(500) NOT NULL,
 -- Адрес назначения поездки.
 trip_amount NUMERIC(7, 2) NOT NULL
 -- Стоимость поездки в денежном выражении
)
```
| Поле               | Описание                                                       |
|--------------------|-----------------------------------------------------------------|
| trip_id            | Уникальный идентификатор поездки в системе банка                |
| user_id            | Уникальный ID пользователя в системе такси                      |
| trip_date_time     | Дата и время начала поездки                                     |
| origin_address     | Адрес начала поездки                                            |
| destination_address| Адрес назначения поездки                                        |
| trip_amount        | Стоимость поездки в денежном выражении                          |

---

Поле `user_id` в таблице **corporate_taxi_trips_T_Bank**
связывается с полем `user_id` в таблице **employee_taxi_link_T_Bank**,  
которая содержит информацию о сотрудниках Т-Банка, подтвердивших подключение к корпоративному счёту такси.

Это позволяет однозначно определить, какому сотруднику Т-Банка принадлежит каждая поездка.

### Назначение таблицы:

1. Хранение всех корпоративных поездок, которые поступают в банк от партнёра (такси).
2. Связь поездок с конкретными сотрудниками через `user_id`.
3. Использование для расчёта компенсаций, отчётности и аналитики.

---

## Шаг 4. Передача информации о корпоративных поездках от партнёра в Т-Банк

### Где хранятся данные на стороне партнёра

На стороне службы такси ведётся корпоративный профиль Т-Банка,  
в котором хранится список сотрудников, подключённых к корпоративному счёту.

Все поездки сотрудников сохраняются в общей базе поездок такси,  
но имеют привязку к корпоративному счёту Т-Банка.

Эта привязка позволяет партнёру выгружать только корпоративные поездки, относящиеся к Т-Банку.
### Передача данных в банк

Партнёр ежедневно формирует выгрузку корпоративных поездок,  
совершённых сотрудниками Т-Банка, и передаёт её в банк через API в формате JSON.

#### Пример структуры передаваемых данных:

```json
[
    {
        "user_id": "acc_ivanov",
        "trip_date_time": "2025-05-12T09:30:00",
        "origin_address": "БЦ Т-Банк",
        "destination_address": "Аэропорт",
        "trip_amount": 1450.50
    }
]
```
Банк принимает эти данные и сохраняет их в собственной таблице  
**corporate_taxi_trips_T_Bank**.

### Назначение передаваемой информации

Банк принимает эти данные и:

1. Сохраняет поездки в таблице `corporate_taxi_trips_T_Bank`.
2. Связывает поездки с сотрудниками через таблицу `employee_taxi_link_T_Bank`.
3. Использует информацию для расчётов компенсаций и подготовки отчётности.

---

## Шаг 5. Связь с сотрудниками через таблицу подтверждённых подключений

Далее банк связывает загруженные поездки с конкретными сотрудниками,  
используя таблицу `employee_taxi_link_T_Bank`, которая содержит связку между:

- `employee_id` — уникальным идентификатором сотрудника Т-Банка.
- `user_id` — уникальным идентификатором пользователя в системе такси.

**Пример SQL-запроса для получения информации о поездках сотрудников:**

```sql
SELECT 
    l.employee_id,
    l.full_name,
    t.trip_date_time,
    t.origin_address,
    t.destination_address,
    t.trip_amount
FROM corporate_taxi_trips_T_Bank t
JOIN employee_taxi_link_T_Bank l ON t.user_id = l.user_id;
```
---
## Структура хранения информации в базе данных Т-Банка

Сама структура хранения информации в базе данных Т-Банка  
включает три связанные между собой таблицы и выглядит следующим образом:
![Схема архитектуры](https://drive.google.com/uc?export=view&id=1iuq9KXsms7eRKv0kzbkglYrBbtC0Stwa)


---

## Оценка предложенной архитектуры

Предложенная архитектура интеграции обеспечивает надёжный и управляемый процесс учёта корпоративных поездок сотрудников Т-Банка с использованием сервиса такси.  
Решение построено на разделении ответственности между партнёром и Т-Банком, что позволяет минимизировать дублирование данных и обеспечивает актуальность информации.

На стороне партнёра хранится список подключённых сотрудников и полная база всех совершённых поездок с указанием привязки к корпоративному счёту Т-Банка.  
На стороне Т-Банка формируется собственная база, включающая:

- Справочник сотрудников.
- Таблицу связей между сотрудниками и аккаунтами в системе такси.
- Таблицу всех полученных от партнёра поездок.

Передача данных реализуется через защищённые каналы в формате **JSON**,  
что обеспечивает надёжность и автоматизацию обмена.

Связывание поездок с сотрудниками происходит через заранее зафиксированные связки  
`employee_id` и `user_id`, что гарантирует корректную идентификацию сотрудников в отчётности и расчётах.

### Преимущества архитектуры:

1. Исключение ошибок сопоставления поездок и сотрудников.
2. Контроль над расходами на корпоративное такси.
3. Формирование прозрачной отчётности на основе достоверных данных.
4. Лёгкая масштабируемость под другие корпоративные сервисы или партнёров.

Такое решение соответствует требованиям бизнеса и стандартам надёжного интеграционного взаимодействия.

