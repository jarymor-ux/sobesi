Пара нюансов, которых не было с каналом отмены:

**Контекст — это матрешка**. Объект контекста неизменяемый. Чтобы добавить контексту новые свойства, создают новый контекст («дочерний») на основе старого («родительского»). Поэтому мы сначала создали пустой контекст, а затем новый (с возможностью отмены) на его основе:

```go
// родительский контекст
ctx := context.Background()

// дочерний
ctx, cancel := context.WithCancel(ctx)
```

Если отменить родительский контекст — отменятся и все дочерние (но не наоборот).

```go
// родительский контекст
parentCtx, parentCancel := context.WithCancel(context.Background())

// дочерний контекст
childCtx, childCancel := context.WithCancel(parentCtx)

// parentCancel() отменит parentCtx и childCtx
// childCancel() отменит только childCtx
```

**Многократная отмена безопасна**. Если два раза вызвать `close()` на канале, получим панику. А вот вызывать `cancel()` контекста можно сколько угодно. Первая отмена сработает, а остальные будут проигнорированы. Это удобно, потому что можно сразу после создания контекста запланировать отложенный `cancel()`, плюс явно отменить контекст при необходимости (как мы сделали в функции `maybeCancel`). С каналом бы так не получилось.

[песочница](https://go.dev/play/p/GYJUuhucVkn)


### **Таймаут**

Настоящая сила контекста в том, что его можно использовать как для ручной отмены, так и для отмены по таймауту. Следите за руками:

```go
// выполняет функцию fn с учетом контекста ctx
func execute(ctx context.Context, fn func() int) (int, error) {
    // код не меняется
}

func main() {
    // работает в течение 100 мс
    work := func() int {
        // ...
    }

    // возвращает случайный аргумент из переданных
    randomChoice := func(arg ...int) int {
        i := rand.Intn(len(arg))
        return arg[i]
    }

    // случайный таймаут - 50 мс либо 150 мс
    timeout := time.Duration(randomChoice(50, 150)) * time.Millisecond
    ctx, cancel := context.WithTimeout(context.Background(), timeout)    // (1)
    defer cancel()

    res, err := execute(ctx, work)
    fmt.Println(res, err)
}

```

Функция `execute()` вообще не изменилась, а в `main()` вместо `context.WithCancel()` теперь `context.WithTimeout()` ➊. Этого достаточно, чтобы `execute()` теперь отваливалась по таймауту в половине случаев (ошибка `context.DeadlineExceeded`):

```
work done
42 <nil>

0 context deadline exceeded

0 context deadline exceeded

work done
42 <nil>
```

Благодаря контексту, функции `execute()` больше не нужно знать, чем вызвана отмена — ручным действием или таймаутом. Все, что от нее требуется — слушать сигнал отмены на канале `ctx.Done()`.

Удобно!

### **Родительский и дочерний таймауты**

Допустим, у нас есть все та же функция `execute()` и две функции, которые она может выполнить — быстрая `work()` и медленная `slow()`:

```go
// выполняет функцию fn с учетом контекста ctx
func execute(ctx context.Context, fn func() int) (int, error) {
    ch := make(chan int, 1)

    go func() {
        ch <- fn()
    }()

    select {
    case res := <-ch:
        return res, nil
    case <-ctx.Done():
        return 0, ctx.Err()
    }
}
```

```go
// работает в течение 100 мс
work := func() int {
    time.Sleep(100 * time.Millisecond)
    return 42
}

// работает в течение 300 мс
slow := func() int {
    time.Sleep(300 * time.Millisecond)
    return 13
}
```

Пусть таймаут по умолчанию составляет 200 мс:

```go
// возвращает контекст
// с умолчательным таймаутом 200 мс
getDefaultCtx := func() (context.Context, context.CancelFunc) {
    const timeout = 200 * time.Millisecond
    return context.WithTimeout(context.Background(), timeout)
}
```

Тогда `work()` с умолчательным контекстом успеет выполниться:

```go
// таймаут 200 мс
ctx, cancel := getDefaultCtx()
defer cancel()

// успеет выполниться
res, err := execute(ctx, work)
fmt.Println(res, err)
// 42 <nil>
```

А `slow()` — не успеет:

```go
// таймаут 200 мс
ctx, cancel := getDefaultCtx()
defer cancel()

// НЕ успеет выполниться
res, err := execute(ctx, slow)
fmt.Println(res, err)
// 0 context deadline exceeded
```

Мы можем создать дочерний контекст, чтобы задать более жесткий таймаут. Тогда применится именно он, а не родительский:

```go
// родительский контекст с таймаутом 200 мс
parentCtx, cancel := getDefaultCtx()
defer cancel()

// дочерний контекст с таймаутом 50 мс
childCtx, cancel := context.WithTimeout(parentCtx, 50*time.Millisecond)
defer cancel()

// теперь work НЕ успеет выполниться
res, err := execute(childCtx, work)
fmt.Println(res, err)
// 0 context deadline exceeded
```

А если создать дочерний контекст с более мягким ограничением — он окажется бесполезен. Таймаут родительского контекста сработает раньше:

```go
// родительский контекст с таймаутом 200 мс
parentCtx, cancel := getDefaultCtx()
defer cancel()

// дочерний контекст с таймаутом 500 мс
childCtx, cancel := context.WithTimeout(parentCtx, 500*time.Millisecond)
defer cancel()

// slow все равно НЕ успеет выполниться
res, err := execute(childCtx, slow)
fmt.Println(res, err)
// 0 context deadline exceeded
```

Получается вот что:

- Из таймаутов, наложенных родительским и дочерним контекстами, всегда срабатывает более жесткий.
- Дочерние контексты могут только ужесточить таймаут родительского, но не ослабить его.

[песочница](https://go.dev/play/p/jLnAKUfj3vq)

### **Контекст со значениями**

Основное назначение контекста в Go — отмена операций. Но есть и еще одно — передача дополнительной информации о вызове. За это отвечает `context.WithValue()`, которая создает контекст со значением по ключу:

```go
type contextKey string

var requestIdKey = contextKey("id")
var userKey = contextKey("user")

func main() {
    work := func() int {
        return 42
    }

    // контекст с идентификатором запроса
    ctx := context.WithValue(context.Background(), requestIdKey, 1234)
    // и пользователем
    ctx = context.WithValue(ctx, userKey, "admin")
    res := execute(ctx, work)
    fmt.Println(res)

    // пустой контекст
    ctx = context.Background()
    res = execute(ctx, work)
    fmt.Println(res)
}

```

В качестве ключей принято использовать не строки или числа, а отдельные типы (`contextKey` в нашем примере). Так не возникнет конфликтов, если один и тот же контекст модифицируется в двух пакетах, и оба решат добавить значение с ключом `"user"`.

Чтобы достать значение по ключу, используют метод контекста `Value()`:

```go
// выполняет функцию fn с учетом контекста ctx
func execute(ctx context.Context, fn func() int) int {
    reqId := ctx.Value(requestIdKey)
    if reqId != nil {
        fmt.Printf("Request ID = %d\\n", reqId)
    } else {
        fmt.Println("Request ID unknown")
    }

    user := ctx.Value(userKey)
    if user != nil {
        fmt.Printf("Request user = %s\\n", user)
    } else {
        fmt.Println("Request user unknown")
    }
    return fn()
}
```

```
Request ID = 1234
Request user = admin
42
Request ID unknown
Request user unknown
42
```

И `context.WithValue()`, и `ctx.Value()` оперируют значениями типа `any`:

```go
func WithValue(parent Context, key, val any) Context

type Context interface {
    // ...
    Value(key any) any
}
```

Эту уродливую нетипизированную парочку применяют в обработке HTTP-запросов от безысходности: там нет нормального способа передать метаданные запроса, кроме как сложить их в контекст. Но нет ни единой причины использовать `WithValue()` в обычном коде. Даже задачку предлагать не буду (ಠ_ಠ)

[песочница](https://go.dev/play/p/mYkb3TnLvU5)


