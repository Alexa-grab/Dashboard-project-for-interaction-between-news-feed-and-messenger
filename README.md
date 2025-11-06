
#### Учебный проект 
от Karpov.Courses
https://karpov.courses/simulator

# Взаимодействие ленты новостей и мессенджера: продуктовый анализ

Проект анализа данных о взаимодействии 2х  продуктов приложения с визуализацией в Apache Superset.


## Вводные данные
Социальная сеть с двумя сервисами:
- **Лента новостей** — просмотр и лайки постов
- **Мессенджер** — обмен сообщениями

Все события хранятся в ClickHouse:
- **feed_actions** — действия в ленте (просмотры, лайки)
- **message_actions** — отправка сообщений

## Задачи
- Составить дашборд, который описывает взаимодействие двух сервисов — ленты и сообщений. 

- Какая у нашего приложения активная аудитория по дням, т.е. пользователи, которые пользуются и лентой новостей, и сервисом сообщений. 

- Сколько пользователей использует только ленту новостей и не пользуются сообщениями.   

- Какие еще можно провести исследования о связи двух продуктов.

## Инструменты

**База данных**: ClickHouse  
**BI-инструменты**: Superset, Redash  

### Структура таблиц БД
**1. Feed_actions**
 - post_id  id поста;
 - user_id  id пользователя;
 - action   совершенное действие view/like;
 - time  время совершенного действия;
 - gender/city/country/os/source информация о пользователе.

**2. Message_actions**
- user_id  id отправителя сообщения;
- receiver_id id получателя сообщения;
- time-время совершенного действия;
- gender/city/country/os/source информация о пользователе.



## **Дашборд**
![Основной дашборд анализа](https://github.com/Alexa-grab/desktop-tutorial/blob/main/dashboard%20feed%20and%20message%201.png)
![Основной дашборд анализа](https://github.com/Alexa-grab/desktop-tutorial/blob/main/dashboard%20feed%20and%20message%202.png)

## **Ключевые SQL запросы для графиков к дашборду**

**1.Распределение аудитории**

**1.1. Пользователи которые взаимодействуют с обоими сервисами**
```
    SELECT DISTINCT user_id
    FROM simulator_20250820.feed_actions
    INTERSECT
    SELECT DISTINCT user_id
    FROM simulator_20250820.message_actions
   ```
     
**1.2. Пользователи которые пользуются только лентой новостей**

```
    SELECT DISTINCT user_id
    FROM simulator_20250820.feed_actions
    EXCEPT
    SELECT DISTINCT user_id
    FROM simulator_20250820.message_actions
```
**2. Динамика пользовательской активности**
 
 **2.1. Основные метрики DAU,MAU,WAU**
 ```
WITH 
-- Все пользователи, которые есть в обеих таблицах
common_users AS (
    SELECT DISTINCT user_id
    FROM simulator_20250820.feed_actions
    INTERSECT
    SELECT DISTINCT user_id
    FROM simulator_20250820.message_actions
)
-- Действия этих пользователей в ленте
SELECT 
    toDate(fa.time) AS day,
    fa.user_id,
--Добавление гендера в текстовом представлении
    CASE
        WHEN fa.gender = 1 THEN 'Woman'
        WHEN fa.gender = 0 THEN 'Man'
        ELSE 'Unknown'
    END AS gender_text,
--Распределение на возрастные группы
    CASE 
        WHEN fa.age > 1 AND fa.age <= 17 THEN 'До 17' 
        WHEN fa.age > 17 AND fa.age <= 24 THEN '18-24 лет' 
        WHEN fa.age > 24 AND fa.age <= 34 THEN '25-34 лет' 
        WHEN fa.age > 34 AND fa.age <= 44 THEN '35-44 лет' 
        WHEN fa.age > 44 AND fa.age <= 54 THEN '45-54 лет' 
        WHEN fa.age > 54 THEN '55+' 
        ELSE 'Не указан'
    END AS age_group,
    fa.action AS feed_action_type,
    fa.source,
    fa.city,
    fa.country,
    fa.os
FROM simulator_20250820.feed_actions fa
WHERE fa.user_id IN (SELECT user_id FROM common_users)
ORDER BY fa.user_id, fa.time
 ```
**3. Анализ активности использования сервиса обмена сообщениями**

 **3.1 Отправленные и полученные сообщения**
 ```
-- Количество отправленных сообщений по дням каждым пользователем
WITH sent AS (
    SELECT 
        toDate(time) as event_date,
        user_id,
        count(receiver_id) as messages_sent
    FROM simulator_20250820.message_actions
    GROUP BY event_date, user_id
    order by event_date
),
-- Количество полученных сообщений по дням каждым пользователем
receive AS (
    SELECT 
        toDate(time) as event_date,
        receiver_id,
        count(user_id) as messages_received
    FROM simulator_20250820.message_actions
    GROUP BY event_date, receiver_id
    order by event_date
) 
-- Выборка по дням с количеством отправленных и полученных сообщений для каждого юзера
SELECT 
    COALESCE(s.event_date, r.event_date) as event_date,
    COALESCE(s.user_id, r.receiver_id) as user_id,
    COALESCE(s.messages_sent, 0) as messages_sent,
    COALESCE(r.messages_received, 0) as messages_received
FROM sent s
FULL OUTER JOIN receive r
    ON s.user_id = r.receiver_id AND s.event_date = r.event_date
WHERE COALESCE(s.user_id, r.receiver_id) != 0
 ```
![Анализ активности использования сервиса обмена сообщениями](https://github.com/Alexa-grab/desktop-tutorial/blob/main/dashboard%20feed%20and%20message%204.png)

 **3.2. Топ 100 самых активных пользователей**
 ```
-- Количество отправленных сообщений каждым пользователем
WITH sent AS (
    SELECT 
        user_id,
        count(receiver_id) as messages_sent  
    FROM simulator_20250820.message_actions
    GROUP BY user_id
),
-- Количество полученных сообщений каждым пользователем
receive AS (
    SELECT 
        receiver_id as user_id,  
        count(user_id) as messages_received
    FROM simulator_20250820.message_actions
    GROUP BY receiver_id
),  
-- Количество действий выполненных каждым пользователем
feed_stats AS (
    SELECT 
        user_id,
        COUNT(*) AS feed_actions_count,
        SUM(action = 'like') AS like_count,
        SUM(action = 'view') AS view_count
    FROM simulator_20250820.feed_actions
    GROUP BY user_id
),
-- Метрики активности по каждому пользователю
combined_actions AS (
    SELECT 
        COALESCE(s.user_id, r.user_id, f.user_id) AS user_id,
        COALESCE(s.messages_sent, 0) + COALESCE(r.messages_received, 0) + COALESCE(f.feed_actions_count, 0) AS total_actions,  
        COALESCE(s.messages_sent, 0) AS sent_messages,  
        COALESCE(r.messages_received, 0) AS received_messages,  
        COALESCE(f.feed_actions_count, 0) AS feed_actions,
        COALESCE(f.like_count, 0) AS like_actions,
        COALESCE(f.view_count, 0) AS view_actions
    FROM sent s
-- Полное объединение с остальными cte, чтобы учесть все активности каждого пользователя
    FULL OUTER JOIN receive r ON s.user_id = r.user_id
    FULL OUTER JOIN feed_stats f ON COALESCE(s.user_id, r.user_id) = f.user_id
--Вывод 100 самых активных пользователей
SELECT 
    user_id,
    sent_messages,
    received_messages,
    feed_actions,
    like_actions,
    view_actions,
    total_actions
FROM combined_actions
ORDER BY total_actions DESC
limit 100
 ```
![100 самых активных пользователей](https://github.com/Alexa-grab/desktop-tutorial/blob/main/dashboard%20feed%20and%20message%203.png)

**4. Анализ аудитории** 

-График распределения аудитории по полу и возрасту.
-Столбчатая диаграмма 10 самых активных городов.
-График распределения аудитории по источнику входа (рекламный или органический).
 ```
--Выборка пользователей использующих оба сервиса 
WITH common_users AS (
    SELECT DISTINCT user_id
    FROM simulator_20250820.feed_actions
    INTERSECT
    SELECT DISTINCT user_id
    FROM simulator_20250820.message_actions
)
--Возраст,пол,город,os,источник входа по каждому пользователю
SELECT DISTINCT
    fa.user_id,
    CASE
        WHEN fa.gender = 1 THEN 'Woman'
        WHEN fa.gender = 0 THEN 'Man'
        ELSE 'Unknown'
    END AS gender_text,
    CASE 
        WHEN fa.age > 1 AND fa.age <= 17 THEN 'До 17' 
        WHEN fa.age > 17 AND fa.age <= 24 THEN '18-24 лет' 
        WHEN fa.age > 24 AND fa.age <= 34 THEN '25-34 лет' 
        WHEN fa.age > 34 AND fa.age <= 44 THEN '35-44 лет' 
        WHEN fa.age > 44 AND fa.age <= 54 THEN '45-54 лет' 
        WHEN fa.age > 54 THEN '55+' 
        ELSE 'Не указан'
    END AS age_group,
    fa.source,
    fa.city,
    fa.os
FROM simulator_20250820.feed_actions fa
WHERE fa.user_id IN (SELECT user_id FROM common_users)
ORDER BY fa.user_id
 ```
![Анализ аудитории](https://github.com/Alexa-grab/desktop-tutorial/blob/main/dashboard%20feed%20and%20message%205.png)

# **Результаты анализа**

**Распределение аудитории:**
- Общая аудитория: 127,800 пользователей;
- Кросс-платформенные пользователи: 29,800 (23.3%);
- Только лента новостей: 98,000 (76.7%);
- Анализ DAU/WAU/MAU показывает последовательный спад активности across всех метрик.
 
**Вывод:**
- Только 23% пользователей используют оба сервиса;
- 77% аудитории не задействует мессенджер — значительный потенциал роста;  
- Наблюдается спад активностей (DAU/WAU/MAU),что указывает на системную проблему удержания пользователей;
- Требуются срочные меры для остановки оттока и роста вовлеченности.

**Рекомендации:**
- Улучшить интеграцию между лентой и мессенджером;
- Внедрить триггерные уведомления для реактивации;
- Разработать сегментные стратегии для разных типов пользователей.



