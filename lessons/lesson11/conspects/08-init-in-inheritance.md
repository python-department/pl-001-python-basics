# Наследование __init__

> **Главная мысль.** `__init__` наследуется и переопределяется по тем же
> правилам, что и остальные методы. Если дочерний класс определяет свой
> `__init__`, родительский **не вызывается автоматически**: это нужно сделать
> явно через `super().__init__(...)`. При множественном наследовании `__init__`
> родителей можно вызвать явно по имени класса. Но если у родителей есть общий
> предок и каждый из них вызывает его `__init__`, этот метод выполнится
> несколько раз. Этого позволяет избежать кооперативная инициализация.

---

## 1. Дочерний класс без собственного __init__

Если дочерний класс не определяет `__init__`, при создании экземпляра по MRO
находится и вызывается `__init__` родителя. Поэтому дочерний класс принимает те же
аргументы:

```python
class Employee:
    def __init__(self, name: str) -> None:
        self.name = name


class Manager(Employee):
    pass


manager = Manager("Bob")
print(manager.__dict__)  # {'name': 'Bob'}
```

---

## 2. Переопределение без вызова родителя

Собственный `__init__` дочернего класса **заменяет** родительский. Атрибуты,
которые создавал родитель, не появятся:

```python
class Manager(Employee):
    def __init__(self, name: str, team_size: int) -> None:
        self.team_size = team_size


manager = Manager("Bob", 5)
print(manager.__dict__)  # {'team_size': 5}
print(manager.name)
# AttributeError: 'Manager' object has no attribute 'name'
```

Атрибуты экземпляра не наследуются сами по себе: они появляются, только когда
выполняется присваивание. Присваивание `self.name = name` находится в
`Employee.__init__`, а этот метод не вызывался.

---

## 3. Вызов родительского __init__

Чтобы выполнить инициализацию родителя и дополнить ее, используется
`super()`:

```python
class Manager(Employee):
    def __init__(self, name: str, team_size: int) -> None:
        super().__init__(name)
        self.team_size = team_size


manager = Manager("Bob", 5)
print(manager.__dict__)  # {'name': 'Bob', 'team_size': 5}
```

Вызывать `super().__init__()` обычно следует **в начале** метода. Тогда
собственный код дочернего класса работает с уже инициализированной
родительской частью состояния и может ее использовать. Кроме того, родитель не
перезапишет атрибуты, созданные дочерним классом.

В [конспекте о наследовании](05-inheritance.md) класс `Rhombus` использовал
`__init__` параллелограмма без изменений. Поэтому при создании ромба длину
стороны приходилось передавать дважды, и можно было передать два разных
значения. Собственный `__init__` позволяет принимать одну длину и передавать
ее родителю в качестве обеих сторон:

```python
import math
from dataclasses import dataclass


@dataclass
class Parallelogram:
    length: float
    width: float
    angle: float

    def get_area(self) -> float:
        return self.length * self.width * math.sin(self.angle)


class Rhombus(Parallelogram):
    def __init__(self, side: float, angle: float) -> None:
        super().__init__(side, side, angle)


rhombus = Rhombus(4, math.pi / 2)
print(rhombus.get_area())  # 16.0
print(rhombus.__dict__)    # {'length': 4, 'width': 4, 'angle': 1.5707963267948966}
```

Сигнатура `__init__` дочернего класса может отличаться от родительской. Важно
лишь, чтобы в `super().__init__()` были переданы аргументы, которые ожидает
родитель.

---

## 4. Проблема множественного наследования

Рассмотрим класс с двумя родителями, каждому из которых нужны свои аргументы:

```python
class Printer:
    def __init__(self, paper_format: str) -> None:
        self.paper_format = paper_format


class Scanner:
    def __init__(self, resolution: int) -> None:
        self.resolution = resolution


class Copier(Printer, Scanner):
    def __init__(self, paper_format: str, resolution: int) -> None:
        super().__init__(paper_format, resolution)


copier = Copier("A4", 600)
# TypeError: Printer.__init__() takes 2 positional arguments but 3 were given
```

`super().__init__()` вызывает **только один** метод — первый найденный после
`Copier` в MRO, то есть `Printer.__init__`. Все переданные аргументы получает
он, а между родителями они не распределяются. `Printer.__init__` принимает
только `paper_format`, поэтому возникает `TypeError`.

Если передать в `super().__init__()` только `paper_format`, ошибки не будет. Но
`Printer.__init__` сам `super()` не вызывает, поэтому инициализация `Scanner`
не выполнится.

### Явные вызовы

Можно вызвать `__init__` каждого родителя по имени класса:

```python
class Copier(Printer, Scanner):
    def __init__(self, paper_format: str, resolution: int) -> None:
        Printer.__init__(self, paper_format)
        Scanner.__init__(self, resolution)


copier = Copier("A4", 600)
print(copier.__dict__)  # {'paper_format': 'A4', 'resolution': 600}
```

Для независимых классов это работает. Однако при ромбовидном наследовании,
когда у `Printer` и `Scanner` есть общий предок, его `__init__` будет вызван
дважды: из каждого родителя. Повторная инициализация может сбросить уже
измененное состояние или повторно выполнить действие, например
зарегистрировать объект.

### Позиционные аргументы и super()

Если просто добавить `super().__init__()` во все классы, возникает другая
трудность. Класс не знает, какой `__init__` будет следующим в MRO, а значит,
не знает, какие аргументы ему передать. Нужен единый способ передачи
аргументов по всей цепочке.

---

## 5. Кооперативный __init__

Корректная инициализация при множественном наследовании строится по следующим
правилам:

1. каждый класс объявляет свои параметры **только именованными** (после `*`);
2. каждый `__init__` дополнительно принимает **`**kwargs`** — аргументы,
   предназначенные другим классам;
3. каждый `__init__`, включая метод общего предка, вызывает
   **`super().__init__(**kwargs)`**;
4. каждый класс забирает из аргументов только свои и **не передает их**
   дальше.

```python
from typing import Any


class Device:
    def __init__(self, *, name: str, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.name = name


class Printer(Device):
    def __init__(self, *, paper_format: str, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.paper_format = paper_format


class Scanner(Device):
    def __init__(self, *, resolution: int, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.resolution = resolution


class Copier(Printer, Scanner):
    def __init__(self, *, copies_limit: int, **kwargs: Any) -> None:
        super().__init__(**kwargs)
        self.copies_limit = copies_limit


copier = Copier(
    name="Office device",
    paper_format="A4",
    resolution=600,
    copies_limit=100,
)
print(copier.__dict__)
# {'name': 'Office device', 'resolution': 600,
#  'paper_format': 'A4', 'copies_limit': 100}
```

MRO класса `Copier`: `Copier`, `Printer`, `Scanner`, `Device`, `object`.
Аргументы передаются по цепочке, и на каждом шаге их становится меньше:

| `__init__` класса | Забирает | Передает дальше в `**kwargs` |
|---|---|---|
| `Copier` | `copies_limit` | `name`, `paper_format`, `resolution` |
| `Printer` | `paper_format` | `name`, `resolution` |
| `Scanner` | `resolution` | `name` |
| `Device` | `name` | ничего |
| `object` | — | — |

Атрибуты создаются в обратном порядке: каждый метод сначала вызывает
`super().__init__()` и только после возврата из него выполняет свое
присваивание. Поэтому первым в `__dict__` появляется `name`.

Каждый класс по-прежнему можно использовать отдельно:

```python
printer = Printer(name="Home printer", paper_format="A5")
print(printer.__dict__)  # {'name': 'Home printer', 'paper_format': 'A5'}
```

### Почему super() вызывается и в Device

`Device` наследуется только от `object`, но вызывать в нем
`super().__init__()` все равно нужно. Если дочерний класс использует несколько
родителей, после `Device` в его MRO может оказаться другой класс, и без этого
вызова он не будет инициализирован.

### Ошибки в аргументах

В конце цепочки находится `object.__init__`, который не принимает
аргументов. Если какой-то аргумент не забрал ни один класс, например из-за
опечатки в имени, он дойдет до `object` и вызовет `TypeError`:

```python
Printer(name="Home printer", paper_format="A5", colour="black")
# TypeError: object.__init__() takes exactly one argument (the instance to initialize)
```

Если обязательный аргумент не передан, ошибку возбудит тот `__init__`,
которому он нужен:

```python
Printer(paper_format="A5")
# TypeError: Device.__init__() missing 1 required keyword-only argument: 'name'
```

### Ограничения подхода

Все классы иерархии должны следовать одним правилам. Класс, `__init__`
которого не вызывает `super()`, прерывает цепочку. Поэтому множественное
наследование следует проектировать заранее. Объединить таким способом
произвольные готовые классы получается не всегда.

---

## Вывод

- `__init__` ищется по MRO, как любой другой метод. Если в дочернем классе его
  нет, используется родительский.
- Собственный `__init__` дочернего класса заменяет родительский. Чтобы
  инициализировать унаследованную часть состояния, нужен вызов
  **`super().__init__(...)`**, обычно в начале метода.
- Один вызов `super().__init__()` запускает только следующий в MRO метод.
  Остальные выполняются, если каждый метод цепочки тоже вызывает `super()`.
- При множественном наследовании с кооперативной инициализацией в каждом
  `__init__` следует:
  - объявить собственные параметры только именованными (после `*`);
  - принимать остальные аргументы в `**kwargs`;
  - передавать их дальше вызовом `super().__init__(**kwargs)`.

---

## Источники

- [Python documentation. Built-in Functions — super](https://docs.python.org/3/library/functions.html#super);
- [Python documentation. The Python Language Reference — object.\_\_init\_\_](https://docs.python.org/3/reference/datamodel.html#object.__init__);
- [Raymond Hettinger. Python's super() considered super!](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/).
