# Параметрические декораторы (фабрики декораторов)

> **Главная мысль.** Декоратор принимает ровно один аргумент — декорируемый
> объект, поэтому передать ему настройки напрямую нельзя. Когда поведение
> декоратора нужно настраивать (число повторов, единицы измерения времени,
> допустимые исключения), используют **параметрический декоратор** — вернее,
> **фабрику декораторов (decorator factory)**: функцию, которая принимает
> параметры и **возвращает декоратор**. Запись `@factory(argument)` сначала
> вызывает фабрику, а затем применяет возвращенный ею декоратор к функции.
> Получается **три уровня вложенности**: фабрика → декоратор → обертка.

---

## 1. Постановка задачи

Пусть требуется декоратор, который вызывает функцию несколько раз подряд.
Число повторов должно задаваться в месте применения:

```python
@repeat(3)
def say_hello(name: str) -> None:
    print(f"Hello, {name}!")
```

Здесь после `@` стоит не имя декоратора, а **выражение с вызовом**
`repeat(3)`. Правило применения `@` не меняется: Python вычисляет выражение
после `@`, получает вызываемый объект (результат вычисления этого выражения)
и вызывает его с декорируемой функцией. Для `@repeat(3)` таким объектом
является значение, возвращенное вызовом `repeat(3)`. Следовательно,
`repeat(3)` должен вернуть **декоратор** — функцию, принимающую функцию:

```python
decorator = repeat(3)  # 1. вызов фабрики, результат -- декоратор


def say_hello(name: str) -> None:
    print(f"Hello, {name}!")


say_hello = decorator(say_hello)  # 2. применение декоратора
```

Запись `@repeat(3)` эквивалентна `say_hello = repeat(3)(say_hello)`.

---

## 2. Три уровня вложенности

Реализуем фабрику `repeat`:

```python
from functools import wraps
from typing import Any, Callable


def repeat(times: int) -> Callable[[Callable[..., Any]], Callable[..., Any]]:
    def decorator(function: Callable[..., Any]) -> Callable[..., Any]:
        @wraps(function)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            result = None

            for attempt in range(times):
                result = function(*args, **kwargs)

            return result

        return wrapper

    return decorator


@repeat(3)
def say_hello(name: str) -> None:
    print(f"Hello, {name}!")


say_hello("Alice")
```

Вывод:
```
Hello, Alice!
Hello, Alice!
Hello, Alice!
```

Роль каждого уровня:

| Уровень | Что принимает | Что возвращает | Когда выполняется |
|---|---|---|---|
| `repeat(times)` — фабрика | параметры настройки | декоратор | один раз при вычислении выражения после `@` |
| `decorator(function)` — декоратор | декорируемую функцию | обертку | один раз при декорировании (выполнении `def`) |
| `wrapper(*args, **kwargs)` — обертка | аргументы вызова | результат функции | при каждом вызове декорированной функции |

Параметр `times` доступен в функции `wrapper` благодаря замыканию: обертка вложена в
декоратор, а декоратор — в фабрику, поэтому поиск имени идет по цепочке
объемлющих областей видимости (E в правиле LEGB) на два уровня вверх:

```python
print(say_hello.__code__.co_freevars)  # ('function', 'times')
```

Так как `repeat` вызывается при каждом применении `@repeat(...)`, разные функции
могут получить разные настройки, и каждая из них хранит собственное значение
`times` в своей ячейке:

```python
@repeat(2)
def say_goodbye(name: str) -> None:
    print(f"Goodbye, {name}!")


say_goodbye("Bob")  # дважды
say_hello("Bob")    # трижды
```

---

## 3. Тот же механизм, что и в `wraps`

Декоратор `@wraps(function)`, который мы использовали в предыдущих конспектах,
— тоже параметрический. Функция `wraps` принимает исходную функцию
(параметр) и возвращает декоратор, который применяется к обертке:

```python
@wraps(function)
def wrapper(*args: Any, **kwargs: Any) -> Any:
    ...
```

равносильно

```python
def wrapper(*args: Any, **kwargs: Any) -> Any:
    ...


wrapper = wraps(function)(wrapper)
```

---

## 4. Типичная ошибка: пропущенные скобки

Фабрику необходимо **вызвать**, даже если все ее параметры имеют значения по
умолчанию. Если написать `@repeat` вместо `@repeat(3)`, Python применит саму
фабрику к функции, ничего не сообщив об ошибке:

```python
@repeat
def say_hello(name: str) -> None:
    print(f"Hello, {name}!")


print(say_hello.__name__)  # decorator
say_hello("Alice")         # ничего не выводится
```

Запись равна `say_hello = repeat(say_hello)`: параметр `times` получил функцию,
а `say_hello` стала ссылаться на внутреннюю функцию `decorator`. При вызове
`say_hello("Alice")` выполняется `decorator("Alice")`, результатом которого
является необработанная обертка, но не вызов исходной функции.

---

## 5. Шаблон параметрического декоратора

```python
from functools import wraps
from typing import Any, Callable


def factory(parameter: Any) -> Callable[[Callable[..., Any]], Callable[..., Any]]:
    # 1. Выполняется при вычислении выражения после @: параметры фабрики.
    def decorator(function: Callable[..., Any]) -> Callable[..., Any]:
        # 2. Выполняется один раз при декорировании функции.
        @wraps(function)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            # 3. Выполняется при каждом вызове функции.
            result = function(*args, **kwargs)
            return result
        return wrapper
    return decorator


@factory("setting")
def function() -> None:
    ...
```

Запись `@factory("setting")` равносильна `function = factory("setting")(function)`.

---

## Вывод

- Декоратор принимает одну функцию, поэтому настройки передаются через
  **фабрику декораторов** — функцию, которая принимает параметры и возвращает
  декоратор.
- `@factory(argument)` выполняется в два шага: вызов `factory(argument)` дает
  декоратор, затем декоратор применяется к функции:
  `function = factory(argument)(function)`.
- Структура из трех уровней: фабрика (параметры) → декоратор (функция) →
  обертка (аргументы вызова). Параметры фабрики доступны обертке через
  замыкание.
- Скобки у фабрики обязательны, даже если параметров нет или они имеют
  значения по умолчанию: `@repeat` и `@repeat(3)` — разные конструкции.
- Сам `functools.wraps` — параметрический декоратор.

---

## Источники

- [Python documentation. The Python Language Reference — Function definitions](https://docs.python.org/3/reference/compound_stmts.html#function-definitions);
- [Python documentation. The Python Standard Library — `functools.wraps`](https://docs.python.org/3/library/functools.html#functools.wraps);
- [PEP 318 — Decorators for Functions and Methods](https://peps.python.org/pep-0318/).
