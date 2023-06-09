# Обобщённый подход и драйверы
Для работы с SQL в Go есть пакет **database/sql.** Чтобы получить доступ к базе данных, используется структура **sql.DB.**


> ***Важно:*** 
>
> + **sql.DB** — это **интерфейс** к драйверу базы данных.
> + **Драйвер** — реализация интерфейса, который содержит логику работы с конкретной СУБД. 
>
> Драйвер спроектирован так, что в коде не нужно вручную работать с базой — можно пользоваться только методами sql.DB. Код не зависит от драйвера, и его легко сменить.
>  


Структура **sql.DB** управляет пулом соединений. Можно сконфигурировать разные политики: либо установить фиксированное количество соединений, либо настроить динамическое открытие новых соединений (при повышенной нагрузке) и закрытие неиспользуемых соединений по тайм-ауту.

С помощью драйвера можно работать со многими БД, например: 
SQLite, MySQL, MS SQL, PostgreSQL, Apache Avatica, Oracle, ClickHouse.

## Импорт драйвера
Будем использовать **sqlite3** — это простая, не требующая дополнительных зависимостей и одновременно функциональная БД. Её легко установить на большинстве операционных систем. В неё легко импортировать csv-файл.

Особенность пакета **database/sql** в том, что он написан обобщённо, все привязки к движку вынесены в драйвер. Вы можете выбрать любую реляционную базу данных и разбирать примеры в ней. 

В следующих уроках и для выполнения инкремента будем использовать **PostgreSQL**. 

Движок **Postgre** мощнее, в нём больше дополнительных возможностей. Go-код и SQL-декларации будут продолжать работать (за исключением специфических особенностей движка и драйвера, документированных в пакетах драйверов).

### sqlite3

Сначала установим драйвер. Можно выбрать любой, например один из самых популярных: github.com/mattn/go-sqlite3.

```bash
go get github.com/mattn/go-sqlite3
```

Затем импортируем интерфейс **sql.DB** и драйвер go-sqlite3 в проект:

```Go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"
) 
```

>Обратите внимание: пакет **go-sqlite3** импортирован анонимно. Не получится обращаться напрямую к go-sqlite3. Внутри пакет зарегистрирует себя самостоятельно и будет доступен для использования через **sql.DB**.


> **Вопрос**: Если нас не интересует пространство имён драйвера github.com/mattn/go-sqlite3, зачем вообще его импортировать? Что происходит при этом импорте?
**Ответ**:Отрабатывают init()-функции пакета драйвера.


Всё готово для работы. Получили ресурс — доступ к базе данных. Используем конструкцию отложенного исполнения **defer**, чтобы закрыть соединение и освободить ресурс.

```go
func main() {
    db, err := sql.Open("sqlite3", "db.db")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // работаем с базой
    // ...
    // можем продиагностировать соединение

    ctx, cancel := context.WithTimeout(ctx, 1*time.Second)
    defer cancel()
    if err = db.PingContext(ctx); err != nil {
        panic(err)
    }
    // в процессе работы
}
```

+ С помощью функции **Open** открываем соединение с базой данных. Первым аргументом указываем 
1.  **драйвер** (MySQL, SQLite, PostgreSQL и др.). 
2.  **Второй аргумент — это строка для драйвера**, она определяется самим драйвером. В нашем случае второй аргумент — путь до файла базы данных.

С помощью **defer db.Close()** закрываем соединение с базой, после того как функция закончит выполнение. Не нужно открывать и закрывать соединение в каждой функции.** Open/Close** разработаны таким образом, чтобы работать постоянно и долго, а не краткосрочно.


На самом деле соединение будет установлено только тогда, когда выполнится первый запрос к базе данных. Проверить соединение можно методом **DB.PingContext()** — он контролирует состояние и обновляет в случае необходимости. Метод принимает контекст и возвращает ошибку, если соединение утеряно и восстановить его невозможно.

```
func (db *DB) PingContext(ctx context.Context) error 
```

> Некоторые драйверы требуют cgo («си-гоу»), а некоторые нет. Если режим cgo включён, можно напрямую исполнять C-код, а результат использовать в Go-коде.


## Основные драйверы

Если открыть список доступных драйверов(https://github.com/golang/go/wiki/SQLDrivers), можно насчитать более пятидесяти. И для каждой из основных баз данных —  SQLite, MySQL (включая MariaDB, Google Cloud SQL и другие), PostgreSQL — есть минимум два или три популярных.

>**Важно** понимать, что через какое-то время может выйти новый драйвер, который будет поддерживать больше возможностей и будет ещё быстрее. При написании проекта обязательно смотрите полный список доступных драйверов.

#### Driver для sqlite
SQLite	https://pkg.go.dev/modernc.org/sqlite	Не требует CGO. Написан на чистом Go.

#### Подготовка базы данных
В следующих уроках вы научитесь получать данные из таблицы, применять SQL-выражения (стейтменты) и транзакции. Чтобы наглядно показать разницу в работе со стейтментами и без них, подготовим базу данных. Использовать будем SQLite.
Для работы возьмём готовый набор данных о популярных в США видеороликах на YouTube. Скачайте csv-файл здесь(https://code.s3.yandex.net/go/USvideos.csv  ).

Посмотрите, из чего состоит файл. Так как он большой, удобно пользоваться стандартными консольными утилитами head и tail. Команда head покажет, что находится в начале файла, а tail — что в конце.

```
head USvideos.csv 
```


## Запросы к базе данных

Запросы можно выполнять двумя функциями:

1. **QueryContext()**, когда сервер возвращает записи — например, когда отправляем SQL-запрос SELECT;
2. **ExecContext()**, когда сервер SQL ничего не возвращает — например, когда пишем INSERT.

Посчитаем количество всех записей в базе данных:

```go
// в прошлом уроке вы уже видели, как открыть соединение с базой
db, err := sql.Open("sqlite3", "db.db")
if err != nil {
    panic(err)
}
defer db.Close()

// готовим контейнер
var id int64

// делаем запрос
row := db.QueryRowContext(ctx, "SELECT COUNT(*) as count FROM videos")

// разбираем результат
err := row.Scan(&id)
if err != nil {
    log.Fatal(err)
}

fmt.Println(id)

```

Итак, порядок действий:
1. Выполняем запрос к базе данных. Так как результатом всегда будет одна запись, можно использовать метод **QueryRowContext()** вместо **QueryContext()**.
2. Вызываем метод **Scan()**, который ассоциирует результат с переменной. В случае нескольких значений указываем их последовательно — так же, как выводим в SQL.
3. В случае ошибки — выводим сообщение об ошибке.

Многие методы пакета database/sql принимают первым аргументом **ctx context.Context**:

```go
func (db *DB) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

!!! Контекст позволяет ограничить по времени или прервать слишком долгие или уже не нужные операции с базой данных, назначить для них дедлайн или тайм-аут. Вот пример использования контекста:
```go
// конструируем контекст с 5-секундным тайм-аутом
// после 5 секунд затянувшаяся операция с БД будет прервана
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)

// не забываем освободить ресурс
defer cancel()

// обращаемся к БД
row := db.QueryRowContext(ctx, "SELECT COUNT(*) as count FROM videos") 
```

> Go — язык преимущественно бэкенд-разработки. Поэтому запросы к БД вы скорее всего будете делать из обработчиков (http.Handler) ваших серверов. В таких случаях стоит наследовать контекст HTTP-запроса:

```go
func MyHandler(w http.ResponseWriter, r *http.Request) {
  
    // наследуем контекcт запроса r *http.Request,
    // оснащая его Timeout
    ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
    
    // не забываем освободить ресурс
    defer cancel()
    
    // делаем обращение к db в рамках полученного контекста
    rows, err := db.QueryContext(ctx, "SELECT something")
    
    // отрабатываем запрос
    // ...
} 
```

> **Важно**: методы Query() или Exec() (без Context) под капотом используют context.Background. Контекст в операциях с БД присутствует всегда. Если вы пользуетесь инструментарием context.Context, нужно применять методы QueryContext() и ExecContext(), позволяющие установить контекст явно.

**Задание 1 из 6** Как передаётся SQL-инструкция методу QueryContext()?
**Правильный ответ** . Обычным SQL-синтаксисом в виде строки string.
Да. Строковый аргумент query принимает инструкцию обычным SQL-синтаксисом.


## Как работает Scan
Когда вызывается метод **Scan()**, пакет **database/sql** автоматически преобразует значения в тот тип переменных, который был передан.

Представьте, что есть атрибут **is_disabled** и это строковое поле **VARCHAR(10)** или TEXT — со значениями **true, T, TRUE, 1 или 0, false, F**. 

Если передадим ссылку на строку, после этого нужно будет вызвать метод **ParseBool()**, чтобы сделать преобразование или вернуть ошибки. И так для каждой переменной.

Если же сразу передадим ссылку на переменную типа bool, пакет database/sql сам сделает преобразования и вернёт переменную (или ошибку в случае невозможности). **Код станет чище и компактнее.**


Метод Scan() поддерживает: 

 + Многие встроенные типы языка Go, например: *int64, *float64, *bool, *[]byte, *string, *time.Time, nil. Заметьте, что перечислены референсные типы, то есть указатели на контейнеры определённых типов, куда Scan() будет помещать значения.

+ Специфические типы пакета **database/sql**, например ***sql.NullString.**
  
+ Любые пользовательские типы данных, реализующие интерфейсы **database/sql**.**Scanner** и **database/driver.Valuer**.

Остановимся на типе **sql.NullString**. Пакет database/sql декларирует ещё sql.NullFloat64, sql.NullByte и подобные. Названия типов начинаются с **Null**. Так сделано потому, что стандарт SQL позволяет присваивать значение NULL полям самых разных таблиц


**Задание 2**
Как быть, если в строке SQL-записи, к которой применён метод row.Scan(), несколько колонок?
**ответ** : Методу Scan() нужно передать столько аргументов и такого типа, 
чтобы в них можно было поместить все поля полученной из БД строки.


## Выбор нескольких строк
Снова вернёмся к примеру. Многие запросы к БД возвращают не одну, а несколько записей SQL-таблицы. Предположим, надо выбрать первые **X** самых популярных роликов, где **X — количество**.

```sql
SELECT video_id, title, views from videos ORDER BY views LIMIT X; 
```

Определим структуру **Video**, в которую функция Scan будет записывать значения. 

```go
// Video — структура видео.
type Video struct {
    Id    string
    Title string
    Views int64
}

// limit — максимальное количество записей.
const limit = 20
```

Запрос к базе данных:
```go
func QueryVideos(ctx context.Context, db sql.DB, limit int) ([]Video, error) {
    videos := make([]Video, 0, limit)

    rows, err := db.QueryContext(ctx, "SELECT video_id, title, views from videos ORDER BY views LIMIT ?", limit)
    if err != nil {
        return nil, err
    }

    // обязательно закрываем перед возвратом функции
    defer rows.Close()

    // пробегаем по всем записям
    for rows.Next() {
        var v Video
        err = rows.Scan(&v.Id, &v.Title, &v.Views)
        if err != nil {
            return nil, err
        }

        videos = append(videos, v)
    }

    // проверяем на ошибки
    err = rows.Err()
    if err != nil {
        return nil, err
    }
    return videos, nil
}
```

Что изменилось по сравнению с первым примером?
+ Используем **defer** **rows.Close()** — это обязательно. Пока доступны результаты, соединение активно и не закроется ещё долгое время. Если не закрыть, соединение будет «грязным». За ним будет закреплён курсор с данными, и драйверу придётся закрыть соединение и создать новое, что требует ресурсов.

+ Проходим по всем записям методом **rows.Next()** до тех пор, пока не пройдём все доступные результаты.

+ После цикла проверяем записи на потенциальные ошибки методом **rows.Err()**. Метод возвращает ошибку, которая могла произойти в момент получения новой порции данных (новой итерации цикла** rows.Next()**). Ошибка заставит итерацию прекратиться, и алгоритм не сможет проверить её другим способом. Пример такой ошибки — разрыв сетевого соединения с сервером базы данных в процессе получения результатов запроса.

+ db.Query("SELECT video_id, title, views from videos ORDER BY views LIMIT ?", 20) — запрос выполняется с помощью специального встроенного форматирования, с помощью символа ?. 
  
> **Важно:** Никогда не используйте в запросе **fmt.Sprintf**, этот способ небезопасен, он приводит к SQL-инъекциям.


> **Важно:** Параметр **?** будет разным для разных типов баз данных:
> 1. Для MySQL, 
> 2. SQLite3 и 
> 3. MS SQL это будет **?**. 
> 
> Для **PostgreSQL** — **$N**, **число**. 
> 
> Для Oracle — **:param**.



## Расширение поддерживаемых типов
Как уже говорилось выше, Scan поддерживает ограниченный набор типов переменных: строки, числа, булевы значения и байты. А что, если нужно использовать массивы или собственные структуры?

Допустим, в базе данных есть теги, представленные одной строкой. В таком формате использовать теги невозможно. Было бы здорово хранить их в массиве строк.

Пример тега:
**last week tonight trump presidency|last week tonight donald trump|john oliver trump|donald trump.**

```
Tags []string
```

Чтобы сканировать теги, реализуем интерфейсы database/driver.Valuer и database/sql.Scanner.

Для этого нужно определить методы:
+ Value() — для записи в БД и приведения данных к простому типу;
+ Scan() — для чтения из БД и приведения данных к сложным типам и структурам Go.

Наследуем функцию Value():
```go
// Value — функция для тегов, которая наследуется из пакета database/sql.
func (tags Tags) Value() (driver.Value, error) {
    // преобразуем []string в string
    if len(tags) == 0 {
        return "", nil
    }
    return strings.Join(tags, "|"), nil
} 
```

Важно преобразовать значения в один из поддерживаемых типов: int64, float64, bool, []byte, string, time.Time, nil.

Теперь можно записывать значения в базу. Но нужно и считывать данные — для этого наследуем функцию **Scan()**.

```go
func (tags *Tags) Scan(value interface{}) error {
    if value == nil {
        *tags = Tags{}
        return nil
    }

    sv, err := driver.String.ConvertValue(value)
    if err != nil {
        return fmt.Errorf("cannot scan value. %w", err)
    }

    v, ok := sv.(string)
    if !ok {
        return errors.New("cannot scan value. cannot convert value to string")
    }

    *tags = strings.Split(v, "|")

    return nil
} 
```

Если value — это поле nil, возвращаем пустой массив.

Функцией **driver.String.ConvertValue(value)** пытаемся конвертировать значение в строку. Если не получится, функция вернёт ошибку. Открыв функцию driver.**String.ConvertValue(value)**, увидим, что она никогда не возвращает ошибку.
После этого указываем, чему должен равняться атрибут tags, функцией:  __*tags__ = **string.Split(v, "|")**.


```go
func (stringType) ConvertValue(v interface{}) (Value, error) {
    switch v.(type) {
    case string, []byte:
        return v, nil
    }
    return fmt.Sprintf("%v", v), nil
}
```

Можно ли пропустить проверку ошибок для строк? Нет. В будущем пакет может измениться.
Теперь вы можете делать запросы к БД в Go-программе и знаете, что:


+ декларация запроса пишется на стандартном языке SQL и передаётся методам **Query()** в виде аргумента **string**;
+ полученные в результате запроса строки перебирает метод Rows.Next();
+ разбор строки в целевую структуру Go делает метод Row.Scan();
+ конверсия между SQL и Go для базовых типов уже реализована в пакете database/sql;
+ пакет предоставляет интерфейс для конверсии пользовательских типов.
Предлагаем закрепить знания на практике.

**Задание 6**
В базе есть атрибут trending_date. Его значения могут быть такими:
18.31.05,
17.29.12,
05.01.12.

```go
type Trend struct{
    T time.Time
    Count int
}
func (db *sql.DB) TrendingCount() []Trend, err {
    // проверяем на всякий случай
    if db == nil {
        return nil, errors.New("You haven`t open the database connection")
    }

    rows, err := db.Query("SELECT trending_date, COUNT(trending_date) FROM videos GROUP BY trending_date")

    if err != nil {
        return nil, err
    }

    trends := make([]Trend, 0)
    date := new(string)

    for rows.Next() {
        trend := Trend{}
        err = rows.Scan(date, &trend.Count)
        if err != nil {
            return nil, err
        }
        
        if trend.T , err := time.Parse("06.02.01", *date); err != nil {
            return nil, err
        }
        trends = append(trends, trend)
    }
    return trends, nil
}
```

+ PostgreSQL поддерживает стандартные операции SQL и совместим с database/sql.
+ MySQL поддерживает стандартные операции SQL и совместим с database/sql.
+ MariaDB поддерживает стандартные операции SQL и совместим с database/sqL/

НЕ ПОДДЕРЖ
+ MongoDB — документоориентированная СУБД со специфичным языком запросов.
+ ArangoDB — графовая СУБД и не поддеррживает операции SQL.
+ Redis — key-value хранилище со специфичным протоколом взаимодействия.


