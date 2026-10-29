# Подмена исходного объекта и functools.wraps

> **Главная мысль.** Запись `@decorator` над `def compute_sum` равносильна
> присваиванию `compute_sum = decorator(compute_sum)`. После него имя
> `compute_sum` ссылается на функцию `wrapper`, которую вернул декоратор, а не
> на исходную функцию. Поэтому атрибуты, прочитанные через это имя
> (`__name__`, `__doc__`, `__qualname__`, аннотации, сигнатура), принадлежат
> `wrapper`, а не исходной функции. Это затрудняет отладку, ухудшает
> сообщения об ошибках и ломает инструменты, которые анализируют функции.
> Стандартное решение — декоратор **`functools.wraps`**, который копирует
> метаданные исходной функции в обертку.

---

## 1. Что происходит с метаданными

Возьмем декоратор из предыдущих конспектов и снабдим исходную функцию
docstring и аннотациями:

```python
import time
from typing import Any, Callable


def measure_time(function: Callable[..., Any]) -> Callable[..., Any]:
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start_time = time.perf_counter()
        result = function(*args, **kwargs)
        elapsed_time = time.perf_counter() - start_time
        print(f"{function.__name__} took {elapsed_time:.4f} s")

        return result

    return wrapper


def compute_sum(limit: int) -> int:
    """Return the sum of squares of numbers from 0 to limit - 1."""
    total = 0

    for number in range(limit):
        total += number * number

    return total
```

Сравним характеристики функции до и после декорирования:

```python
print(compute_sum.__name__)         # compute_sum
print(compute_sum.__doc__)          # Return the sum of squares of numbers from 0 to limit - 1.
print(compute_sum.__annotations__)  # {'limit': <class 'int'>, 'return': <class 'int'>}
compute_sum = measure_time(compute_sum)
print(compute_sum.__name__)         # wrapper
print(compute_sum.__doc__)          # None
print(compute_sum.__annotations__)  # {'args': typing.Any, 'kwargs': typing.Any, 'return': typing.Any}
```

Имя `compute_sum` теперь указывает на объект-функцию `wrapper`, поэтому все
атрибуты берутся у обертки. Это и есть **подмена исходного объекта**: функция
под именем `compute_sum` на самом деле называется `wrapper`, не имеет
docstring, а ее аннотации описывают `*args` и `**kwargs`, а не параметр
`limit`.

К чему это приводит:

- в трассировке стека и в сообщениях об ошибках фигурирует `wrapper`, а не имя
  исходной функции. Если декорировано несколько функций, различить их
  по этим сообщениям невозможно;
- встроенная справка `help(compute_sum)` показывает описание обертки, а не
  исходной функции;
- инструменты, анализирующие сигнатуру (`inspect.signature`, автодополнение в
  редакторе, генераторы документации), видят `(*args, **kwargs)`;
- код, который использует `__name__` (например, реестры из предыдущего
  конспекта), получает неверное имя.

### Ручное исправление

Можно скопировать нужные атрибуты в обертку самостоятельно:

```python
def measure_time(function: Callable[..., Any]) -> Callable[..., Any]:
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start_time = time.perf_counter()
        result = function(*args, **kwargs)
        elapsed_time = time.perf_counter() - start_time
        print(f"{function.__name__} took {elapsed_time:.4f} s")

        return result

    wrapper.__name__ = function.__name__
    wrapper.__doc__ = function.__doc__
    return wrapper
```

Но атрибутов много (`__module__`, `__qualname__`, аннотации, словарь
`__dict__`), и копировать их вручную в каждом декораторе неудобно и
ненадежно.

---

## 2. functools.wraps

Модуль `functools` содержит декоратор `wraps`, который выполняет это копирование.
Применяется он к **обертке**, а в качестве аргумента принимает **исходную
функцию**:

```python
import time
from functools import wraps
from typing import Any, Callable


def measure_time(function: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        start_time = time.perf_counter()
        result = function(*args, **kwargs)
        elapsed_time = time.perf_counter() - start_time
        print(f"{function.__name__} took {elapsed_time:.4f} s")

        return result

    return wrapper


@measure_time
def compute_sum(limit: int) -> int:
    """Return the sum of squares of numbers from 0 to limit - 1."""
    total = 0

    for number in range(limit):
        total += number * number

    return total


print(compute_sum.__name__)         # compute_sum
print(compute_sum.__doc__)          # Return the sum of squares of numbers from 0 to limit - 1.
print(compute_sum.__annotations__)  # {'limit': <class 'int'>, 'return': <class 'int'>}
```

`wraps` копирует из исходной функции в обертку атрибуты `__module__`,
`__name__`, `__qualname__`, `__doc__`, аннотации типов, а также обновляет
словарь `__dict__` обертки словарем исходной функции. Благодаря этому
декорированная функция сохраняет имя, docstring, аннотации и сигнатуру
исходной.

Обратите внимание: `@wraps(function)` — декоратор с аргументом (**параметрический
декоратор**). Устройство таких декораторов рассматривается в одном из
следующих конспектов.

---

## 3. Где хранится исходная функция

Подмена имени не удаляет исходную функцию: на нее остается ссылка в
замыкании обертки (независимо от использования `wraps`). Это та самая свободная переменная `function`, с которой
мы познакомились в конспекте про замыкания:

```python
print(compute_sum.__code__.co_freevars)             # ('function',)
print(compute_sum.__closure__[0].cell_contents)     # <function compute_sum at 0x...>
```

Декорированная функция — это объект `wrapper`, у которого исходная функция
хранится в ячейке замыкания.

---

## 4. Практическое правило

Обертка, создаваемая в декораторе, должна быть оформлена с `@wraps(function)`.
Типичный шаблон декоратора:

```python
from functools import wraps
from typing import Any, Callable


def decorator(function: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        # действия перед вызовом
        result = function(*args, **kwargs)
        # действия после вызова
        return result

    return wrapper
```

---

## Вывод

- После выполнения `@decorator` имя функции связано с объектом-оберткой. Атрибуты
  `__name__`, `__doc__`, `__qualname__`, `__annotations__` и сигнатура
  принадлежат обертке: `compute_sum.__name__` равно `'wrapper'`.
- Подмена метаданных ухудшает отладку (трассировка, сообщения об ошибках),
  справку (`help`) и работу инструментов, анализирующих функции.
- `@functools.wraps(function)`, примененный к обертке, копирует метаданные
  исходной функции (имя, docstring, аннотации, сигнатуру).
- Независимо от `wraps` исходная функция хранится в замыкании обертки
  (`__closure__[0].cell_contents`).
- Обертку в декораторах принято всегда оформлять через `@wraps`.

---

## Источники

- [Python documentation. The Python Standard Library — `functools.wraps`](https://docs.python.org/3/library/functools.html#functools.wraps).
