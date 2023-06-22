# errors, log

Так как Go позволяет возвращать из функций несколько значений, принято последним значением возвращать ошибку. Если возвращаемое значение ошибки равно nil, функция завершилась корректно. В противном случае нужно обработать ошибку и/или вернуть её выше по стеку. Если функция завершилась с ошибкой, не стоит использовать остальные возвращаемые значения: они могут быть не определены, функция может выполниться не полностью и не успеть вычислить их значения.

Тип error
Ошибкой в языке Go может быть значение любого типа, который совместим с интерфейсным типом error.

```go
type error interface {
    Error() string
} 
```
Для создания ошибки чаще всего применяют:

функцию ( в которой указывают шаблон форматирования и дополнительные параметры.)
```go
fmt.Errorf(format string, a ...interface{})
```
или функцию 
```go
errors.New(text string)
```
**пример:**
```go
if userID <= 0 {
    return errors.New(`invalid userID`)
} else if !User(userID) {
    return fmt.Errorf(`can't find user id (%d)`, userID)
}
```

По умолчанию ошибки имеют тип, который может хранить только строку. Можно создать свой тип с необходимыми полями, определить для него метод Error() string и возвращать ошибки с дополнительной информацией. 

Например, для сохранения времени возникновения ошибки можно определить такой тип:

```go
// TimeError предназначен для ошибок с фиксацией времени возникновения.
type TimeError struct {
    Time time.Time
    Err  error
}

// Error добавляет поддержку интерфейса error для типа TimeError.
func (te *TimeError) Error() string {
    return fmt.Sprintf("%v %v", te.Time.Format(`2006/01/02 15:04:05`), te.Err)
}

// NewTimeError упаковывает ошибку err в тип TimeError c текущим временем.
func NewTimeError(err error) error {
    return &TimeError{
        Time: time.Now(),
        Err:  err,
    }
}
```

>При создании ошибок всегда нужно возвращать указатели на структуру. Тогда, если создать ошибки с одинаковыми данными, они не будут равны друг другу. Например, если создать ошибку errors.New("EOF"), она не будет равна io.EOF, которая определяется точно так же. Если вместо указателя возвращать структуру, то нельзя будет однозначно идентифицировать ошибку.


**Задание 1 из 6**
В программе ниже определён тип **LabelError**, который добавляет к ошибке строковую метку. Опишите для этого типа методы **Error()** и **NewLabelError(string, error)**, чтобы программа выводила **[FILE] open mytest.txt: no such file or directory.**

```go
package main

import (
    "fmt"
    "os"
    "strings"
)

// LabelError описывает ошибку с дополнительной меткой.
type LabelError struct {
    Label string // метка должна быть в верхнем регистре
    Err   error
}

// добавьте методы Error() и NewLabelError(string, error)
// РЕШЕНИЕ:
// Error добавляет поддержку интерфейса error для типа LabelError.
func (le *LabelError) Error() string {
    return fmt.Sprintf("[%s] %v", le.Label, le.Err)
}

// NewLabelError упаковывает ошибку err в тип LabelError.
func NewLabelError(label string, err error) error {
    return &LabelError{
        Label: strings.ToUpper(label),
        Err:   err,
    }
}

func main() {
    _, err := os.ReadFile("mytest.txt")
    if err != nil {
        err = NewLabelError("file", err)
    }
    fmt.Println(err)
    // должна выводить
    // [FILE] open mytest.txt: no such file or directory
}
```
## Упаковка ошибок
Часто при возникновении ошибки она возвращается вверх по стеку с дополнительным комментарием. В момент, когда функция хочет обработать ошибку, уже сложно понять, какой была исходная ошибка.

Чтобы решить эту проблему, Go даёт возможность упаковывать ошибки (error wrapping). Для этого достаточно в функции Errorf добавить к ошибке спецификатор %w. Исходную ошибку можно получить функцией errors.Unwrap.

Допустим, при старте программы нужно прочитать файл конфигурации или создать его, если он отсутствует.

```go
func ReadTextFile(filename string) (string, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return ``, fmt.Errorf(`failed reading file: %w`, err)
    }
    return string(data), nil
}

func main() {
    if data, err := ReadTextFile(`myconfig.yaml`); err != nil {
        if os.IsNotExist(errors.Unwrap(err)) {
            // создаём файл конфигурации
            return
        }
        panic(err)
    }
}
```

В этом примере исходная ошибка распаковывается функцией **Unwrap** и проверяется на нужное значение. Если тип ошибки не имеет метода **Unwrap**, то **errors.Unwrap** вернёт **nil**. Поэтому следует определять метод Unwrap для типов ошибок, которые упаковывают исходные ошибки. 

Вот пример реализации для типа TimeError, который описан выше:
```go
func (te *TimeError) Unwrap() error {
    return te.Err
}
```

В пакете **errors** есть ещё две функции для работы с упакованными ошибками. Используя спецификатор **%w** или упаковывая ошибки в свои типы, можно создать длинную последовательность вложенных ошибок — тогда применять функцию **Unwrap** становится неудобно.

+ Функция **errors.Is(err, target error) bool** сравнивает ошибку со значением. Она возвращает **true**, если текущая ошибка **err** равна **target** или содержит ошибку **target**. Проверку ошибки из предыдущего примера можно переписать так:

```go
  if errors.Is(err, os.ErrNotExist) {
      // создаём файл конфигурации
  }
```

Функция **errors.Is** не только проверяет равенство каждой ошибки в цепочке контрольному значению. Если какой-то тип из упакованных ошибок имеет метод **Is(err error) bool** и он для **target** вернёт **true**, то и **errors.Is** вернёт **true**.

Функция **errors.As(err error, target interface{}) bool** работает как утверждение типа, но анализирует всю последовательность ошибок. Если она находит ошибку в цепочке, которая имеет такой же тип, на какой указывает **target**, то присваивает значению **target** эту ошибку и возвращает true. В противном случае возвращается **false**. Параметр **target** должен быть ненулевым указателем. Подобно функции **errors.Is**, **errors.As** проверяет наличие метода **As(interface{}) bool** и его возвращаемое значение для **target**. 

```go
 var te *TimeError
  if errors.As(err, &te) {
      log.Printf(`ошибка %v возникла %d миллисекунд назад `, err, time.Since(te.Time).Milliseconds())
  }
   
```

Как уже было сказано, ошибки должны быть указателями, а не структурами. Поэтому в примере **te** — это указатель на структуру **TimeError**, а не просто структура.

Стоит ли возвращать упакованные ошибки из функций пакета? Если нужно скрыть какие-то моменты реализации, то ошибки не упаковываются. Если в пакете есть экспортируемая ошибка и в дальнейшем возможны её изменения, то её лучше возвращать в упакованном виде.

```go
var ErrAccessDenied = errors.New(`access denied`)

func LoadSettings(userID int) error {
    if !AdminUser(userID) {
        return fmt.Errorf(`%w`, ErrAccessDenied)
    }
    // загружаем настройки
}
```

Закрепим материал примерами правильной и неправильной работы с ошибками:

```go

// плохо
if err == ErrAccessDenied {
}
// хорошо
if errors.Is(err, ErrAccessDenied) {
}

var myErr *MyError
// плохо
if myErr, ok = err.(*MyError); ok {
}
// хорошо
if errors.As(err, &myErr) {
}

```
## задание
В функцию main из задания 1 были внесены изменения. Допишите код, чтобы она работала правильно.

```go
// программа должна выводить правильное значение

// решение
// Unwrap возвращает исходную ошибку.
func (le *LabelError) Unwrap() error {
    return le.Err
} 

func main() {
    _, err := os.ReadFile("mytest.txt")
    if err != nil {
        err = NewLabelError("file", err)
    }
    fmt.Println(errors.Is(err, os.ErrNotExist), err)
    // должна выводить
    // true [FILE] open mytest.txt: no such file or directory
}
```

# Логирование

Хорошая практика — записывать сведения о состоянии программы. Обязательно фиксируются возникающие ошибки, нештатные ситуации и события.

Go предлагает использовать пакет log для журналирования информации любого типа. По умолчанию логи записываются в поток стандартного вывода ошибок os.Stderr. 

