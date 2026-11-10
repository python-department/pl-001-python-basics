# Несколько блоков except и порядок их следования

> **Главная мысль.** Один блок `try` может иметь **несколько блоков `except`**
> для разных исключений. Интерпретатор проверяет их **сверху вниз** и выполняет
> **первый подходящий**: тот, тип которого совпадает с типом исключения
> или включает его. Остальные блоки пропускаются, даже если они тоже
> подошли бы. Поэтому порядок блоков важен: **более специфичные исключения
> необходимо указывать раньше более общих**. Если поставить `except Exception`
> перед `except ValueError`, второй блок никогда не выполнится. Несколько
> исключений можно обработать одним блоком, указав их кортежем.

---

## 1. Несколько блоков except

Разные исключения часто требуют разной реакции. Для этого после `try` можно
написать несколько блоков `except`:

```python
def divide_numbers(raw_dividend: str, raw_divisor: str) -> None:
    try:
        dividend = int(raw_dividend)
        divisor = int(raw_divisor)
        result = dividend / divisor
        print(f"Result: {result}")

    except ValueError:
        print("Both values must be integers")

    except ZeroDivisionError:
        print("Divisor must not be zero")


divide_numbers("10", "2")    # Result: 5.0
divide_numbers("10", "abc")  # Both values must be integers
divide_numbers("10", "0")    # Divisor must not be zero
```

При каждом вызове выполняется **не более одного** блока `except`: исключение
обрабатывает первый подходящий по типу блок `except`, остальные блоки `except`
пропускаются, после чего выполнение программы продолжается с команд,
расположенных после блока `try-except`.

---

## 2. Один блок except для нескольких исключений

Если реакция на разные исключения одинакова, дублировать код обработчиков не
нужно. Типы исключений перечисляются **в круглых скобках через запятую**,
то есть задаются кортежем:

```python
def parse_ratio(raw_dividend: str, raw_divisor: str) -> float:
    try:
        return int(raw_dividend) / int(raw_divisor)

    except (ValueError, ZeroDivisionError) as error:
        print(f"Cannot calculate ratio: {error}")
        return 0.0


print(parse_ratio("10", "4"))    # 2.5
print(parse_ratio("10", "abc"))  # Cannot calculate ratio: invalid literal ...
print(parse_ratio("10", "0"))    # Cannot calculate ratio: division by zero
```

Блок выполняется, если исключение имеет **любой** из перечисленных типов (или
входит в один из них). Переменная, объявленная в заголовке `except` после `as`,
позволяет использовать в обработчике конкретный перехваченный объект
исключения.

Скобки необходимо ставить всегда. В версиях Python до 3.14 запись без скобок
является синтаксической ошибкой:

```python
try:
    result = int("abc")

except ValueError, ZeroDivisionError:  # SyntaxError в Python < 3.14
    pass
```

> **Примечание.** Начиная с Python 3.14 запись `except A, B:` без скобок
> допустима, если не используется `as`; вместе с `as` скобки по-прежнему
> обязательны. Для совместимости с более ранними версиями и для читаемости в
> курсе всегда используется кортеж в скобках.

Если одинаковой должна быть реакция на семейство родственных исключений,
иногда проще указать общий тип, включающий их (`LookupError` вместо
`(IndexError, KeyError)`), но только если не будут перехвачены лишние
исключения, не относящиеся к задаче.

Блоки можно комбинировать: часть исключений обрабатывается общим блоком, часть —
отдельными:

```python
try:
    result = int(raw_dividend) / int(raw_divisor)

except (ValueError, TypeError):
    print("Invalid input")

except ZeroDivisionError:
    print("Divisor must not be zero")
```

---

## 3. Порядок поиска нужного блока except

Когда в блоке `try` возникает исключение, интерпретатор **последовательно, сверху
вниз** сравнивает его с типами в каждом `except`. Блок подходит, если тип
исключения совпадает с указанным или входит в него.

```python
try:
    int("abc")  # возбуждается ValueError: строку нельзя преобразовать в int

except KeyError:
    print("KeyError block")       # не подходит

except ValueError:
    print("ValueError block")     # подходит -- выполняется

except Exception:
    # подошел бы, но пропускается: выше есть подходящий блок
    print("Exception block")
```

Вывод:

```
ValueError block
```

Алгоритм:

1. Начать с самого верхнего блока `except`.
2. Проверить, подходит ли текущий блок к возникшему исключению.
3. Если подходит — выполнить его тело и **прекратить поиск**. Остальные блоки
   не проверяются.
4. Если не подходит — перейти к следующему блоку ниже и повторить проверку
   (шаги 2-4).
5. Если блоки закончились, а подходящий не найден, исключение продолжает
   распространяться дальше.

Важно, что выбирается **первый подходящий**, а не наиболее точный. Интерпретатор не
сравнивает блоки между собой по специфичности: он только идет по порядку
сверху вниз.

---

## 4. Ошибки порядка: общий блок перед частным

Поскольку выбирается первый подходящий блок, общий обработчик, расположенный
выше частного, **перехватывает** все исключения, которые должен был получить
частный:

```python
try:
    int("abc")

except Exception:
    print("Generic error")

except ValueError:
    print("Invalid value")  # недостижимый код
```

Вывод:

```
Generic error
```

`ValueError` входит в `Exception`, поэтому первый же блок подходит,
и до `except ValueError` управление **никогда не дойдет**. Блок с
`ValueError` — **недостижимый код (dead code)**.

Особенность этой ошибки в том, что интерпретатор **не сообщает о ней**: синтаксически
программа корректна, исключение не возникает, и предупреждения не выводятся. Такую ошибку можно найти только внимательным чтением кода или
с помощью статических анализаторов (линтеров), например `pylint` или `ruff`.

### Правило

Блоки `except` располагаются **от частных к общим**: сначала более частные
типы, затем более общие типы, включающие их:

```python
try:
    int("abc")

except ValueError:
    print("Invalid value")

except Exception:
    print("Generic error")
```

Теперь `ValueError` обрабатывается собственным блоком, а все остальные ошибки
прикладного кода — общим `except Exception`, расположенным последним.

### Независимые типы

Порядок имеет значение только тогда, когда один из типов входит в другой.
Для независимых типов (`ValueError` и `ZeroDivisionError` не входят друг в
друга) порядок блоков не влияет на результат:

```python
try:
    result = 10 / 0

except ValueError:
    print("Invalid value")

except ZeroDivisionError:
    print("Divisor must not be zero")

# Divisor must not be zero
```

Если поменять эти два блока местами, будет выведено то же самое. Исключение
`ZeroDivisionError` не относится к типу `ValueError`, поэтому блок
`except ValueError` не подходит для него независимо от своего положения, и
выполнится только `except ZeroDivisionError`.

### Примеры вложенных типов

В следующих примерах порядок блоков важен:

```python
try:
    value = scores[key]

except KeyError:                  # конкретный тип
    print("Key is missing")

except LookupError:               # общий тип: сработает для IndexError и др.
    print("Lookup failed")
```

Если поменять блоки местами, `except KeyError` станет недостижимым: любой
`KeyError` будет перехвачен обработчиком `LookupError` раньше.

Аналогично, при работе с файлами:

```python
try:
    file = open("data.txt")

except FileNotFoundError:
    print("File does not exist")

except OSError:
    print("Cannot open file")
```

`FileNotFoundError` входит в `OSError`, поэтому его блок обязан стоять выше.

---

## 5. Неперехваченные исключения и общий блок в конце

Если ни один блок не подошел, исключение продолжает распространяться. Общий
`except Exception` в конце списка обычно добавляют на верхних уровнях
программы, чтобы записать информацию о непредвиденной ошибке:

```python
try:
    result = int(raw_dividend) / int(raw_divisor)

except ZeroDivisionError:
    print("Divisor must not be zero")

except ValueError:
    print("Invalid input")

except Exception as error:
    print(f"Unexpected error: {type(error).__name__}: {error}")
```

Блок `except Exception` перехватывает любое исключение, не обработанное
предыдущими блоками, поэтому сообщение о непредвиденной ошибке будет выведено
в любом случае.

---

## Вывод

- Блок `try` может иметь **несколько `except`** для разных исключений;
  выполняется не более одного из них.
- Для одинаковой реакции на разные исключения используется один блок с
  **кортежем** типов: `except (ValueError, TypeError):`. Конструкция
  `as error` необязательна и нужна, только если в обработчике требуется
  объект исключения.
- Интерпретатор проверяет блоки **сверху вниз** и выполняет **первый подходящий**:
  тот, чей тип совпадает с типом исключения или включает его.
  После этого поиск прекращается.
- Блоки следует располагать **от частных к общим**. Общий блок
  (`except Exception`), стоящий выше частного (`except ValueError`), перехватит
  все исключения первым, и частный блок станет недостижимым.
- Интерпретатор не сообщает об ошибке порядка: ее обнаруживают только
  внимательным чтением кода и линтерами.
- Для несвязанных типов исключений порядок блоков значения не имеет.

---

## Источники

- [Python documentation. The Python Tutorial — Handling Exceptions](https://docs.python.org/3/tutorial/errors.html#handling-exceptions);
- [Python documentation. The Python Language Reference — The try statement](https://docs.python.org/3/reference/compound_stmts.html#the-try-statement).
