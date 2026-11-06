# Композиция декораторов

> **Главная мысль.** К одной функции можно применить **несколько декораторов**
> подряд, записав их друг под другом. Запись
> `@first` / `@second` / `def function` эквивалентна
> `function = first(second(function))`: декораторы применяются **снизу вверх**
> (ближайший к `def` — первым), а при вызове обертки выполняются **сверху
> вниз** (внешняя обертка получает управление первой). Порядок декораторов
> влияет на результат, поэтому его нужно выбирать осознанно.

---

## 1. Синтаксис и эквивалентная запись

Несколько строк с `@` перед `def` задают **композицию** декораторов:

```python
@decorator_a
@decorator_b
@decorator_c
def function() -> None:
    ...
```

Эта запись равносильна цепочке вызовов, в которой результат каждого
декоратора передается следующему:

```python
def function() -> None:
    ...


function = decorator_a(decorator_b(decorator_c(function)))
```

Первым вызывается декоратор, расположенный **ближе всего к `def`**
(`decorator_c`). Он получает исходную функцию и возвращает обертку. Эта
обертка передается в `decorator_b`, его результат — в `decorator_a`. В имени
`function` остается обертка, созданная последним декоратором из списка, то
есть самым верхним.

---

## 2. Порядок применения и порядок выполнения

Чтобы увидеть порядок применения, определим два декоратора, печатающих сообщения при
декорировании и при вызове:

```python
from functools import wraps
from typing import Any, Callable


def decorator_one(function: Callable[..., Any]) -> Callable[..., Any]:
    print("Decorating with decorator_one")

    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        print("decorator_one: before")
        result = function(*args, **kwargs)
        print("decorator_one: after")
        return result

    return wrapper


def decorator_two(function: Callable[..., Any]) -> Callable[..., Any]:
    print("Decorating with decorator_two")

    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        print("decorator_two: before")
        result = function(*args, **kwargs)
        print("decorator_two: after")
        return result

    return wrapper


@decorator_one
@decorator_two
def say_hello() -> None:
    print("Hello")


print("--- call ---")
say_hello()
```

Вывод:
```
Decorating with decorator_two
Decorating with decorator_one
--- call ---
decorator_one: before
decorator_two: before
Hello
decorator_two: after
decorator_one: after
```

Из вывода следуют два правила:

1. **Применение — снизу вверх.** Сначала декорирует `decorator_two`
   (ближайший к `def`), затем `decorator_one`.
2. **Выполнение при вызове — сверху вниз на входе и снизу вверх на выходе.**
   Вызов `say_hello()` попадает в самую внешнюю обертку (`decorator_one`),
   она вызывает следующую (`decorator_two`), та — исходную функцию. Результат
   возвращается в обратном порядке.

Структуру можно изобразить как вложенные вызовы:

```
say_hello()  →  wrapper декоратора decorator_one
                  └─ вызывает wrapper декоратора decorator_two
                       └─ вызывает исходную функцию say_hello
```

Именно из-за такой вложенности запись с несколькими декораторами называют
**композицией**, по аналогии с композицией функций в математике:
`decorator_one(decorator_two(say_hello))`.

Благодаря `functools.wraps` метаданные сохраняются во всей цепочке:
`say_hello.__name__` остается равным `'say_hello'`.

---

## 3. Порядок влияет на результат

Если декораторы изменяют результат функции, перестановка меняет итог. Пусть
первый декоратор оборачивает результат в квадратные скобки, а второй — в
кавычки:

```python
from functools import wraps
from typing import Any, Callable


def add_brackets(function: Callable[..., str]) -> Callable[..., str]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> str:
        return f"[{function(*args, **kwargs)}]"
    return wrapper


def add_quotes(function: Callable[..., str]) -> Callable[..., str]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> str:
        return f'"{function(*args, **kwargs)}"'
    return wrapper


@add_brackets
@add_quotes
def get_title() -> str:
    return "Python"


@add_quotes
@add_brackets
def get_same_title() -> str:
    return "Python"


print(get_title())       # ["Python"]
print(get_same_title())  # "[Python]"
```

В первом случае сначала применяется `add_quotes` (`"Python"`), и результат
затем оборачивается в скобки. Во втором — наоборот.

Порядок важен не только для преобразования результата. Рассмотрим более
практичный пример: журналирование вызовов и проверка аргументов.

```python
from functools import wraps
from typing import Any, Callable


def log_call(function: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        print(f"Calling {function.__name__}{args}")
        return function(*args, **kwargs)

    return wrapper


def require_positive(function: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(function)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        for argument in args:
            if argument <= 0:
                raise ValueError("arguments must be positive")

        return function(*args, **kwargs)

    return wrapper


@log_call
@require_positive
def compute_area(width: float, height: float) -> float:
    return width * height


@require_positive
@log_call
def compute_perimeter(width: float, height: float) -> float:
    return 2 * (width + height)


print(compute_area(2, 3))
print(compute_perimeter(2, 3))
compute_area(-1, 3)
```

Вывод:
```
Calling compute_area(2, 3)
6
Calling compute_perimeter(2, 3)
10
Calling compute_area(-1, 3)
ValueError: arguments must be positive
```

Некорректный вызов `compute_area(-1, 3)` сначала попадает в `log_call` и
записывается в журнал, после чего `require_positive` вызывает `ValueError`.
Для `compute_perimeter(-1, 3)` порядок обратный: `require_positive` откажет
раньше, и в журнал некорректный вызов не попадет. Какой вариант нужен —
зависит от задачи, но выбор определяется именно порядком декораторов.

Практическое правило: декоратор, который должен выполняться **раньше всех**
(например, проверка прав доступа), записывается **выше**; декоратор, который
должен быть ближе всего к исходной функции, — **ниже**.

---

## 4. Композиция без синтаксиса `@`

Поскольку `@` — сокращенная запись, ту же композицию можно записать вручную.
Это бывает полезно, если набор декораторов формируется динамически:

```python
decorators = [add_brackets, add_quotes]


def get_title() -> str:
    return "Python"


for decorator in reversed(decorators):
    get_title = decorator(get_title)
print(get_title())  # ["Python"]
```

Список записан в том же порядке, в каком декораторы стояли бы над `def`
сверху вниз; применять их нужно с конца (`reversed`), чтобы нижний декоратор
был применен первым.

---

## Вывод

- Несколько декораторов над одним `def` образуют композицию:
  `@a @b def f` ≡ `f = a(b(f))`.
- Применяются декораторы **снизу вверх**, то есть первым декорирует тот, что
  ближе к `def`. При вызове управление передается **сверху вниз**: сначала
  выполняется верхняя обертка, затем нижняя, затем исходная функция; результат
  возвращается в обратном порядке.
- Перестановка декораторов может изменить результат и поведение (формат
  результата, момент проверки аргументов, содержимое журнала), поэтому
  порядок нужно выбирать намеренно.
- `functools.wraps` в каждом декораторе сохраняет метаданные исходной функции
  через всю цепочку оберток.

---

## Источники

- [Python documentation. The Python Language Reference — Function definitions](https://docs.python.org/3/reference/compound_stmts.html#function-definitions);
- [PEP 318 — Decorators for Functions and Methods](https://peps.python.org/pep-0318/).
