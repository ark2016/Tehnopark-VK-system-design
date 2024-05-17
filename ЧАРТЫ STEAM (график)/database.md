### Разработка Базы Данных для Раздела "Чарты Steam"

Для реализации раздела "Чарты Steam" выберем реляционную базу данных, поскольку она позволяет структурированно хранить данные и выполнять сложные аналитические запросы с использованием SQL. Реляционная база данных будет поддерживать транзакции и соответствовать требованиям 3НФ для обеспечения целостности данных.

#### Основные Таблицы

1. **Users:** Пользователи Steam
2. **Games:** Игры
3. **Tags:** Теги
4. **GameTags:** Связь тегов с играми
5. **Purchases:** Покупки пользователей
6. **Reviews:** Отзывы пользователей
7. **ConcurrentPlayers:** График активности пользователей в играх
8. **TopSellers:** Еженедельные лидеры продаж
9. **BestOfYear:** Лучшие игры за год

#### Структура Базы Данных

**Таблица Users**
```sql
CREATE TABLE Users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    country VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Таблица Games**
```sql
CREATE TABLE Games (
    game_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    developer VARCHAR(100),
    publisher VARCHAR(100),
    release_date DATE,
    price DECIMAL(8, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Таблица Tags**
```sql
CREATE TABLE Tags (
    tag_id INT PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);
```

**Таблица GameTags**
```sql
CREATE TABLE GameTags (
    game_id INT,
    tag_id INT,
    PRIMARY KEY (game_id, tag_id),
    FOREIGN KEY (game_id) REFERENCES Games (game_id),
    FOREIGN KEY (tag_id) REFERENCES Tags (tag_id)
);
```

**Таблица Purchases**
```sql
CREATE TABLE Purchases (
    purchase_id INT PRIMARY KEY,
    user_id INT,
    game_id INT,
    purchase_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    amount DECIMAL(8, 2),
    FOREIGN KEY (user_id) REFERENCES Users (user_id),
    FOREIGN KEY (game_id) REFERENCES Games (game_id)
);
```

**Таблица Reviews**
```sql
CREATE TABLE Reviews (
    review_id INT PRIMARY KEY,
    user_id INT,
    game_id INT,
    rating INT CHECK (rating BETWEEN 0 AND 10),
    text TEXT,
    review_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES Users (user_id),
    FOREIGN KEY (game_id) REFERENCES Games (game_id)
);
```

**Таблица ConcurrentPlayers**
```sql
CREATE TABLE ConcurrentPlayers (
    game_id INT,
    timestamp TIMESTAMP,
    concurrent_players INT,
    PRIMARY KEY (game_id, timestamp),
    FOREIGN KEY (game_id) REFERENCES Games (game_id)
);
```

**Таблица TopSellers**
```sql
CREATE TABLE TopSellers (
    rank INT,
    game_id INT,
    week_start DATE,
    week_end DATE,
    PRIMARY KEY (rank, week_start, week_end),
    FOREIGN KEY (game_id) REFERENCES Games (game_id)
);
```

**Таблица BestOfYear**
```sql
CREATE TABLE BestOfYear (
    rank INT,
    game_id INT,
    year INT,
    PRIMARY KEY (rank, year),
    FOREIGN KEY (game_id) REFERENCES Games (game_id)
);
```

### API Запросы

1. **SteamAPI_ISteamChartsService_GetGamesByConcurrentPlayers**
   ```sql
   SELECT game_id, name, MAX(concurrent_players) AS max_concurrent_players
   FROM ConcurrentPlayers
   JOIN Games USING (game_id)
   WHERE timestamp BETWEEN @start_date AND @end_date
   GROUP BY game_id, name
   ORDER BY max_concurrent_players DESC;
   ```

2. **SteamAPI_IShoppingCartService_GetShoppingCartContents**
   ```sql
   SELECT game_id, name, price
   FROM Games
   WHERE game_id IN (SELECT game_id FROM Purchases WHERE user_id = @user_id);
   ```

3. **SteamAPI_ISteamChartsService_GetBestOfYearPages**
   ```sql
   SELECT rank, game_id, name, price
   FROM BestOfYear
   JOIN Games USING (game_id)
   WHERE year = @year
   ORDER BY rank;
   ```

4. **SteamAPI_ISteamChartsService_GetTopReleasesPages**
   ```sql
   SELECT game_id, name, price, SUM(amount) AS total_sales
   FROM Purchases
   JOIN Games USING (game_id)
   WHERE release_date BETWEEN @start_date AND @end_date
   GROUP BY game_id, name, price
   ORDER BY total_sales DESC;
   ```

5. **SteamAPI_IStoreBrowseService_GetItems**
   ```sql
   SELECT game_id, name, price, developer, publisher, release_date
   FROM Games
   WHERE game_id IN @game_ids
   ORDER BY name;
   ```

6. **SteamAPI_IStoreQueryService_Query**
   ```sql
   SELECT game_id, name, price
   FROM Games
   WHERE name ILIKE CONCAT('%', @query, '%')
   OR game_id IN (
       SELECT game_id
       FROM GameTags
       JOIN Tags USING (tag_id)
       WHERE name ILIKE CONCAT('%', @query, '%')
   )
   ORDER BY name;
   ```

7. **SteamAPI_IStoreTopSellersService_GetWeeklyTopSellers**
   ```sql
   SELECT rank, game_id, name, price
   FROM TopSellers
   JOIN Games USING (game_id)
   WHERE week_start = @week_start AND week_end = @week_end
   ORDER BY rank;
   ```

8. **SteamAPI_dynamicstore_saledata**
   ```sql
   SELECT game_id, name, price, SUM(amount) AS total_sales
   FROM Purchases
   JOIN Games USING (game_id)
   WHERE purchase_date BETWEEN @start_date AND @end_date
   GROUP BY game_id, name, price
   ORDER BY total_sales DESC;
   ```

9. **SteamAPI_dynamicstore_userdata**
   ```sql
   SELECT Users.user_id, Users.username, Purchases.game_id, Games.name
   FROM Purchases
   JOIN Users USING (user_id)
   JOIN Games USING (game_id)
   WHERE Users.user_id = @user_id;
   ```

10. **SteamAPI_stats_userdata**
   ```sql
   SELECT game_id, timestamp, concurrent_players
   FROM ConcurrentPlayers
   WHERE game_id IN (
       SELECT game_id FROM Purchases WHERE user_id = @user_id
   )
   ORDER BY timestamp DESC
   LIMIT @limit;
   ```

### Расчет Размеров Данных и Узких Мест

**Предположения:**
1. **1 млн. пользователей**
2. **100 тыс. игр**
3. **500 тыс. тегов**
4. **10 млн. покупок**
5. **5 млн. отзывов**
6. **10 тыс. недельных топов**
7. **100 млн. записей о количестве игроков**

**Размер Таблиц:**

- **Users**
    - `user_id`: 4 байта
    - `username`: 50 байт
    - `email`: 100 байт
    - `country`: 50 байт
    - `created_at`: 8 байт
    - **Итого:** 4 + 50 + 100 + 50 + 8 = 212 байт на пользователя
    - **Размер:** 1 млн. * 212 байт = **212 МБ**

- **Games**
    - `game_id`: 4 байта
    - `name`: 100 байт
    - `developer`: 100 байт
    - `publisher`: 100 байт
    - `release_date`: 4 байта
    - `price`: 8 байт
    - `created_at`: 8 байт
    - **Итого:** 4 + 100 + 100 + 100 + 4 + 8 + 8 = 324 байта на игру
    - **Размер:** 100 тыс. * 324 байта = **32.4 МБ**

- **Tags**
    - `tag_id`: 4 байта
    - `name`: 50 байт
    - **Итого:** 4 + 50 = 54 байта на тег
    - **Размер:** 500 тыс. * 54 байта = **27 МБ**

- **GameTags**
    - `game_id`: 4 байта
    - `tag_id`: 4 байта
    - **Итого:** 4 + 4 = 8 байт на запись
    - **Предположим по 5 тегов на игру:** 100 тыс. * 5 * 8 байт = **4 МБ**

- **Purchases**
    - `purchase_id`: 4 байта
    - `user_id`: 4 байта
    - `game_id`: 4 байта
    - `purchase_date`: 8 байт
    - `amount`: 8 байт
    - **Итого:** 4 + 4 + 4 + 8 + 8 = 28 байт на покупку
    - **Размер:** 10 млн. * 28 байт = **280 МБ**

- **Reviews**
    - `review_id`: 4 байта
    - `user_id`: 4 байта
    - `game_id`: 4 байта
    - `rating`: 4 байта
    - `text`: 500 байт (средний размер)
    - `review_date`: 8 байт
    - **Итого:** 4 + 4 + 4 + 4 + 500 + 8 = 524 байта на отзыв
    - **Размер:** 5 млн. * 524 байта = **2.62 ГБ**

- **ConcurrentPlayers**
    - `game_id`: 4 байта
    - `timestamp`: 8 байт
    - `concurrent_players`: 4 байта
    - **Итого:** 4 + 8 + 4 = 16 байт на запись
    - **Размер:** 100 млн. * 16 байт = **1.6 ГБ**

- **TopSellers**
    - `rank`: 4 байта
    - `game_id`: 4 байта
    - `week_start`: 4 байта
    - `week_end`: 4 байта
    - **Итого:** 4 + 4 + 4 + 4 = 16 байт на запись
    - **Предположим 100 топов в неделю в среднем, за 10 тыс. недель:** 100 * 10 тыс. * 16 байт = **16 МБ**

- **BestOfYear**
    - `rank`: 4 байта
    - `game_id`: 4 байта
    - `year`: 4 байта
    - **Итого:** 4 + 4 + 4 = 12 байт на запись
    - **Предположим 100 игр в год:** 100 * 10 * 12 байт = **12 КБ**

**Общая Оценка Размеров Данных**

- **Users:** 212 МБ
- **Games:** 32.4 МБ
- **Tags:** 27 МБ
- **GameTags:** 4 МБ
- **Purchases:** 280 МБ
- **Reviews:** 2.62 ГБ
- **ConcurrentPlayers:** 1.6 ГБ
- **TopSellers:** 16 МБ
- **BestOfYear:** 12 КБ

**Общий размер базы данных:**  
`212 МБ + 32.4 МБ + 27 МБ + 4 МБ + 280 МБ + 2.62 ГБ + 1.6 ГБ + 16 МБ + 12 КБ ≈ 4.79 ГБ`

### Потенциальные Узкие Места 

1. **Запросы к таблице ConcurrentPlayers:** Из-за большого количества записей (100 млн.) запросы на агрегацию данных об активности пользователей могут быть медленными.

   **Решения:**
    - Индексы по `game_id` и `timestamp`.
    - Создать материализованные представления для часто запрашиваемых периодов.
    - Разделить таблицу на партиции по временным периодам (например, по месяцам).

2. **Запросы к таблице Reviews:** Из-за большого количества отзывов (5 млн.) запросы на агрегацию данных по отзывам могут быть долгими.

   **Решения:**
    - Добавить индексы по `game_id` и `user_id`.
    - Создать материализованные представления для часто используемых агрегатов, например, средние оценки и количество отзывов на игру.

3. **Запросы к таблице Purchases:** Из-за большого количества покупок (10 млн.) запросы на анализ продаж могут быть медленными.

   **Решения:**
    - Индексы по `user_id`, `game_id` и `purchase_date`.
    - Создать отдельные таблицы-агрегаты для топовых игр по продажам за определенные периоды.

4. **Запросы к таблице TopSellers:** Узким местом могут быть запросы к недельным топам, если период большой и запросы требуют много данных.

   **Решения:**
    - Индексы по `week_start` и `week_end`.
    - Материализованные представления для часто используемых периодов.

### Использование Кэша

**Зачем нужен кэш?**  
Основное назначение кэша - ускорить доступ к наиболее востребованным данным, особенно агрегированным. Можно использовать Redis или Memcached.

**Какие данные кэшировать?**
1. **Топ недельных продаж:** Кэшировать результаты запросов к таблице `TopSellers`.
    - **Ключ:** `top_sellers:{week_start}:{week_end}`
    - **Значение:** JSON-данные с рангом, `game_id`, названием и ценой игры.

   **Пример кэша:**
   ```json
   {
     "top_sellers:2024-05-01:2024-05-07": [
       {"rank": 1, "game_id": 101, "name": "Game A", "price": 29.99},
       {"rank": 2, "game_id": 102, "name": "Game B", "price": 19.99},
       "..."
     ]
   }
   ```

2. **Лучшие игры по годам:** Кэшировать результаты агрегационного запроса `SteamAPI_ISteamChartsService_GetBestOfYearPages`.
    - **Ключ:** `best_of_year:{year}`
    - **Значение:** JSON-данные с `game_id`, названием, ценой и продажами.

3. **Список тегов:** Кэшировать полный список тегов.
    - **Ключ:** `all_tags`
    - **Значение:** JSON-данные со списком тегов.

4. **Информация об игре:** Кэшировать информацию об отдельных играх.
    - **Ключ:** `game_info:{game_id}`
    - **Значение:** JSON-данные с информацией об игре.

5. **График активности пользователей:** Кэшировать результаты агрегационных запросов к таблице `ConcurrentPlayers`.
    - **Ключ:** `concurrent_players:{game_id}:{start_date}:{end_date}`
    - **Значение:** JSON-данные с временными метками и количеством активных пользователей.

### Расчет Размеров Кэша

**Предположения:**
1. **Топ недельных продаж:** 100 игр за 52 недели.
2. **Лучшие игры за год:** Топ-100 игр за последние 5 лет.
3. **График активности пользователей:** Топ-1000 игр, 24 записи в день за 30 дней.
4. **Информация об игре:** Топ-1000 игр.
5. **Средние рейтинги игр:** Средний рейтинг и количество отзывов для топ-1000 игр.

**Размер Топа Недельных Продаж:**
- **Оценка размера одной записи:**
    - `rank`: 4 байта
    - `game_id`: 4 байта
    - `name`: 100 байт
    - `price`: 8 байт
    - **Итого:** 4 + 4 + 100 + 8 = 116 байт

- **Размер одной недели:**
    - 100 игр * 116 байт = 11.6 КБ

- **Размер всех недель:**
    - 52 недели * 11.6 КБ = **0.6 МБ**

**Размер Лучших Игр Года:**
- **Оценка размера одной записи:**
    - `rank`: 4 байта
    - `game_id`: 4 байта
    - `name`: 100 байт
    - `price`: 8 байт
    - `total_sales`: 8 байт
    - **Итого:** 4 + 4 + 100 + 8 + 8 = 124 байта

- **Размер одного года:**
    - 100 игр * 124 байта = 12.4 КБ

- **Размер всех годов:**
    - 5 лет * 12.4 КБ = **0.06 МБ**

**Размер Графика Активности Пользователей:**
- **Оценка размера одной записи:**
    - `game_id`: 4 байта
    - `timestamp`: 8 байт
    - `concurrent_players`: 4 байта
    - **Итого:** 4 + 8 + 4 = 16 байт

- **Размер всех записей:**
    - 1000 игр * 24 записи/день * 30 дней * 16 байт = **11.52 МБ**

**Размер Информации Об Игре:**
- **Оценка размера одной записи:**
    - `game_id`: 4 байта
    - `name`: 100 байт
    - `developer`: 100 байт
    - `publisher`: 100 байт
    - `release_date`: 10 байт
    - `price`: 8 байт
    - **Итого:** 4 + 100 + 100 + 100 + 10 + 8 = 322 байта

- **Размер всех игр:**
    - 1000 игр * 322 байта = **0.32 МБ**

**Размер Средних Рейтингов Игр:**
- **Оценка размера одной записи:**
    - `game_id`: 4 байта
    - `average_rating`: 8 байт
    - `count_reviews`: 4 байта
    - **Итого:** 4 + 8 + 4 = 16 байт

- **Размер всех записей:**
    - 1000 игр * 16 байт = **0.016 МБ**

### Общий Размер Кэша

Суммируя размеры кэша для всех ключевых таблиц и данных, получим общий ориентировочный размер кэша:

- **TopSellers**: 0.6 MB
- **BestOfYear**: 0.06 MB
- **ConcurrentPlayers**: 11.52 MB
- **Games (детали игр)**: 0.32 MB
- **Reviews (средние рейтинги)**: 0.016 MB

**Общий предполагаемый размер кэша**: 
0.6 + 0.06 + 11.52 + 0.32 + 0.016 MB = **12.52 MB**

### Заполнение кэша:
1. **При запросе данных:**
    - Проверять кэш перед выполнением запроса.
    - Если данные не найдены, выполнить запрос к базе данных, затем записать результат в кэш.

2. **Обновление кэша:**
    - При обновлении базы данных (новые покупки, отзывы) обновлять соответствующие ключи в кэше.
    - Использовать механизм TTL (Time-To-Live) для автоматического удаления устаревших данных.

### Узкие Места и Решения

**1. Топ недельных продаж:**
- **Узкое место:** Запросы могут стать медленными, если таблица `TopSellers` содержит большой объем данных.
- **Решения:**
    - Индексировать таблицу по `week_start` и `week_end`.
    - Использовать кэширование недельных топов, как показано выше.

**2. Лучшие игры за год:**
- **Узкое место:** Запросы могут замедлиться с увеличением количества покупок.
- **Решения:**
    - Добавить индексы по `purchase_date` в таблице `Purchases`.
    - Создать материализованные представления для часто используемых агрегатов.
    - Использовать кэширование лучших игр за год.

**3. Список тегов:**
- **Узкое место:** Запросы по тегам могут замедляться при большом количестве тегов.
- **Решения:**
    - Индексировать таблицу `Tags` по `name`.
    - Использовать кэширование полного списка тегов.

**4. Информация об игре:**
- **Узкое место:** Запросы по отдельным играм могут замедляться при большом количестве игр.
- **Решения:**
    - Индексировать таблицу `Games` по `game_id` и `name`.
    - Использовать кэширование информации об игре.

**5. График активности пользователей:**
- **Узкое место:** Запросы к таблице `ConcurrentPlayers` могут быть медленными из-за большого количества записей.
- **Решения:**
    - Индексировать таблицу по `game_id` и `timestamp`.
    - Использовать кэширование результатов агрегационных запросов к `ConcurrentPlayers`.

### Заключение

1. **Тип Базы Данных:** Реляционная база данных выбрана из-за сложных запросов, нормализации данных и поддержки ACID-транзакций.

2. **Структура Базы Данных:**
    - Разработана структура базы данных с основными таблицами: `Users`, `Games`, `Tags`, `GameTags`, `Purchases`, `Reviews`, `ConcurrentPlayers`, `TopSellers`, `BestOfYear`.

3. **Размер Базы Данных и Кэша:**
    - Общий размер базы данных: **4.79 ГБ**
    - Общий размер кэша: **12.52 MB**

4. **Узкие Места и Решения:**
    - Идентифицированы потенциальные узкие места и предложены решения для их оптимизации:
        - Кэширование наиболее запрашиваемых данных