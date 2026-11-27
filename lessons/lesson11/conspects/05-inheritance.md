# Наследование

> **Главная мысль.** **Наследование (inheritance)** — создание класса на основе
> другого класса. **Дочерний класс** получает доступ ко всем атрибутам и
> методам **родительского**, может добавлять новые и переопределять
> существующие. Наследование подчеркивает связь, при которой один тип
> является частным случаем другого, и позволяет переиспользовать код, не копируя его. Если
> атрибут не найден в дочернем классе, поиск продолжается в родительском.

---

## 1. Мотивация

Вернемся к полиморфизму. Следующая функция принимает объекты, описывающие
геометрические фигуры, и для расчета общей площади вызывает их метод
`get_area`:

```python
from typing import Any


def get_total_area(figures: list[Any]) -> float:
    total_area = 0

    for figure in figures:
        total_area += figure.get_area()

    return total_area
```

Пусть есть класс параллелограмма. Он хранит длины двух смежных сторон и угол
между ними в радианах, а площадь вычисляет как произведение сторон на синус
угла:

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

    def get_perimeter(self) -> float:
        return 2 * (self.length + self.width)
```

Теперь нужен класс ромба, который тоже можно передать в `get_total_area`.
Благодаря утиной типизации достаточно определить класс с методом `get_area`:

```python
@dataclass
class Rhombus:
    length: float
    width: float
    angle: float

    def get_area(self) -> float:
        return self.length * self.width * math.sin(self.angle)

    def get_perimeter(self) -> float:
        return 2 * (self.length + self.width)


figures = [Parallelogram(5, 4, math.pi / 6), Rhombus(4, 4, math.pi / 6)]
print(round(get_total_area(figures), 2))  # 18.0
```

Код работает, но классы совпадают полностью. Это не случайно: **ромб — частный
случай параллелограмма**, у которого все стороны равны. У них одинаковое
состояние (длины сторон и угол) и одинаковая функциональность (вычисление
площади и периметра). Дублирование приводит к проблемам:

- исправление ошибки в одном классе нужно не забыть повторить в другом;
- новый метод параллелограмма придется копировать в ромб;
- связь между фигурами никак не отражена в коде: для интерпретатора это два
  не связанных друг с другом типа.

Все три проблемы решает наследование.

---

## 2. Синтаксис наследования

Родительский класс указывается в круглых скобках после имени создаваемого
класса:

```python
class Parent:
    pass


class Child(Parent):
    pass
```

Используется следующая терминология:

| Класс | Другие названия |
|---|---|
| `Parent` | **родительский**, **базовый (base class)**, **суперкласс (superclass)** |
| `Child` | **дочерний**, **производный (derived class)**, **подкласс (subclass)** |

Объявим ромб дочерним классом параллелограмма. Класс `Parallelogram` остается
таким же, как в разделе 1:

```python
class Rhombus(Parallelogram):
    pass


rhombus = Rhombus(4, 4, math.pi / 6)
print(round(rhombus.get_area(), 2))  # 8.0
print(rhombus.get_perimeter())       # 16
print(rhombus.__dict__)              # {'length': 4, 'width': 4, 'angle': 0.5235987755982988}
```

В классе `Rhombus` не определено ни одного метода. Тем не менее можно создать
его экземпляр и вызвать у него `get_area` и `get_perimeter`. Эти методы, а
также `__init__`, созданный декоратором `dataclass`, унаследованы от
`Parallelogram`.

При этом `Rhombus` использует инициализацию родителя без изменений, поэтому две
равные длины приходится передавать по отдельности, и ничто не мешает передать
разные значения.

---

## 3. Как работает наследование

При наследовании атрибуты **не копируются** в дочерний класс. Объект
дочернего класса лишь хранит ссылку на родительские классы. Они доступны
через атрибут `__bases__`:

```python
print(Rhombus.__bases__)                           # (<class '__main__.Parallelogram'>,)
print("get_area" in Rhombus.__dict__)              # False
print("get_area" in Parallelogram.__dict__)        # True
print(Rhombus.get_area is Parallelogram.get_area)  # True
```

Когда атрибут не найден ни в экземпляре, ни в его классе, поиск продолжается в
родительском классе. При вызове `rhombus.get_area()`:

1. атрибут `get_area` ищется в пространстве имен экземпляра `rhombus`;
2. затем в пространстве имен класса `Rhombus`;
3. затем в пространстве имен класса `Parallelogram`, где он и найден.

Найденная функция вызывается с экземпляром `rhombus` в качестве `self`. Таким
образом, метод родителя работает с состоянием экземпляра дочернего класса.

Поскольку поиск выполняется при каждом обращении, изменения родительского
класса сразу видны дочерним.

### Класс object

Любой класс наследуется от встроенного класса **`object`**, и указывать это
явно не нужно. Если родительский класс не указан, `object` неявно становится
непосредственным родителем. Если родитель указан, класс наследуется от
`object` через него: цепочка родительских классов заканчивается на `object`.

```python
print(Rhombus.__bases__)        # (<class '__main__.Parallelogram'>,)
print(Parallelogram.__bases__)  # (<class 'object'>,)
```

Поэтому реализации многих специальных методов, определенных в `object`,
доступны любому классу.

---

## 4. Расширение дочернего класса

Дочерний класс может определять собственные атрибуты и методы. Добавим ромбу
вычисление длин диагоналей. Для произвольного параллелограмма эти формулы
неверны, поэтому метод относится только к ромбу:

```python
class Rhombus(Parallelogram):
    def get_diagonals(self) -> tuple[float, float]:
        half_angle = self.angle / 2
        first_diagonal = 2 * self.length * math.cos(half_angle)
        second_diagonal = 2 * self.length * math.sin(half_angle)
        return first_diagonal, second_diagonal


rhombus = Rhombus(4, 4, math.pi / 2)
print(rhombus.get_area())  # 16.0
first_diagonal, second_diagonal = rhombus.get_diagonals()
print(round(first_diagonal, 2), round(second_diagonal, 2))  # 5.66 5.66
parallelogram = Parallelogram(5, 4, math.pi / 2)
parallelogram.get_diagonals()
# AttributeError: 'Parallelogram' object has no attribute 'get_diagonals'
```

Наследование действует **только в одну сторону**: дочерний класс видит
атрибуты родителя, а родитель об атрибутах дочернего класса ничего не знает.

---

## 5. Переопределение методов

Если имя атрибута дочернего класса совпадает с именем атрибута родительского
класса, поиск раньше найдет атрибут дочернего класса. Это называется
**переопределением (overriding)**. Родительский класс при этом не изменяется:

```python
class Employee:
    def __init__(self, name: str) -> None:
        self.name = name

    def get_role(self) -> str:
        return "employee"

    def describe(self) -> str:
        return f"{self.name}: {self.get_role()}"


class Manager(Employee):
    def get_role(self) -> str:
        return "manager"


print(Employee("Alice").describe())  # Alice: employee
print(Manager("Bob").describe())     # Bob: manager
```

Обратите внимание на вызов `Manager("Bob").describe()`. Метод `describe`
найден в родительском классе, но выражение `self.get_role()` внутри него
вычисляется по общему правилу: поиск начинается с экземпляра, на который
ссылается `self`, и продолжается в **его** классе, то есть в `Manager`.
Поэтому вызывается переопределенный метод.

Это тоже проявление полиморфизма: один и тот же код метода `describe` ведет
себя по-разному в зависимости от класса экземпляра.

---

## 6. Проверка типов при наследовании

Экземпляр дочернего класса является также экземпляром всех его родительских
классов. Это учитывает функция `isinstance`. Для проверки отношения между
самими классами используется функция `issubclass`:

```python
rhombus = Rhombus(4, 4, math.pi / 2)
print(isinstance(rhombus, Rhombus))        # True
print(isinstance(rhombus, Parallelogram))  # True
print(isinstance(rhombus, object))         # True
print(issubclass(Rhombus, Parallelogram))  # True
print(issubclass(Parallelogram, Rhombus))  # False
print(type(rhombus) is Parallelogram)      # False
```

В отличие от `isinstance`, сравнение с результатом `type()` наследование не
учитывает.

---

## 7. Name mangling при наследовании

Дочерний класс может случайно переопределить служебный атрибут родителя,
использовав то же имя:

```python
class Account:
    def __init__(self) -> None:
        self._history = []

    def add_event(self, event: str) -> None:
        self._history.append(event)


class ReportAccount(Account):
    def prepare_report(self) -> None:
        self._history = "report history"  # затирает список родителя


account = ReportAccount()
account.prepare_report()
account.add_event("deposit")
# AttributeError: 'str' object has no attribute 'append'
```

Объект `account` один, и в его пространстве имен может быть только один
атрибут с именем `_history`. Поэтому в методах `add_event` и `prepare_report`
выражение `self._history` обращается к одному и тому же атрибуту, хотя методы
определены в разных классах. Метод `prepare_report` заменяет список строкой, и
после этого `add_event` не может вызвать `append`.

Такого конфликта не будет, если имя атрибута начинается с двух подчеркиваний.
При name mangling к этому имени добавляется имя класса, в теле которого оно
записано. В результате у экземпляра появляются два разных атрибута:

```python
class Account:
    def __init__(self) -> None:
        self.__history = []

    def add_event(self, event: str) -> None:
        self.__history.append(event)


class ReportAccount(Account):
    def prepare_report(self) -> None:
        self.__history = "report history"


account = ReportAccount()
account.prepare_report()
account.add_event("deposit")
print(account.__dict__)
# {'_Account__history': ['deposit'], '_ReportAccount__history': 'report history'}
```

---

## Вывод

- **Наследование** создает класс на основе другого и отражает связь между
  частным и общим случаем: ромб является параллелограммом.
- Синтаксис: `class Child(Parent):`.
- Дочерний класс **не копирует** атрибуты родителя. Если атрибут не найден в
  экземпляре и его классе, поиск продолжается в родительском классе.
- Дочерний класс может **добавлять** атрибуты и **переопределять**
  унаследованные. Родительский класс при этом не изменяется.
- Вызов `self.method()` в методе родителя использует переопределенный метод,
  если `self` — экземпляр дочернего класса.
- Все классы прямо или косвенно наследуются от **`object`**.
- `isinstance` и `issubclass` учитывают наследование.
- Имена с префиксом `__` защищают атрибуты родителя и дочернего класса от
  конфликта.

---

## Источники

- [Python documentation. The Python Tutorial — Inheritance](https://docs.python.org/3/tutorial/classes.html#inheritance);
- [Python documentation. Built-in Functions — isinstance](https://docs.python.org/3/library/functions.html#isinstance);
- [Python documentation. Built-in Functions — issubclass](https://docs.python.org/3/library/functions.html#issubclass).
