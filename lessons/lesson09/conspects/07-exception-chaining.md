# Цепочки исключений: raise from

> **Главная мысль.** Если исключение возбуждается во время обработки другого
> исключения, Python **связывает** их в **цепочку (exception chaining)**, чтобы
> первопричина не потерялась. Связь бывает двух видов. **Неявная** возникает
> автоматически, а в трассировке выводится фраза *During handling of the above
> exception, another exception occurred*. **Явная** задается конструкцией
> `raise NewError from original_error`, а в трассировке выводится фраза *The
> above exception was the direct cause of the following exception*. С помощью
> `raise ... from None` вывод цепочки можно **подавить**, если исходное
> исключение является деталью реализации и пользователю не нужно.

---

## 1. Зачем нужны цепочки

Встроенные функции и библиотеки возбуждают исключения, отражающие их внутреннее
устройство (`ValueError` при разборе строки, `KeyError` при поиске ключа).
Такое исключение не всегда подходит для передачи дальше: вызывающий код может
рассчитывать на исключения определенного типа, а пользователю может быть
понятнее сообщение об ошибке, сформулированное в терминах решаемой задачи.
Кроме того, текст встроенных исключений часто малоинформативен. Например, для
`KeyError` в качестве сообщения выводится только сам отсутствующий ключ
(`KeyError: 'email'`), а контекст ошибки (в каком словаре и при выполнении какой
задачи ключ не найден) в сообщении отсутствует.

Для решения перечисленных проблем в Python предусмотрен механизм, позволяющий
возбуждать **другое** исключение непосредственно в блоке `except` и при этом
сохранять контекст исходного исключения. Сохранение первопричины упрощает
отладку. Исключения, связанные таким образом, образуют **цепочку исключений
(exception chaining)**.

---

## 2. Неявная цепочка

Если новое исключение возбуждено **внутри блока `except`** (или `finally`), Python
автоматически связывает его с обрабатываемым исключением:

```python
def load_age(raw_age: str) -> int:
    try:
        return int(raw_age)

    except ValueError:
        raise RuntimeError("failed to load age")


load_age("abc")
```

Трассировка:

```
Traceback (most recent call last):
  File "example.py", line 3, in load_age
    return int(raw_age)
ValueError: invalid literal for int() with base 10: 'abc'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "example.py", line 9, in <module>
    load_age("abc")
  File "example.py", line 6, in load_age
    raise RuntimeError("failed to load age")
RuntimeError: failed to load age
```

Python выводит сначала исходное исключение, затем новое. Фраза *During
handling of the above exception, another exception occurred* означает, что
второе исключение возникло **в процессе обработки** первого. Это может быть как
намеренное действие, так и **ошибка в самом обработчике**; интерпретатор не
различает эти случаи.

---

## 3. Явная цепочка: raise from

Если замена исключения выполняется намеренно, связь следует указать явно с
помощью конструкции `raise ... from ...`:

```python
def load_age(raw_age: str) -> int:
    try:
        return int(raw_age)

    except ValueError as error:
        raise RuntimeError("failed to load age") from error


load_age("abc")
```

Трассировка:

```
Traceback (most recent call last):
  File "example.py", line 3, in load_age
    return int(raw_age)
ValueError: invalid literal for int() with base 10: 'abc'

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "example.py", line 9, in <module>
    load_age("abc")
  File "example.py", line 6, in load_age
    raise RuntimeError("failed to load age") from error
RuntimeError: failed to load age
```

Теперь вместо сообщения *During handling of the above exception, another
exception occurred* выводится сообщение *The above exception was the direct cause
of the following exception*: исходное исключение названо **прямой причиной**
нового. Это сообщает читателю трассировки, что замена выполнена сознательно.

Сравнение двух видов связи:

| | Неявная (`raise New`) | Явная (`raise New from error`) |
|---|---|---|
| Фраза в трассировке | *During handling of the above exception, another exception occurred* | *The above exception was the direct cause of the following exception* |
| Смысл | второе исключение возникло в процессе обработки первого | первое исключение — прямая причина второго |
| Когда использовать | происходит автоматически | при намеренной замене исключения |

В `from` можно указывать **любое** исключение, не только то, которое
перехвачено в `except`:

```python
raise RuntimeError("failed") from ValueError("original problem")
```

---

## 4. raise from None: подавление цепочки

Иногда исходное исключение — внутренняя деталь, которая не нужна
пользователю и только загромождает трассировку. Например, функция ищет
значение в словаре и сообщает собственную ошибку:

```python
def get_price(prices: dict[str, int], product: str) -> int:
    try:
        return prices[product]

    except KeyError:
        raise KeyError(f"unknown product: {product}")


get_price({"apple": 10}, "pear")
```

Без подавления в трассировке будут две секции: `KeyError: 'pear'` и
`KeyError: 'unknown product: pear'`. Первая избыточна. Конструкция
`from None` отключает вывод связанного исключения:

```python
def get_price(prices: dict[str, int], product: str) -> int:
    try:
        return prices[product]

    except KeyError:
        raise KeyError(f"unknown product: {product}") from None
```

Трассировка:

```
Traceback (most recent call last):
  File "example.py", line 11, in <module>
    get_price({"apple": 10}, "pear")
  File "example.py", line 6, in get_price
    raise KeyError(f"unknown product: {product}") from None
KeyError: 'unknown product: pear'
```

### Когда применять from None

- исходное исключение — деталь реализации (внутренняя структура данных,
  промежуточное преобразование), которая не нужна вызывающему коду;
- новое исключение полностью и точно описывает проблему, а исходное только
  дублирует информацию.

### Когда не применять

Если исходное исключение помогает найти причину (например, ошибка
соединения с базой данных, превратившаяся в обобщенное исключение
приложения), его следует сохранить через `raise ... from error`. Подавление
цепочки скрывает информацию, поэтому применяется осознанно, а не по
умолчанию.

---

## 5. Выбор формы возбуждения

| Задача | Запись |
|---|---|
| Передать то же исключение дальше без изменений | `raise` |
| Заменить исключение другим, сохранив причину | `raise NewError(...) from error` |
| Заменить исключение другим, скрыв причину | `raise NewError(...) from None` |
| Возбудить исключение в обработчике без явного указания связи | `raise NewError(...)` — связь неявная; обычно лучше указать `from` |

Если исключение заменяется намеренно, следует **явно** выбрать между
`from error` и `from None`. Неявная форма оправдана только тогда, когда новое
исключение возникает как случайный результат ошибки в обработчике.

---

## Вывод

- Исключение, возбужденное при обработке другого, связывается с ним в
  **цепочку**, чтобы не терять первопричину.
- **Неявная связь** создается автоматически; в трассировке выводится *During
  handling of the above exception, another exception occurred*.
- **Явная связь** задается с помощью конструкции `raise New from error`; в трассировке выводится
  сообщение *The above exception was the direct cause of the following exception*.
- `raise New from None` скрывает цепочку в трассировке. Применять его следует,
  только если исходное исключение — деталь реализации.
- При намеренной замене исключения следует явно указывать `from error` или
  `from None`.

---

## Источники

- [Python documentation. The Python Language Reference — The raise statement](https://docs.python.org/3/reference/simple_stmts.html#raise);
- [Python documentation. The Python Tutorial — Exception Chaining](https://docs.python.org/3/tutorial/errors.html#exception-chaining);
- [PEP 3134 — Exception Chaining and Embedded Tracebacks](https://peps.python.org/pep-3134/).
