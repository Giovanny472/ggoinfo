# Запись в базу данных, стейтменты и транзакции

В предыдущем уроке вы заполняли базу данных с помощью встроенных функций sqlite3. Теперь расскажем, как заполнять её на Go, используя функции пакета database/sql. Это нужно для того, чтобы вы научились применять Go не только для операций SELECT, но и для UPDATE, INSERT, DELETE, а также работать с подготовленными SQL-инструкциями и транзакциями.

## Запросы вставки данных, обновления и удаления
Сначала вставим записи из csv-файла вручную:
1. Загрузим csv-файл в приложение.
2. Сделаем запрос INSERT.

пример. Теперь откроем файл и создадим слайс из всех записей:

```go
func main() {
    // инициируем контекст 
    ctx := context.Background()
    // открываем соединение с БД
    db, err := sql.Open("sqlite3", "db.db")
    if err != nil {
        log.Fatal(err)
    }
    // открываем csv-файл
    file, err := os.Open("USvideos.csv")
    if err != nil {
        log.Fatal(err)
    }
    // конструируем Reader из пакета encoding/csv
    // он умеет читать строки csv-файла
    csvReader := csv.NewReader(file)
    // читаем записи из файла в слайс []Video вспомогательной функцией
    videos, err := readVideos(csvReader)
    if err != nil {
        log.Fatal(err)
    }
    // записываем []Video в базу данных в инициированном контексте
    // тоже вспомогательной функцией
    err = insertVideos(ctx, db, videos)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Всего csv-записей %v\n", len(videos))
}
```

Вспомогательная функция для чтения csv:
```
func readVideos(r *csv.Reader) ([]Video, error)
```

**csv.Reader** методом **Read()** возвращает одну строку csv-файла в виде []string. Но структура Video декларирована с полями других типов, не **string**. Надо разобрать записи и привести их к нужным типам.

**Выполнять запросы, которые не возвращают записи, лучшего всего функцией ExecContext().**
```go
result, err := db.ExecContext(сtx, "INSERT INTO videos(title, description, views, likes) VALUES(?,?,?,?)", v.Title, v.Description, v.Views, v.Likes) // так правильно

resultWithRows, err := db.QueryContext(ctx, "INSERT INTO videos(title, description, views, likes) VALUES(?,?,?,?)", v.Title, v.Description, v.Views, v.Likes) // так плохо
```

Как использовать SQL-декларации в методах Exec() и Query?
ответ: Так же, как и в консольном клиенте: в виде string.


## Использование переменных в запросе
Представьте, что нужно выполнить **DELETE-запро**с: удалить опубликованные в определённый день видео, у которых были отключены или включены комментарии.
Пример такой операции:

Пример такой операции:
```go
DELETE FROM videos WHERE PublishTime BETWEEN `2018-01-01 00:00:00` AND `2018-01-31 23:59:59 AND CommentDisabled = 1`
```

На Go этот запрос можно написать так:
```go
    _, err := db.ExecContext(сtx, "DELETE FROM videos WHERE PublishTime BETWEEN ? AND ? and CommentsDisabled = ?;",
        fromDate,
        toDate,
        isCommentDisabled,
    )
```

Запрос с использованием **?** плохо читается, особенно если в нём больше трёх аргументов.

```go
func Named(name string, value interface{}) NamedArg
```

Тогда получится переделать запрос:
```go
    _, err := db.ExecContext(сtx,
        "DELETE FROM videos WHERE PublishTime BETWEEN @start AND @end and CommentsDisabled = @isDisabledComment;",
        sql.Named("end", toDate),
        sql.Named("start", fromDate),
        sql.Named("isDisabledComment", isDisabledComment),
    )
```

**Такой подход позволяет совершать меньше ошибок при выполнении запросов.**


**Задание 3**
С помощью функций time.Now() и time.Since() замерьте время выполнения запросов и запишите ответ в миллисекундах в поле ниже.
Ваш ответ в миллисекундах
ответ:

```go
startTimer := time.Now()
err = insertVideos(ctx, db, videos)
if err != nil {
    log.Fatal(err)
}
duration := time.Since(startTimer)
fmt.Printf("Время выполнения %d\n", duration.Milliseconds())
```

# Подготовленные SQL-инструкции, транзакции и роллбэки

На мощном компьютере добавление 40 000 записей занимает более 9 секунд. Сколько же займёт добавление 100 000 записей? Для небольших проектов, где максимум 50 **INSERT** и **SELECT** в час, это нормально. Но для проектов с большой нагрузкой такой способ не подойдёт.
Улучшим алгоритм. Используем **транзакции**, **стейтменты** и **роллбэки**.

**Транзакция** — это набор последовательных запросов к БД, который представляет собой одну атомарную операцию с данными. Транзакция может быть выполнена либо полностью, либо не выполнена вообще — тогда она не внесёт изменений в базу данных.

**Begin**, **Commit** и **Rollback** — основные команды управления транзакциями:

**Begin** указывает на начало транзакции.
**Commit** сохраняет изменения, которые в неё внесли.
**Rollback** откатывает изменения. Например, удалили пользователя, но при удалении его адреса произошла системная ошибка. Тогда можно использовать Rollback, чтобы вернуть данные пользователя.

Подготовленные SQL-инструкции (стейтменты) — это предподготовка запроса к базе данных, которая ускоряет добавление новых записей. Как именно? Функции **Begin**, **Commit** и **Rollback** позволяют использовать стейтменты: подготавливают запрос, выполняют его с параметрами и закрывают его.

Подготовленные инструкции позволяют СУБД разобрать синтаксис запроса один раз (например, на старте программы). Это экономит десятки или даже сотни миллисекунд при выполнении некоторых запросов, в зависимости от СУБД.

пример:

```go
func insertVideos(ctx context.Context, db *sql.DB, videos []Video) error {
    // шаг 1 — объявляем транзакцию
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    // шаг 1.1 — если возникает ошибка, откатываем изменения
    defer tx.Rollback()

    // шаг 2 — готовим инструкцию
    stmt, err := tx.PrepareContext(ctx, "INSERT INTO videos(title, description, views, likes) VALUES(?,?,?,?)")
    if err != nil {
        return err
    }
    // шаг 2.1 — не забываем закрыть инструкцию, когда она больше не нужна
    defer stmt.Close()

    for _, v := range videos {
        // шаг 3 — указываем, что каждое видео будет добавлено в транзакцию
        if _, err = stmt.ExecContext(ctx, v.Title, v.Description, v.Views, v.Likes); err != nil {
            return err
        }
    }
    // шаг 4 — сохраняем изменения
    return tx.Commit()
}

```


**Шаг 1** — объявляем транзакцию функцией Begin(). Также можно использовать функцию BeginTx(ctx context.Context), которая в качестве аргумента принимает контекст, — тогда, если контекст будет отменён, транзакция тоже будет автоматически отменена. В случае ошибки на любом из шагов откатываем изменения (шаг 1.1).

**Шаг 2** — подготавливаем SQL-заявление функцией PrepareContext(ctx context.Context). Важно: аргументы внутри запроса VALUES(?,?,?,?) будут разными (как и для запросов SELECT, примеры которых рассмотрели в первом уроке). В шаге 2.1 не забудьте закрыть стейтмент после выполнения функции. Стейтмент создаётся только для транзакции и доступнен только в этой функции. Кроме этого, можно использовать стейтмент на уровне базы данных — DB.PrepareContext().

**Шаг 3** — добавляем каждое видео в транзакцию со значениями из видео.

**Шаг 4** — записываем изменения в базу данных.


Возможно, стейтмент применяется настолько широко, что уже был подготовлен на уровне БД.

```go
var db *sql.DB
insertStmt, err := db.PrepareContext(ctx, "INSERT INTO videos(title, description, views, likes) VALUES(?,?,?,?)")
```

В таких случаях вы можете получить его копию, специфицированную для конкретной транзакции и в контексте этой транзакции.

```go
txStmt := tx.StmtContext(ctx, insertStmt)
```

Такой стейтмент работает только в рамках своей транзакции и закроется, когда транзакция будет исполнена или произойдёт откат. Код функции немного изменится.

```go
var insertStmt *sql.Stmt

func insertVideos(ctx context.Context, db *sql.DB, videos []Video) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    txStmt := tx.StmtContext(ctx, insertStmt)
    
    for _, v := range videos {
        if _, err = txStmt.ExecContext(ctx, v.Title, v.Description, v.Views, v.Likes); err != nil {
            return err
        }
    }
    return tx.Commit()
```
Замерим скорость добавления роликов в базу данных через транзакции:
```
~ go run main.go
Время выполнения 69.844334ms
```

Всего около 70 миллисекунд! Использование заранее подготовленных и скомпилированных SQL-инструкций повысило скорость работы почти в сотню раз.

**Задание**
Что сильнее повышает производительность при работе с БД — транзакции sql.Tx или подготовленные директивы sql.Stmt?

+ Транзакции.
(Нет. Транзакции важны не для производительности, а для консистентности.)


**+ Подготовленные стейтменты.**  (OK)

+ Директивы транзакции stmt, err := tx.Prepare(query).


**задание**
Напишите для базы данных метод, который будет обновлять запись ролика с именем Video.Title, если он уже есть в базе.

```go
func (db *sql.DB) Update(v Video) (exists int64, err error)
```

В параметре exists функция будет возвращать, сколько записей с названием Title было найдено.
Поскольку операция простая и короткая, транзакция не нужна. Операция будет применяться часто, поэтому лучше приготовить SQL-директиву на уровне БД в пространстве имён пакета или области видимости main(). 

```go
// готовим SQL-директивы
var upStmt sql.Stmt
var db *sql.DB 

func main() {
    //...
    upStmt, err := db.Prepare("UPDATE videos SET description = ?, views = ?, likes = ? WHERE title = ?")
    if err != nil {
        panic(err)
    }
    //...
}

func (db *sql.DB) Update(v Video) (exists int64, err error){
    // проверим на всякий случай
    if db == nil {
        return errors.New("You haven`t opened the database connection")
    }
    // используем подготовленный стейтмент
    res, err = upStmt.Exec(v.Description, v.Views, v.Likes, v.Title)
    if err != nil {
        return
    }
    // посмотрим, сколько записей было обновлено
    exists, err = res.RowsAffected()
    return
}
```


**Множественная вставка**
Перенос данных из файла в базу требует больших ресурсов памяти. Сейчас в csv-файле 40 000 записей. А что, если их будет миллион? И размер файла будет 10, 200, 500 гигабайт? Тогда нужно будет записывать строки в базу данных пачками.
Идея:
+ читать csv-файл построчно,
+ добавлять строки во временный буфер,
+ при достижении лимита в 1000 строк делать запись в базу данных,
+ повторять, пока не запишем все значения из csv-файла.

>Важно: вы можете возразить, что так лучше не делать. Множественную вставку для INSERT можно реализовать с помощью SQL. Например, INSERT (id, name) INTO table VALUES (1, "video1"), (2, "video2");. А если надо вставлять по 1000 записей? Удобно ли будет перечислять их в одном огромном запросе? Чтобы научиться использовать транзакции, рассмотрим такой вариант. Хотя для этого примера он избыточен.


Создадим новую структуру: 
Ограничиваем буфер 1000 записей.
```go
type VideoDB struct {
    *sql.DB
    buffer []Video
}

func NewVideoDB(fileName string) (*VideoDB, error) {
    db, err := sql.Open("sqlite3", fileName)
    if err != nil {
        return nil, err
    }

    return &VideoDB{
        DB:     db,
        buffer: make([]Video, 0, 1000),
    }, nil
}
```

Чтобы добавить запись в базу данных, напишем метод:
```go
func (db *VideoDB) AddVideo(v *Video) error {
    db.buffer = append(db.buffer, *v)

    if cap(db.buffer) == len(db.buffer) {
        err := db.Flush()
        if err != nil {
            return errors.New("cannot add records to the database")
        }
    }
    return nil
} 

```

Чтобы не сравнивать длину буфера с константой, можно сравнить её с размером буфера при помощи функции **cap**.(https://pkg.go.dev/builtin#cap)
Если буфер заполнен, можно сделать запись и очистить буфер.
Теперь осталось перенести код из функции insertVideos() в функцию Flush и затем очистить буфер. Готовы сделать это самостоятельно?


Теперь осталось перенести код из функции insertVideos() в функцию Flush и затем очистить буфер. Готовы сделать это самостоятельно?


>Задание 6
Напишите функцию func (db *VideoDB) Flush() error.
Последовательность действий:
>+ начать транзакцию (команда BEGIN);
>+ подготовить стейтмент, SQL-директиву для транзакции;
 +добавить все видео из буфера в транзакцию, в случае ошибки выполнить Rollback (функция Exec() и команда ROLLBACK);
>+ сохранить транзакцию (команда COMMIT).
>+ очистить буфер.
>
>**Очистить буфер можно так: db.buffer = db.buffer[:0].**


```go
func (db *VideoDB) Flush() error {
    // проверим на всякий случай
    if db.DB == nil {
        return errors.New("You haven`t opened the database connection")
    }
    tx, err := db.Begin()
    if err != nil {
        return err
    }

    stmt, err := tx.Prepare("INSERT INTO videos(title, description, views, likes) VALUES(?,?,?,?)")
    if err != nil {
        return err
    }
    defer stmt.Close()

    for _, v := range db.buffer {
        if _, err = stmt.Exec(v.Title, v.Description, v.Views, v.Likes); err != nil {
            if err = tx.Rollback(); err != nil {
                log.Fatalf("update drivers: unable to rollback: %v", err)
            }
            return err
        }       
    }

    if err := tx.Commit(); err != nil {
        log.Fatalf("update drivers: unable to commit: %v", err)
    return err
    }

    db.buffer = db.buffer[:0]
    return nil
}
```

**Функция чтения/записи**
После того как выполните задание, останется переписать функцию чтения и записи данных. Например, на следующую:
```go

func VideoFromCsvToDB(r *csv.Reader, db *VideoDB) error {
    for {
        l, err := r.Read()
        if errors.Is(err, io.EOF) {
            break
        } else if err != nil {
            log.Panic(err)
        }

        v := Video{
            Id:           l[0],
            TrendingDate: l[1],
            Title:        l[2],
            Channel:      l[3],
            CategoryId:   l[4],
            // и так далее
        }

        err = db.AddVideo(v)
        if err != nil {
            return err
        }

    }
    // в конце записываем оставшиеся данные из буфера
    db.Flush()
    return nil
}

```
**Задание 7**
Перепишите функцию так, чтобы запись добавлялась в базу только в том случае, если там ещё нет видео с таким названием (Video.Title).

```go
func VideoFromCsvToDB(r *csv.Reader, db *VideoDB) error {
    // проверим на всякий случай
    if db.DB == nil {
        return errors.New("You haven`t opened the database connection")
    }
    // готовим контейнер для проверок
    check := new(string)
    // готовим стейтмент
    stmt, err := Prepare("SELECT title FROM video WHERE title = ?")
    if err != nil {
        return err
    }

    for {
        l, err := r.Read()
        if errors.Is(err, io.EOF) {
            break
        } else if err != nil {
            log.Panic(err)
        }

        // проверяем, есть ли видео в базе
        row := stmt.QueryRow(l[2])
        // хотим убедиться, нашлось или нет
        if err:= row.Scan(check); err != sql.ErrNoRows {
            continue
        }

        v := Video{
            Id:           l[0],
            TrendingDate: l[1],
            Title:        l[2],
            Channel:      l[3],
            CategoryId:   l[4],
            // и так далее
        }

        err = db.AddVideo(v)
        if err != nil {
            return err
        }

    }
    // в конце записываем оставшиеся данные из буфера
    db.Flush()
    return nil
}
```

**Лимитирование**
Пакет database/sql поддерживает пул соединений. Чтобы в реальных проектах резко не начался отказ или деградирование работы базы данных, лучше ограничить продолжительность активного соединения и количество одновременных подключений. Можно это сделать с помощью функций:
```go
db.SetMaxOpenConns(20)
db.SetMaxIdleConns(20)
db.SetConnMaxIdleTime(time.Second * 30)
db.SetConnMaxLifetime(time.Minute * 2)
```

**db.SetMaxOpenConns(**) служит для ограничения количества соединений, используемых приложением. Рекомендуемого числа нет: оно зависит от задач и особенностей работы. Можете поиграться с настройками.

**db.SetMaxIdleConns()** служит для ограничения количества простаивающих соединений. Устанавливайте его с тем же значением, что и db.SetMaxOpenConns(). Если значение меньше, чем SetMaxOpenConns(), соединения могут открываться и закрываться гораздо чаще.

**db.SetConnMaxIdleTime()** задаёт максимальное время простаивания одного соединения.

**db.SetConnMaxLifetime()** задаёт максимальное время работы одного соединения. Функция нужна, чтобы безопасно закрыть соединения в приложении до того, как соединение будет закрыто сервером SQL или промежуточным инфраструктурным приложением. Поскольку некоторые промежуточные инфраструктурные приложения закрывают простаивающие соединения до 5 минут, рекомендуется устанавливать значение меньше.


**SQLx**
**(https://github.com/jmoiron/sqlx)**

Преимущество пакета **database/sql** — простота. Но в рабочих проектах, в больших таблицах использование функции **Scan** может быть утомительным. Нужно следить за порядком столбцов, обязательно делать итерацию по результатам.

Пакет **SQLx** помогает облегчить жизнь разработчика на Go. Расскажем об основных особенностях пакета.
Вместо того чтобы записывать значения из Scan по одному, можно воспользоваться методом **StructScan**.

```go
type Car struct {
    Model       string
    Brand       string
    IsAvailable bool   `db:"is_available"`
    Weight      string `db:"car_weight"`
}

var c Car
err = rows.StructScan(&c)
```

Метод **Select** позволяет записывать все полученные записи в слайс всего двумя строчками кода:

```go
var cars []Car
err := db.Select(&cars, "SELECT * FROM cars WHERE car_weight > ?", 50)
```

Если нужно получить только одну запись, можно воспользоваться методом **Get**:

```go
var car Car
err := db.Get(&car, "SELECT * FROM cars WHERE model = ? and brand = ? LIMIT 1", "polo", "vw")
```

Метод NamedQuery позволяет использовать структуры в запросах:

```go
car := &Car{
    Brand: "VW",
    Model: "Polo",
}

rows, err := db.NamedQuery("SELECT * FROM cars WHERE brand=:brand", car)
```

Также можно записывать данные пачками:
```go
// массив машин
cars := []Cars{
    {},
}
_, err = db.NamedExec(`INSERT INTO cars (brand, model, is_available)
        VALUES (:brand, :model, :is_available)`, cars)
```

**Почему в Go не принято использовать ORM**
ORM (Object-Relational Mapping) — это технология, которая связывает таблицы в базах данных и объекты (классы, структуры) в языке программирования.

Код с ORM выглядит красиво, и кажется, что следует использовать его и в Go. Но это не принято и вот почему:

+ Основное преимущество ORM — абстракция от разных типов базы данных. На локальной машине вы можете использовать SQLlite3, а в рабочем проекте — MySQL. Но пакет database/sql уже предоставляет эту возможность.
+ Каждый раз при использовании ORM нужно изучать его синтаксис. Если у вас есть специфичный запрос, гораздо проще найти решение на чистом SQL, чем для ORM. Использование «костылей» при сложных запросах (или запросах на чистом SQL) сильно замедляет работу.
+ Разные компании используют и будут использовать разные ORM — переключение из одного проекта в другой будет требовать от вас очередного погружения в детали работы ORM. Выгоднее использовать чистый SQL.

Получается, что ORM в Go не решает ни одной проблемы — всё уже есть в пакете database/sql.

Когда можно использовать ORM? Когда в проекте множество однотипных и простых CRUD-операций (Create, Read, Update и Delete). Это сэкономит много времени на разработку. В собственных небольших проектах, где важно быстро разработать решение «на коленке», ORM могут помочь сконцентрироваться на идее, а не на коде.

Один из самых популярных ORM-пакетов в Go — GORM. Миграции, CRUD-операции, хуки, работа со связями — всё это есть в GORM.