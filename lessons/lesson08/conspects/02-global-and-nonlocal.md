# Ключевые слова global и nonlocal

> **Главная мысль.** По умолчанию присваивание внутри функции создает локальное
> имя. Чтобы присваивание изменяло имя из **внешней** области видимости, нужно
> явно объявить это в теле функции. Для этого есть две инструкции: **`global`**
> привязывает имя к пространству имен **модуля**, **`nonlocal`** — к
> пространству имен ближайшей **объемлющей функции**. Обе инструкции влияют
> на символьную таблицу функции: объявленное имя перестает быть локальным.

---

## 1. Инструкция global

Инструкция `global имя` сообщает компилятору, что в данной функции указанное
имя относится к глобальному пространству имен. Все присваивания этому имени в
теле функции изменяют **глобальную** переменную:

```python
counter = 0


def increment() -> None:
    global counter
    counter += 1


increment()
increment()
print(counter)  # 2
```

Без `global` строка `counter += 1` приводила бы к `UnboundLocalError`
(см. предыдущий конспект). С `global` имя `counter` не попадает в список
локальных имен функции:

```python
print(increment.__code__.co_varnames)  # () -- локальных имен нет
print(increment.__code__.co_names)     # ('counter',) -- обращение к глобальному имени
```

### Особенности global

Объявление можно использовать для нескольких имен сразу: `global first, second`.

Если глобального имени еще нет, присваивание после `global` **создаст** его в
пространстве модуля:

```python
def create_setting() -> None:
    global debug_mode
    debug_mode = True


create_setting()
print(debug_mode)  # True
```

Инструкция `global` должна стоять **раньше** первого использования имени в
теле функции, иначе возникает `SyntaxError`:

```python
def broken() -> None:
    print(counter)
    global counter  # SyntaxError: name 'counter' is used prior to global declaration
```

Инструкция `global` **не нужна**, если функция только читает глобальную
переменную или изменяет изменяемый объект без присваивания имени:

```python
history = []


def add_to_history(item: str) -> None:
    history.append(item)  # имя history только читается -- global не нужен


add_to_history("first")

print(history)  # ['first']
```

### Почему global следует использовать осторожно

Функция, меняющая глобальные переменные, имеет **побочный эффект**: ее
результат зависит не только от аргументов, но и от текущего состояния модуля,
а вызов изменяет состояние, видимое остальному коду. Такие функции сложнее
тестировать, использовать повторно и отлаживать: причину неверного значения
глобальной переменной приходится искать во всех функциях, которые объявляют ее
через `global`.

Если функции необходимо сохранять состояние между вызовами, лучше использовать
другие механизмы. Один из них — замыкания, которым посвящены следующие
конспекты.

---

## 2. Инструкция nonlocal

Для вложенных функций существует та же проблема: присваивание во вложенной
функции создает локальное имя, а не изменяет имя объемлющей функции.

```python
def outer() -> None:
    message = "outer"

    def inner() -> None:
        message = "inner"  # новая локальная переменная inner

    inner()
    print(message)  # outer -- значение не изменилось


outer()
```

А составное присваивание приводит к `UnboundLocalError`:

```python
def outer() -> None:
    counter = 0

    def inner() -> None:
        counter += 1  # UnboundLocalError

    inner()


outer()
```

Инструкция **`nonlocal имя`** указывает, что имя нужно искать не в локальной
области видимости, а в ближайшей объемлющей **функции**, где оно определено.
Присваивания изменяют именно эту переменную:

```python
def outer() -> None:
    counter = 0

    def inner() -> None:
        nonlocal counter
        counter += 1

    inner()
    inner()
    print(counter)  # 2


outer()
```

Имя, объявленное как `nonlocal`, не входит в локальные переменные вложенной
функции. Вместо этого оно попадает в список **свободных переменных**
(`co_freevars`) — имен, которые функция использует, но которые определены
в объемлющей области видимости:

```python
def outer() -> None:
    counter = 0

    def inner() -> None:
        nonlocal counter
        counter += 1

    print(inner.__code__.co_varnames)   # () -- локальных имен нет
    print(inner.__code__.co_freevars)   # ('counter',) -- свободная переменная


outer()
```

### Правила nonlocal

`nonlocal` ищет имя только в **объемлющих функциях**. Глобальные переменные
через `nonlocal` изменить нельзя, а если имя нигде в объемлющих функциях не
определено, возникает `SyntaxError` — причем на этапе компиляции, до запуска
программы:

```python
counter = 0


def outer() -> None:
    def inner() -> None:
        nonlocal counter  # SyntaxError: no binding for nonlocal 'counter' found
        counter += 1
```

Если имя определено в нескольких объемлющих функциях, выбирается
**ближайшая** к вложенной:

```python
def outer() -> None:
    level = "outer"

    def middle() -> None:
        level = "middle"

        def inner() -> None:
            nonlocal level
            level = "changed"

        inner()
        print(level)  # changed -- изменилась переменная middle

    middle()
    print(level)  # outer -- переменная outer не затронута


outer()
```

На уровне модуля `nonlocal` использовать нельзя, т.к. там нет объемлющей функции.

---

## 3. Сравнение global и nonlocal

| | `global` | `nonlocal` |
|---|---|---|
| Привязывает имя к | пространству имен модуля | ближайшей объемлющей функции |
| Имя должно существовать заранее | нет (присваивание создаст его) | да (иначе `SyntaxError`) |
| Где применяется | в любой функции | только во вложенной функции |
| Попадает в символьную таблицу как | глобальное имя (`co_names`) | свободная переменная (`co_freevars`) |

Обе инструкции нужны только для **присваивания**. Для чтения внешнего имени и
для изменения изменяемого объекта (`append`, `update`, `d[key] = value`) они не
требуются.

```python
def outer() -> None:
    items = []

    def inner() -> None:
        items.append(1)  # nonlocal не нужен: имя items не присваивается

    inner()
    print(items)  # [1]


outer()
```

---

## Вывод

- Присваивание в функции по умолчанию создает локальное имя. Чтобы изменить
  имя из внешней области видимости, его нужно объявить в теле функции.
- **`global имя`** связывает имя с пространством имен модуля. Имя можно
  создать присваиванием; инструкция должна стоять до первого использования
  имени. Изменение глобального состояния через `global` усложняет понимание и
  тестирование кода.
- **`nonlocal имя`** связывает имя с ближайшей объемлющей функцией, в которой оно
  определено. Если такого имени нет, возникает `SyntaxError`. Глобальные имена
  через `nonlocal` недоступны.
- Объявленные `nonlocal` имена хранятся как свободные переменные
  (`__code__.co_freevars`) — это основа механизма замыканий.
- Для чтения внешнего имени и вызова методов изменяемых объектов `global` и
  `nonlocal` не требуются.

---

## Источники

- [Python documentation. The Python Language Reference — The `global` statement](https://docs.python.org/3/reference/simple_stmts.html#the-global-statement);
- [Python documentation. The Python Language Reference — The `nonlocal` statement](https://docs.python.org/3/reference/simple_stmts.html#the-nonlocal-statement);
- [Python documentation. The Python Tutorial — Scopes and Namespaces Example](https://docs.python.org/3/tutorial/classes.html#scopes-and-namespaces-example).
