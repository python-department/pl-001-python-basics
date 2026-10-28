# Замыкания

> **Главная мысль.** **Замыкание (closure)** — это функция вместе со ссылками на
> переменные объемлющих функций, которые она использует. Локальное
> пространство имен внешней функции исчезает после ее завершения, но если
> вложенная функция обращается к его переменным, эти переменные продолжают
> существовать: они сохраняются в специальных объектах-**ячейках (cell)**,
> на которые ссылается вложенная функция. Благодаря этому функция сохраняет
> значения, с которыми была создана, и может хранить состояние между вызовами
> без глобальных переменных.

---

## 1. Проблема времени жизни локальных переменных

В конспекте о пространствах имен мы выяснили, что локальные переменные
существуют только во время вызова функции. Рассмотрим пример, в котором это
правило, на первый взгляд, нарушается:

```python
from typing import Callable


def make_multiplier(factor: int) -> Callable[[int], int]:
    def multiply(number: int) -> int:
        return number * factor

    return multiply


double = make_multiplier(2)
triple = make_multiplier(3)

print(double(10))  # 20
print(triple(10))  # 30
```

К моменту вызова `double(10)` функция `make_multiplier` уже завершилась, а ее
локальное пространство имен с параметром `factor` должно было быть удалено.
Тем не менее `multiply` успешно использует `factor` и для каждой из двух
функций значение свое: `2` и `3`.

Причина в том, что `factor` для `multiply` — **свободная переменная**: она
не локальная (в `multiply` ей ничего не присваивается), а определена в
объемлющей функции. Такая переменная хранится в **ячейке (cell)**, а объект
вложенной функции содержит ссылку на эту ячейку (атрибут `__closure__`).
Пока на ячейку ссылается хотя бы одна функция, переменная не удаляется, даже
если пространство имен внешней функции уже уничтожено. Устройство ячеек
рассмотрено в следующем разделе.

---

## 2. Устройство замыкания

Компилятор при разборе `make_multiplier` обнаруживает, что имя `factor`
используется во вложенной функции. Поэтому в `make_multiplier` оно хранится не
как обычная локальная переменная, а в **ячейке (cell)** — небольшом объекте,
содержащем ссылку на значение. Каждый вызов `make_multiplier` создает новую
ячейку, а созданная функция `multiply` получает ссылки на нужные ячейки.

Эту структуру можно изучить у объекта функции:

```python
print(double.__code__.co_freevars)  # ('factor',) -- имена свободных переменных
print(double.__closure__)           # (<cell at 0x...: int object at 0x...>,)

print(double.__closure__[0].cell_contents)  # 2
print(triple.__closure__[0].cell_contents)  # 3
```

- `__code__.co_freevars` — имена свободных переменных функции;
- `__closure__` — кортеж ячеек, по одной на каждую свободную переменную;
- `cell_contents` — текущее значение, лежащее в ячейке.

Функции `double` и `triple` созданы одним и тем же кодом, но имеют **разные
ячейки**, поэтому ведут себя по-разному. Замыкание образуют код функции
и принадлежащие ей ячейки.

Если вложенная функция не использует переменных объемлющих функций, замыкания
не образуется, а `__closure__` равен `None`. То же верно для глобальных имен:
они ищутся в пространстве модуля при каждом вызове и в ячейки не помещаются.

```python
rate = 2


def get_rate() -> int:
    return rate


print(get_rate.__closure__)  # None
```

---

## 3. Замыкание хранит переменную, а не значение

Ячейка содержит ссылку на **переменную** объемлющей функции, а не копию ее
значения на момент создания вложенной функции. Если переменная позднее будет
перепривязана, вложенная функция увидит новое значение. Это называется
**поздним связыванием (late binding)**:

```python
from typing import Callable


def make_printer() -> Callable[[], None]:
    message = "first"

    def print_message() -> None:
        print(message)

    message = "second"  # переменная изменена после создания print_message
    return print_message


printer = make_printer()
printer()  # second
```

### Типичная ошибка: функции, создаваемые в цикле

Позднее связывание приводит к распространенной ошибке. Переменная цикла `number` —
одна на весь цикл, поэтому все функции, созданные в нем, используют **одну и
ту же ячейку**:

```python
from typing import Callable


def make_functions() -> list[Callable[[], int]]:
    functions = []

    for number in range(3):
        def get_number() -> int:
            return number

        functions.append(get_number)

    return functions


for function in make_functions():
    print(function())  # 2, 2, 2 -- а не 0, 1, 2
```

К моменту вызова цикл закончился, и в ячейке `number` лежит последнее значение
`2`. Есть два способа получить ожидаемый результат.

**Способ 1.** Вынести создание функции во вспомогательную функцию-фабрику.
Каждый ее вызов создает собственное пространство имен и собственную ячейку:

```python
from typing import Callable


def make_getter(value: int) -> Callable[[], int]:
    def get_value() -> int:
        return value

    return get_value


def make_functions() -> list[Callable[[], int]]:
    functions = []

    for number in range(3):
        functions.append(make_getter(number))

    return functions


for function in make_functions():
    print(function())  # 0, 1, 2
```

**Способ 2.** Зафиксировать значение через параметр со значением по
умолчанию. Значения по умолчанию вычисляются один раз в момент выполнения
`def`, то есть на каждой итерации цикла свое:

```python
from typing import Callable


def make_functions() -> list[Callable[[], int]]:
    functions = []

    for number in range(3):
        def get_number(fixed_number: int = number) -> int:
            return fixed_number

        functions.append(get_number)

    return functions


for function in make_functions():
    print(function())  # 0, 1, 2
```

В этом варианте замыкание отсутствует, а значение хранится в
`__defaults__`. Недостаток — у функции появляется лишний параметр, который
можно случайно переопределить при вызове. Поэтому предпочтителен способ 1.

---

## 4. Состояние в замыкании

Если вложенная функция меняет свободную переменную, то для этого используется
`nonlocal`. Значение сохраняется в ячейке между вызовами, поэтому замыкание
способно хранить **состояние**:

```python
from typing import Callable


def make_counter() -> Callable[[], int]:
    count = 0

    def increment() -> int:
        nonlocal count
        count += 1
        return count

    return increment


first_counter = make_counter()
second_counter = make_counter()

print(first_counter())   # 1
print(first_counter())   # 2
print(first_counter())   # 3
print(second_counter())  # 1 -- у каждого счетчика свое состояние
```

Сравним с вариантом на глобальной переменной (`global counter`):

| | Глобальная переменная | Замыкание |
|---|---|---|
| Число независимых состояний | одно на весь модуль | по одному на каждый вызов фабрики |
| Доступ к состоянию из других частей программы | любой код может изменить | только через возвращенную функцию |
| Риск конфликта имен | есть | нет |

Состояние замыкания **инкапсулировано**: к переменной `count` нет прямого
доступа извне, ее можно изменить только вызовом `increment`.

Состояние можно хранить и в изменяемом объекте, тогда `nonlocal` не нужен,
поскольку имя не присваивается:

```python
from typing import Callable


def make_history() -> Callable[[str], list[str]]:
    records = []

    def add(record: str) -> list[str]:
        records.append(record)
        return records

    return add


add_record = make_history()
add_record("first")
print(add_record("second"))  # ['first', 'second']
```

---

## Вывод

- **Замыкание** — функция вместе со ссылками на переменные объемлющих
  функций (свободные переменные), которые она использует. Эти переменные
  живут дольше, чем вызов внешней функции.
- Свободные переменные хранятся в ячейках (`cell`). Просмотреть их можно через
  `__code__.co_freevars`, `__closure__` и `cell_contents`. Каждый вызов
  внешней функции создает новые ячейки.
- Замыкание хранит ссылку на **переменную**, а не копию значения (позднее
  связывание). Поэтому функции, создаваемые в цикле, используют одну общую
  ячейку; исправить это можно созданием через вспомогательную фабрику или
  значением по умолчанию.
- Для изменения свободной переменной во вложенной функции нужен `nonlocal`.
  Так замыкание хранит состояние между вызовами, при этом каждое замыкание
  имеет собственное состояние, недоступное извне.
- Обращение к глобальным именам замыкания не образует (`__closure__` равен
  `None`).

---

## Источники

- [Python documentation. The Python Language Reference — Resolution of names](https://docs.python.org/3/reference/executionmodel.html#resolution-of-names);
- [Python documentation. Programming FAQ — Why do lambdas defined in a loop with different values all return the same result?](https://docs.python.org/3/faq/programming.html#why-do-lambdas-defined-in-a-loop-with-different-values-all-return-the-same-result).
