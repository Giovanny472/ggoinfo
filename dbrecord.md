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