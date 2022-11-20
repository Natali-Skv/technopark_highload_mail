# Проектирование веб-сервиса

## [Методические указания](https://github.com/init/highload/blob/main/homework_architecture.md)
---

## Содержание

* ### [Тема и целевая аудитория](#тема_и_ца)
* ### [Расчет нагрузки](#рассчет_нагрузки)
* ### [Логическая схема](#логическая_схема)
* ### [Физическая схема](#физическая_схема)
* ### [Технологии](#технологии)
* ### [Схема проекта](#схема_проекта)
* ### [Список серверов](#список_серверов)
* ### [Источники](#источники)

---

## 1. <a name="тема_и_ца">Тема и целевая аудитория</a>

#### :white_medium_small_square: тип сервиса: почта

#### :white_medium_small_square: о сервисе: сервис по пересылке и получению электронных сообщений между пользователями интернета

#### :white_medium_small_square: mvp:

* регистрация
* авторизация
* просмотр списка входящих писем (со статусами прочитанно/не прочитанно)
* просмотр списка исходящих писем
* просмотр спама
* чтение письма
* отправка письма
* скачивание вложения из письма
* возможность прикрепить вложение
* удаление письма
* отметка письма как спам, внесение отправителя в спам-список
* удаление спам-писем по истечении 30 дней
* редактирование личного профиля
* удаление почтового ящика
* просмотр всех отправленных вложений (во всех письмах)
* удаление вложения
  
#### :white_medium_small_square: Целевая аудитория

* Сервис ориентирован на жителей РФ.
* Размер целевой аудитории [[1]](#источник_1)

***таблица 1***
| метрика                             | значение   |
|-------------------------------------|------------|
| месячная аудитория (кроссдевайс)    | 16 млн     |
| дневная аудитория (кроссдевайс)     | 6 млн      |
| среднее время (кроссдевайс) в месяц | 5 ч 46 мин |

---

## 2. <a name="рассчет_нагрузки">Расчет нагрузки</a>

### 2.1 <ins>Продуктовые метрики</ins>

#### :white_medium_small_square: Размер целевой аудитории [[1]](#источник_1)

  | метрика                             | значение   |
  |-------------------------------------|------------|
  | месячная аудитория (кроссдевайс)    | 16 млн     |
  | дневная аудитория (кроссдевайс)     | 6 млн      |
  | среднее время (кроссдевайс) в месяц | 5 ч 46 мин |

#### :white_medium_small_square: Предположение о среднем размере хранилища пользователя

* **профиль пользователя**

  ***таблица 2***

  | поле                                     | ~объём
  |------------------------------------------|  ---------
  | id                                       | **4 байт**
  | почтовый адрес                           | 20 сиволов (UTF-8) = **20 байт**
  | почтовый адрес для восстановления пароля | 20 сиволов (UTF-8) = **20 байт**
  | номер телефона для восстановления пароля | 11 символов = **11 байт**
  | хеш-пароля                               | **32 байт**
  | соль                                     | **8 байт**
  | имя                                      | 7 символов (UTF-8) = **14 байт**
  | фамилия                                  | 10 символов (UTF-8) = **20 байт**
  | фото профиля                             | 180 * 180, jpeg = **10 Кбайт**
  | пол                                      | **1 байт**
  | часовой пояс                             | **16 байт**
  | список отправителей спама                | 50 * 4 байт = **200 байт**
  | **ИТОГО**                                | **~10,4 Кб**

* **хранилище писем пользователя**
  Предоставим для каждого пользователя **10 Гб**, для хранения своих входящих и исходящих писем (анологичные ограничения
  присутствуют в Почта@mail.ru [[3]](#источник_3) = 8 Гб, gmail [[4]](#источник_4) = 15 Гб).

  - Средний объем письма без учета вложения — **0,1 Мб**.
  - **10%** входящих писем содержат вложения, а **90%** не содержат.
  - **40%** исходящих писем содержат вложения, а **60%** не содержат.
  - Средний размер вложения — **10 Мб**
  - Средний объем почтового ящика **5 Гб**.
  - Вложения письма будем хранить в хранилище отправляющего.
  - Исходящих писем в 5 раз меньше входящих.
  Таким образом средний пользователь сможет хранить `5 Гб / (40% * 10 Мб / 5) ~= 6 250 писем`.

#### :white_medium_small_square: Предположение о среднем количестве действий пользователя по типам в день

***таблица 3***

| действие                                      | количество в день |
|-----------------------------------------------|-------------------|
| загрузка главной страницы (входящие)          | 5                 |
| чтение письма                                 | 20                |
| отправка письма                               | 2                 |
| просмотр входящих писем (не первая страница)  | 1                 |
| скачивание вложения                           | 3                 |
| удаление письма                               | 3                 |
| просмотр исходящих писем                      | 1                 |
| просмотр спама                                | 0,5               |
| просмотр корзины                              | 0,5               |
| авторизация                                   | 0,05              |
| добавление отправителя в спам                 | 0,05              |
| просмотр всех отправленных вложений (во всех письмах)| 0,05       |
| удаление вложения                             | 0,05              |
| редактирование личного профиля                | 0,005             |
| регистрация                                   | 0                 |

### 2.2 <ins>Технические метрики</ins>

#### :white_medium_small_square: Размер хранилища в разбивке по типам данных (в Тб) - для существенных блоков данных

* Размер хранилища для информации о профилях пользователей
  При наличии 16 млн. месячной аудитории, будем считать, что всего зарегистрированных аккаунтов 20 млн.
  Аккаунты, которые не проявляли активность 2 года автоматически удаляются (аналогично яндекс почте [[5]](#источник_5)).
  Таким образом, на хранение информации профиля пользователей:
  `20 млн * 10,4 Кб = 0,19 Тб`
* Размер хранилища для вложений пользователей
  `20 млн * 5 Гб =  100 Пб`
* Размер хранилища для писем пользователей
  `0,1 Мб * 1 250 * 20 млд = 2 Пб`

#### :white_medium_small_square: Основной сетевой трафик

* **RPS в разбивке по типам запросов (запросов в секунду) - для основных запросов**
  Будем считать, что в пик нагрузки в 2 раза больше запросов, чем в среднем    
  Пусть в день у нас регистрируется 1000 аккаунтов
  `rps_avg = (кол-во действий на одного пользователя/сутки * 6 млн )/(24 * 60 * 60)`

  ***таблица 4***

  | действие                                     |кол-во на одного пользователя/сутки| rps (пик) | rps (avg) |
  |----------------------------------------------|-----------------------------------|-----------|-----------|
  | загрузка главной страницы (входящие)         | 5                                 | 694       | 347       |
  | чтение письма                                | 20                                | 2778      | 1389      |
  | отправка письма                              | 2                                 | 278       | 139       |
  | просмотр входящих писем (не первая страница) | 1                                 | 140       | 70        |
  | скачивание вложения                          | 3                                 | 417       | 208       |
  | удаление письма                              | 3                                 | 417       | 208       |
  | просмотр исходящих писем                     | 1                                 | 140       | 70        |
  | просмотр спама                               | 0,5                               | 70        | 35        |
  | просмотр корзины                             | 0,5                               | 70        | 35        |
  | авторизация                                  | 0,05                              | 7         | 3,5       |
  | добавление отправителя в спам                | 0,05                              | 7         | 3,5       |
  | просмотр всех отправленных вложений (во всех письмах)| 0,05                      | 7         | 3,5       |
  | удаление вложения                            | 0,05                              | 7         | 3,5       |
  | редактирование личного профиля               | 0,005                             | 0,7       | 0,4       |
  | регистрация                                  | 0                                 | 0,02      | 0,01      |

* **Рассчет сетевого потребления на статику.**
  Для рассчета трафика преходящегося на статику (JS,CSS,imgs) проведем небольшой эксперимент: определим объем такого трафика на *yandex почте* и *почте     mail.ru*.
  Отразим в таблице трафик, переданный по сети (transferred over network), то есть не будем учитывать статику, взятую из кеша браузера.
  Будем следовать более или менее классическому сценарию использования почтового сервиса:

        1. перейти на страницу входящих
        2. перейти на страницу одного письма
        3. вернуться на страницу входящих
        4. повторим пункты 2-3 10 раз для разных писем
        5. перейти на страницу исходящих
        6. перейти на страницу одного исходящего письма

  Так как страница входящих писем -- главная, то при открытии ее единственный раз загружается общая для всего web-приложения статика      (JS,CSS,логотипы,иконки итд).
  Отобразим результаты в таблице. Трафик от рекламных банеров не учитывался.

  ***таблица 5***

  | страница                                                   |yandex почта  | почта mail.ru |avg     |
  |------------------------------------------------------------|--------------|---------------|--------|
  | главная страница (входящие письма) disabled cache          | 3,4 Мб       | 3,4 Мб        | 3,4 Мб |
  | главная страница (входящие письма) enabled cache           | 350 Кб       | 400 Кб        | 375 Кб |
  | входящие письма, просмотр не первой страницы (по 30 писем) | 50 Кб        | 327 Кб        | 188 Мб |
  | одного письма(средний объем трафика среди 10 писем)        | 110 Кб       | 100 Кб        | 105 Кб |
  В качестве значений для рассчета сети нашего сервиса, возьмём среднее (столбез avg).
  Также будем считать, что 70% загрузок главной страницы происходит с использованием закешированной статики (строка 2).

* **Пиковое потребление в теченнии суток (в Гбит/с). Разбивка по существенным типам трафика (для выбора сетевых
  интерфейсов)**
  Список входящих, исходящих и спама будем выдавать постанично, по 30 писем на странице.
  Так как страницы списка входящих, исходящих и спама очень похожи по своей структуре, будем считать сетевое потребление на их загрузку одинаковым и равным значению колонки **avg** в таблице, приведенной в прошлом пункте.

  ***таблица 6***

  | действие                                    | rps (пик) |потребление на одну страницу (вложение)|пиковое потребление Гбит/с|
  |---------------------------------------------|-----------|---------------------------------------|--------------------------|
  | загрузка главной страницы (входящие)        | 694       | (3,4 Мб * 30%) + (375 Кб * 70%)       | 7,1  |
  | чтение письма                               | 2778      | 105 Кб                                | 2,3  |
  | отправка письма                             | 278       | 0,1 Мб + (40% * 10 Мб)                | 9,1  |
  | просмотр входящих писем (не первая страница)| 140       | 188 Кб                                | 0,2  |
  | скачивание вложения                         | 417       | 10 Мб                                 | 33,4 |
  | просмотр исходящих писем                    | 140       | 188 Кб                                | 0,2  |


* **Суммарный суточный (Гбайт/сутки) - разбивка по существенным типам трафика (опционально, для подсчета стоимости
  хостинга)**

  ***таблица 7***

  | действие                                    |суточное потребление, Гбайт/сутки|
  |---------------------------------------------|--------------------------------|
  | загрузка главной страницы (входящие)        | 38 340                         |
  | чтение письма                               | 12 420                         |
  | отправка письма                             | 49 140                         |
  | просмотр входящих писем (не первая страница)| 1 080                          |
  | скачивание вложения                         | 180 360                        |
  | просмотр исходящих писем                    | 1 080                          |
  | **ИТОГО**                                   | **282 420 Гбайт/сутки**        |

----

## 3. <a name="логическая_схема">Логическая схема</a>
<details>
  <summary>:black_circle: Развернуть краткое описание таблиц</summary>  

- **ATTACHMENTS**
  - user_references []INT -- список айдишников, пользователей, имеющих соответствующее письмо(имеющих доступ к этому вложению). Добавлено для того, чтобы когда пользователь нажимал "скачать вложение" по ID пользователя мы проверяли, есть ли текущий пользователь в user_references, то есть может ли этот пользователь скачать это вложение
- **WIDE_MAILS** таблица создана, чтобы не дублировать текст письма для каждого из получателей (их может быть несколько) и отправителя. В эту таблицу мы должны ходить только для получения 1 конкретного письма пользователя. С этой целью в таблицы USER_X_DIRECTORY_X_MAILS были добавлены поля Preview, date, theme_preview, has_attaches.
- **USER_X_MAILS** -- таблица всех писем, которые отображаются у пользователя. Эта таблица создана, чтобы в USER_X_DIRECTORY_X_MAILS можно было обозначать answer_to.
- **USER_X_DIRECTORIES** -- таблица директорий конкретного пользователя.
  - name -- имя директории, например: "входящие", "корзина", "спам", "исходящие".
  - new_mail_count -- количество писем, которые пользователь еще не видел. Для отображения значка с количеством новых писем около соответствующей директории.
- **USER_X_DIRECTORY_X_MAILS** -- семейство таблиц писем по пользователям, по папкам. То есть на каждого пользователя, по количеству папок. Данная таблица предназначена для страниц со списком писем.
- **USER_X_ATTACHES** -- таблица вложений, конкретного пользователя Х. Вложение считается принадлежащим пользователю, если он его прикрепил в исходящем письме. Такое вложение увеличивает storage_size данного пользователя.
Данная таблица создана для того, чтобы по запросу пользователь мог получить список своих вложений, а также удалить. При удалении каскадно удаляется и запись в ATTACHMENTS, также в соответствующем WIDE_MAIL соответствующий элемент attachment_sizes := -1. Далее, при отображении письма получателю, удаленное отправителем вложение, будет отмечено как "удалено отправителем"
</details>

[открыть в новой вкладке](https://raw.githubusercontent.com/Natali-Skv/technopark_highload_mail/master/imgs/logic_db.png)
![](https://github.com/Natali-Skv/technopark_highload_mail/blob/master/imgs/logic_db_100.png)

----

## 4.<a name="физическая_схема">Физическая схема</a>

|данные            |количество записей      | количество памяти        |пик ips|пик ops|
|------------------|------------------------|--------------------------|-------|-------|
|SESSIONS          |3 * 20 млн = 60 млн     |164 Б * 60 млн = 9,8 Гб   |7,002  |4 948  |
|USERS             |20 млн                  |478 Б * 20 млн = 9,6 Гб   |119    |7      |
|WIDE_MAILS        |20 млн * 1 250 = 25 млд |100 Кб * 25 млд = 2 Пб    |278    |2 778  |
|ATTACHMENTS       |500 * 20 млн = 10 млд   |164 Б * 60 млн = 1 Тб     |114    |417    |
|Файлы вложений    |500 * 20 млн = 10 млд   |10 Мб * 10 млд = 100 Пб   |114    |417    |
|Файлы аватарок    |20 млн                  |20 млн * 10 Кб = 200 Гб   | 0,7   |23 400 |
|USER_X_* набор    |20 млн                  |2,26 Мб * 20 млн = 45,2 Tб|18 520 |17 037 |

<details>
  <summary>:black_circle: SESSIONS. Развернуть пояснение</summary>  
  
|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|user_id|INT|4 Б|
|email|VARCHAR|20 сиволов (UTF-8) = 20 Б|
|session_key|VARCHAR|128 сиволов (UTF-8) = 128 Б|
|timezone|VARCHAR|4 сивола (UTF-8) = 4 Б|
|expired|DATE|4 Б|
|**Итого**||**164 Б**|
  
* Пусть каждый пользователь залогинен на 3-х девайсах.
* Будем очищать таблицу от протухших сессий 1 раз в день. Изначально установим
* ips -- `rps авторизации + rps регистрации + rps на выход` из *таблицы 4*
* ops -- `сумма rps запросов требующих проверку сессии` (все запросы из *таблицы 4*, кроме авторизации и регистрации)  
</details>

<details>
  <summary>:black_circle: USERS. Развернуть пояснение</summary>  

|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|email|VARCHAR|20 сиволов (UTF-8) = 20 Б|
|name|VARCHAR|7 сиволов (UTF-8) = 14 Б|
|surname|VARCHAR|10 сиволов (UTF-8) = 20 Б|
|password|[]BYTE|128 Б|
|password_salt|[]BYTE|32 Б|
|restore_email|VARCHAR|20 сиволов (UTF-8) = 20 Б|
|restore_phone|VARCHAR|11 сиволов (UTF-8) = 11 Б|
|avatar|VARCHAR|20 сиволов (UTF-8) = 20 Б|
|sex|CHAR|1 символ (UTF-8) = 1 Б|
|timezone|VARCHAR|4 сивола (UTF-8) = 4 Б|
|spam_list|[]INT|50 * 4 Б = 200 Б|
|storage_size|INT|4 Б|
|**Итого**||**478 Б**|

* ips -- `rps на редактирование профиля + rps на отправление письма с вложением + rps на удаление вложения` из *таблицы 4*. При запросе на отправку письма с вложением необходимо увеличить user storage_size
* ops -- `rps авторизации + rps регистрации` из *таблицы 4*
</details> 

<details>
  <summary>:black_circle: WIDE_MAILS. Развернуть пояснение</summary>  

|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|theme|VARCHAR|50 символов UTF-8 = 100 Б|
|text|VARCHAR|0,1 Мб|
|attachment_ids|[]INT|2 * 4 Б = 8 Б|
|attachment_sizes|[]INT|2 * 4 Б = 8 Б|
|**Итого**||**100 Кб**|

* На каждое отправленное письмо приходится одна запись в этой таблице, независимо от количества получателей  
* Пусть на каждого пользователя приходится ~1 250 писем (по количеству отправленных)   
* ips -- `rps на отправление письма` из *таблицы 4*
* ops -- `rps на чтение письма` из *таблицы 4*
</details>

<details>
  <summary>:black_circle: ATTACHMENTS. Развернуть пояснение</summary>  
  
|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|size|INT|4 Б|
|name|VARCHAR|20 символов UTF-8 = 20 Б|
|path|VARCHAR|50 символов UTF-8 = 50 Б|
|user_references|[]INT|5 * 4 Б = 20 Б|
|**Итого**||**98 Б**|

* Размер хранилища среднего пользователя 5Гб, то есть `5 Гб/10 Мб = 500 вложений`   
* ips -- `rps на отправление письма с вложением + rps на удаление вложения = 278*40% + 7 = 114` из *таблицы 4*
* ops -- `rps на скачивание вложения = 417` из *таблицы 4*
</details>

<details>
  <summary>:black_circle: Файлы вложений. Развернуть пояснение</summary>  
* Размер хранилища среднего пользователя 5Гб, то есть `5 Гб/10 Мб = 500 вложений`   
* ips -- `rps на отправление письма с вложением + rps на удаление вложения = 278*40% + 7 = 114` из *таблицы 4*
* ops -- `rps на скачивание вложения = 417` из *таблицы 4*

</details>

<details>
  <summary>:black_circle: Файлы аватарок. Развернуть пояснение</summary>  
* ips -- `rps на регистрацию + rps на редактирование профиля`
* ops -- `(rps на загрузку главной страницы + rps на входящие + rps на исходящие + rps на просмотр спама + rps на просмотр корзины)*30*70%`. 30 -- количество писем на странице. Пусть 30% аватарок берется из кеша.
</details>

<details>
  <summary>:black_circle: USER_X_* набор таблиц. Развернуть пояснение</summary>  
USER_X_* набор таблиц -- USER_X_MAILS, USER_X_DIRECTORIES, USER_X_DIRECTORY_X_MAILS, USER_X_ATTACHES.   
  Это персональный набор таблиц для каждого пользователя.
       
  <details>
  <summary>:black_medium_small_square: USER_X_MAILS. Развернуть пояснение</summary>  
    
|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|status|STATUS|4 Б|
|directory|VARCHAR|20 символов UTF-8 = 20 Б|
|**Итого**||**28 Б**|
  
  * В этой таблице хранится информация об всех полученных и отправленных письмах данного пользователя. 
  * Пусть у пользователя 7500 писем
  * В эту таблицу мы ходим при запросе на конкретное письмо, которое содержит *answer_to*. Пусть 20% писем пользователей содержат answer_to
  * ipd -- `rpd на отправку письма`
  * opd -- `rpd на прочтение письма * 20%` *из таблицы 4*
  
|таблица          |количество записей      | объем таблицы        | input/day| output/day|
|-----------------|------------------------|----------------------|-------|-------|
|USER_X_MAILS     | 7500                   |7500 * 28 Б = 0,21 Мб |2 |4|
</details>

  <details>
  <summary>:black_medium_small_square: USER_X_DIRECTORIES. Развернуть пояснение</summary>  

|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|name|VARCHAR|20 символов UTF-8 = 20 Б|
|mail_count|INT|4 Б|
|**Итого**||**28 Б**|
   
  * у пользователя 4 директории (входящие, исходящие,спам,корзина)
  * ipd -- 0, запись в эту таблицу происходит 1 раз, при регистрации пользователя
  * opd -- `rpd на загрузку главной страницы + rpd на входящие + rpd на исходящие + rpd на просмотр спама + rpd на просмотр корзины` *из таблицы 4*
  
|таблица           |количество записей      | объем таблицы         | input/day| output/day|
|------------------|------------------------|-----------------------|-------|-------|
|USER_X_DIRECTORIES| 4                      |4 * 28 Б = 112 Б |0 |8|
</details>
 
  <details>
  <summary>:black_medium_small_square: USER_X_DIRECTORY_X_MAILS. Развернуть пояснение</summary>   
  
|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|sendler_id|INT|4 Б|
|receiver_emails|[]VARCHAR|5*20 сиволов (UTF-8) = 100 Б|
|status|STATUS|4 Б|
|wide_mail|INT|4 Б|
|answer_to|INT|4 Б|
|theme_preview|VARCHAR|35 символов UTF-8 = 70 Б|
|preview|VARCHAR|35 символов UTF-8 = 70 Б|
|date|TIMESTAMP|4 Б|  
|has_attaches|BOOL|1 Б|
|**Итого**||**265 Б**|
     
  * у пользователя 4 директории (входящие, исходящие,спам,корзина), то есть 4 таблицы USER_X_DIRECTORY_X_MAILS
  * Пусть в день пользователь получает 22 письма.
  * ipd -- `rpd на отправку письма + rpd на получение письма + rpd на чтение письма + rpd на удаление письма`
  * opd -- `rpd на загрузку главной страницы + rpd на входящие + rpd на исходящие + rpd на просмотр спама + rpd на просмотр корзины + rpd на чтение письма + rpd на удаление письма` *из таблицы 4*
  
|набор таблиц            |количество записей      | суммарный объем таблиц         | input/day| output/day|
|------------------------|------------------------|--------------------------------|-------|-------|
|USER_X_DIRECTORY_X_MAILS| 7500                   |7500 * 265 Б = 2 Мб |47 |31|
</details>

   <details>
  <summary>:black_medium_small_square: USER_X_ATTACHMENTS. Развернуть пояснение</summary>   

|столбец|тип|объем|
|-------|---|-----|
|id|INT|4 Б|
|mail_id|INT|4 Б|
|size|INT|4 Б|
|name|VARCHAR|20 символов UTF-8 = 20 Б|
|path|VARCHAR|50 символов UTF-8 = 50 Б|
|date|TIMESTAMP|4 Б|  
|attach_id|INT|4 Б|
|**Итого**||**90 Б**|

* Размер хранилища среднего пользователя 5Гб, то есть `5 Гб/10 Мб = 500 вложений`   
* ipd -- `rpd на отправление письма с вложением + rpd на удаление вложения` из *таблицы 4*
* opd -- `rpd на скачивание вложения` из *таблицы 4*
  
|таблица            |количество записей  | объем таблицы       | input/day| output/day|
|-------------------|--------------------|---------------------|-------|-------|
|USER_X_ATTACHMENTS | 500                |500 * 90 Б = 0,05 Мб | 1 | 3 |
</details>
 
  
### размер sqlite-файла на одного пользователя 
|таблицы          | количество памяти |input/day|output/day|
|-----------------|-------------------|---------|----------|
|USER_X_*         | 2,26 Мб           | 50      | 46       |

</details>  



  

  | Данные   |Технология| Мотивация | Решающие свойства |
  |----------|--------------|--------|----------|
  | SESSIONS |**Tarantool**| Очень нагруженная таблица, в нее мы будем ходить почти на каждый запрос пользователя, чтобы проверить ключ сессии. Необходим быстрый доступ без возможности потери данных |хранение данных в RAM, cогласованность и сохранность данных в случае сбоя [[6]](#источник_6);|
  | USERS, ATTACHES, WIDE_MAILS | **PostgreSQL** |Данные хорошо ложатся на реляционную модель; нет full-scan запросов, только по id | реляционная модель; популярная технология; высокая поддержка; cогласованность и сохранность данных в случае сбоя;|
  | USER_X_MAILS, USER_X_DIRECTORIES, USER_X_DIRECTORY_X_MAILS, USER_X_ATTACHES | отдельная **SQLite** для каждого набора таблиц конкретного пользователя | Данные хорошо ложатся на реляционную модель; нужна возможность хранить отдельную БД для каждого пользователя, чтобы избежать крайне тяжелые full-scan запросы|реляционная модель; встраиваемая БД; популярная технология; высокая поддержка; cогласованность и сохранность данных в случае сбоя;|
  | Вложения | **S3** |очень большой объем данных; возможность сэкономить на покупке собственных серверов; возможность сэкономить засчет гибкой тарификации(cold data); |популярная технология; высокая поддержка; надежность, доступность;|
  | Аватарки пользователей | **FS** |небольшой объем данных, очень большое количество запросов на получение, крайне редкие запросы на обновление |популярная технология; высокая поддержка; надежность, доступность;|

  ### Индексы
  |ТАБЛИЦА: поле | Мотивация  |
  |----------------|------------|
  |SESSION:session_key|необходимо для всех запросов пользователя (кроме запросов на логин)|
  |USERS:email|необходимо для запросов на логин, отправку письма(поиск получателей)|
  |WIDE_MAILS:id|необходимо для быстрого поиска письма по id (запрос на чтение конкретного письма)|
  |ATTACHMENTS:id|необходимо для быстрого поиска вложения по id (запрос на скачивание вложения, удаление вложения отправителем соответствующего письма)|
  |USER_X_MAILS:id|необходимо для быстрого поиска папки письма (запрос на получение письма с непустым answer_to, в этом случае внизу письма отображается превью answer_to письма)|
  |USER_X_DIRECTORIIES:name|необходимо быстро по имени директории определять id (запрос на получение писем конкретной директории по ее имени)|
  |USER_X_DIRECTORY_X_MAILS:id|необходимо быстро находить письмо по id (запрос на получение новых писем, запрос на чтение конкретного письма, запрос на изменение статуса письма (прочитано/непрочитано), запрос на удаление письма)|
  |USER_X_DIRECTORY_X_MAILS:date|необходимо быстро сортировать письма по дате (запрос на получение писем определенной папки)|
  |USER_X_DIRECTORY_X_MAILS:status|необходимо быстро сортировать письма по статусу (запрос на получение писем определенной папки)|
  |USER_X_ATTACHMENTS:id|необходимо быстро получать вложение по id (запрос на удаление вложения отправителем)|
  |USER_X_ATTACHMENTS:size|необходимо быстро сортировать вложения по размеру(запрос на список вложений пользователя)|

[открыть в новой вкладке](https://raw.githubusercontent.com/Natali-Skv/technopark_highload_mail/master/imgs/store.png)
![](https://github.com/Natali-Skv/technopark_highload_mail/blob/master/imgs/store_100.png)

### Шардинг
**Tarantool**:
В нашем случае на Tarantool основная нагрузка -- SELECT-запросы. Обновление данных происходит только при запросе на выход из аккауна, выход со всех устройств, вход. Максимальное RPS ~ 5 000.
> Критическое значение RPS для запросов типа SELECT/INSERT/UPDATE/DELETE – 100 000.
> Если основная нагрузка генерируется SELECT-запросами, следует добавить slave-сервер и часть запросов обрабатывать на нем.[[8]](#источник_8)

Также данные, которые мы храним в Tarantool будут помещаться в `20 000 000 * 3 * 300 Б = 18Гб`
Таким образом на Tarantool можно не реализовывать шардинг.

**SQLite**:
Данные, расположенные в одном файле SQLite принадлежат одному конкретному пользователю. То есть мы можем сделать шарды по диапозонам id пользователей.
Если конкретный шард становится перегруженным, можно просто перенести файлы SQLite части диапозона пользователей на другой шард.

**PostgreSQL**:
В таблицы, расположенные в PostgreSQL, не будет full-scan запросов, что упрощает шардинг.
В выбранной схеме данных, все данные можно разделить на шарды по диапозонам user_id.
Записи в WIDE_MAILS будут принадлежать конкретному шарду, если id отправителя этого письма принадлежит диапозону user_id данного шарда.
Записи в ATTACHMENTS будут принадлежать конкретному шарду, если id отправителя соответствующего письма принадлежит диапозону user_id данного шарда.
При шардинге следует учесть вероятные проблемы с балансировкой нагрузки между шардами. Для этого на одном сервере создается сразу несколько шардов (например 100). Тогда если сервер начинает переполняться, половину шардов можно перенести на другой сервер.

## 5. <a name="технологии">Технологии</a>

Сводная таблица: технология, область примерения, мотивационная часть

## 6. <a name="схема_проекта">Схема проекта</a>

Сущности:
- пользователь
- письмо
- вложение

Схема взаимодействия сервисов внутри проекта (картинка). Должно быть описано как ходят потоки данных, как устроена
балансировка (внутренняя и внешняя).

## 7. <a name="список_серверов">Список серверов</a>

Таблица с конфигурациями, количеством серверов и сервисами расположенными на них. В случае использования kubernetes или
иной виртуализации, список контейнеров с аллокацией ресурсов.

----

## 8. <a name="источники">Источники</a>

<a name="источник_1"> 1. https://radar.yandex.ru/yandex?month=2022-07 </a>
<a name="источник_2"> 2. https://www.lifewire.com/what-is-the-average-size-of-an-email-message-1171208#:~:text=The%20average%20size%20of%20an,text%20or%20about%2037.5%20pages.</a>
<a name="источник_3"> 3. https://help.mail.ru/cloud_web/faq/free </a>
<a name="источник_4"> 4. https://support.google.com/googleone/answer/9004014?hl=en </a>
<a name="источник_5"> 5. https://yandex.ru/support/mail/freezing.html </a>
<a name="источник_6"> 6. https://www.bigdataschool.ru/wiki/tarantool </a>
<a name="источник_7"> 7. https://www.tarantool.io/ru/doc/latest/concepts/sharding/ </a>
<a name="источник_8"> 7. https://dist.tarantool.io/pdf/Tarantool.2.1.1.ru.pdf </a>
