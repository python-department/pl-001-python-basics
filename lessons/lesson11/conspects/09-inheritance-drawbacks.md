# Недостатки наследования

> **Главная мысль.** Наследование создает **самую сильную связь** между
> классами: дочерний класс зависит не только от публичного интерфейса
> родителя, но часто и от его реализации. Это приводит к проблеме **хрупкого
> базового класса**, когда изменение родителя ломает дочерние классы. Кроме
> того, дочерний класс получает все состояние и поведение родителя, в том
> числе то, что может нарушить его собственную логику.

---

## 1. Хрупкий базовый класс

### Зависимость от служебных методов

Методы дочернего класса имеют доступ ко всем атрибутам родителя, включая
служебные. Рассмотрим класс для формирования отчета и дочерний класс,
поддерживающий формат HTML:

```python
class Report:
    def __init__(self, title: str) -> None:
        self.title = title

    def render(self) -> str:
        return self._format_title()

    def _format_title(self) -> str:
        return self.title.upper()


class HtmlReport(Report):
    def render_html(self) -> str:
        return f"<h1>{self._format_title()}</h1>"


print(HtmlReport("sales").render_html())  # <h1>SALES</h1>
```

Все работает, пока в новой версии разработчик `Report` не переименует
служебный метод. Публичный интерфейс класса при этом не изменяется:

```python
class Report:
    def __init__(self, title: str) -> None:
        self.title = title

    def render(self) -> str:
        return self._build_heading()

    def _build_heading(self) -> str:
        return self.title.upper()


class HtmlReport(Report):
    def render_html(self) -> str:
        return f"<h1>{self._format_title()}</h1>"


report = HtmlReport("sales")
print(report.render())  # SALES
print(report.render_html())
# AttributeError: 'HtmlReport' object has no attribute '_format_title'
```

Разработчик родительского класса ничего не нарушил: поддержка служебных
методов не гарантируется. Он может даже не знать о существовании `HtmlReport`.
Однако дочерний класс перестал работать, причем ошибка проявилась не в момент
создания класса, а только при вызове метода.

### Зависимость от устройства публичных методов

Дочерний класс может сломаться, даже если он не использует служебные имена.
Пусть нужно подсчитать количество добавленных элементов. Метод `add_many`
родителя использует метод `add`, поэтому достаточно переопределить только
`add`:

```python
class Storage:
    def __init__(self) -> None:
        self.items = []

    def add(self, item: str) -> None:
        self.items.append(item)

    def add_many(self, new_items: list[str]) -> None:
        for item in new_items:
            self.add(item)


class CountingStorage(Storage):
    def __init__(self) -> None:
        super().__init__()
        self.added_count = 0

    def add(self, item: str) -> None:
        super().add(item)
        self.added_count += 1


storage = CountingStorage()
storage.add_many(["first", "second"])
print(storage.added_count)  # 2
```

Затем разработчик `Storage` решает не вызывать `add` в цикле и добавляет все
элементы одной операцией:

```python
class Storage:
    def __init__(self) -> None:
        self.items = []

    def add(self, item: str) -> None:
        self.items.append(item)

    def add_many(self, new_items: list[str]) -> None:
        self.items.extend(new_items)


class CountingStorage(Storage):
    def __init__(self) -> None:
        super().__init__()
        self.added_count = 0

    def add(self, item: str) -> None:
        super().add(item)
        self.added_count += 1


storage = CountingStorage()
storage.add_many(["first", "second"])
print(storage.items)        # ['first', 'second']
print(storage.added_count)  # 0 -- исключения нет, но результат неверный
```

Оба элемента добавлены, но счетчик остался равен нулю. `CountingStorage` не
переопределяет `add_many`, поэтому вызывается метод родителя. Раньше он вызывал
`self.add(item)`. Так как `self` — объект `CountingStorage`, выполнялся
переопределенный `add`, который и увеличивал счетчик. Новая версия `add_many`
вызывает `self.items.extend` и не обращается к `add`. Переопределенный метод не
выполняется, и `added_count` не изменяется.

Дочерний класс полагался на то, что `add_many` вызывает `add`, хотя это деталь
реализации. Исключение при этом не возникает, поэтому такую ошибку обнаружить
еще труднее, чем `AttributeError` из предыдущего примера.

Это и есть **проблема хрупкого базового класса (fragile base class)**:
изменение родительского класса, безопасное с точки зрения самого этого класса,
нарушает работу дочерних.

---

## 2. Нарушение логики дочернего класса

Вернемся к ромбу, унаследованному от параллелограмма:

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
```

У ромба все стороны равны, и для описания их длины достаточно **одного**
значения. Но вместе с поведением ромб унаследовал состояние параллелограмма и
хранит **два** независимых атрибута, `length` и `width`. Их равенство
обеспечено только в момент инициализации. Любой из них можно перезаписать:

```python
rhombus = Rhombus(4, math.pi / 2)
print(rhombus.__dict__)              # {'length': 4, 'width': 4, 'angle': 1.5707963267948966}
rhombus.length = 10
print(rhombus.__dict__)              # {'length': 10, 'width': 4, 'angle': 1.5707963267948966}
print(rhombus.get_area())            # 40.0
print(isinstance(rhombus, Rhombus))  # True
```

Стороны объекта стали разными: по состоянию это уже не ромб, а
параллелограмм. При этом его тип остался прежним, и ни одна ошибка не
возникла.

Требование, которое должно выполняться для объекта в течение всего времени его
существования, называется **инвариантом**. Инвариант ромба — равенство
сторон. Родительский класс о нем не знает и не обязан его поддерживать.

### Почему это сложно исправить

Можно переопределить в ромбе обе стороны свойствами, которые работают с одним
общим значением:

```python
class Rhombus(Parallelogram):
    def __init__(self, side: float, angle: float) -> None:
        super().__init__(side, side, angle)

    @property
    def length(self) -> float:
        return self._side

    @length.setter
    def length(self, new_length: float) -> None:
        self._side = new_length

    @property
    def width(self) -> float:
        return self._side

    @width.setter
    def width(self, new_width: float) -> None:
        self._side = new_width


rhombus = Rhombus(4, math.pi / 2)
rhombus.length = 10
print(rhombus.length, rhombus.width)  # 10 10
```

Стороны всегда равны, но теперь ромб ведет себя не так, как параллелограмм.
Код, написанный для параллелограмма, вправе ожидать, что длина и ширина
изменяются независимо:

```python
def resize(figure: Parallelogram) -> float:
    figure.length = 10
    figure.width = 2
    return figure.get_area()


print(resize(Parallelogram(4, 3, math.pi / 2)))  # 20.0
print(resize(Rhombus(4, math.pi / 2)))           # 4.0 -- присваивание width изменило и length
```

Функция принимает параллелограмм, а ромб формально является его частным
случаем, но результат оказывается неожиданным. Нарушен **принцип подстановки
Барбары Лисков (Liskov substitution principle)**: объект дочернего класса
должен быть пригоден везде, где ожидается объект родительского, без нарушения
корректности программы.

В геометрии ромб является частным случаем параллелограмма, но геометрические
фигуры не изменяются. Для изменяемых объектов этого недостаточно: дочерний
класс должен выполнять все обещания родителя, включая то, как можно изменять
его состояние.

---

## 3. Другие недостатки

- **Дочерний класс получает все.** Отказаться от части атрибутов и методов
  родителя нельзя: даже ненужные или неподходящие становятся частью
  интерфейса дочернего класса.
- **Глубокие иерархии трудно читать.** Чтобы понять, какой метод будет вызван,
  приходится изучать несколько классов.
- **Множественное наследование усложняет программу.** Поведение зависит от
  MRO, а классы иерархии должны быть согласованы друг с другом по сигнатурам и
  вызовам `super()`.

---

## 4. Когда использовать наследование

Наследование оправдано, когда одновременно выполняются условия:

- объект дочернего класса действительно **является** объектом родительского и
  может использоваться везде вместо него;
- дочернему классу нужен весь интерфейс родителя;
- родительский класс рассчитан на наследование, или оба класса разрабатываются
  вместе.

Если цель — только переиспользовать часть кода, лучше выбрать другой способ.

### Композиция

**Композиция (composition)** — подход, при котором объект не наследуется от
другого класса, а хранит экземпляр этого класса в своем атрибуте и использует
его публичный интерфейс:

```python
class Storage:
    def __init__(self) -> None:
        self.items = []

    def add_many(self, new_items: list[str]) -> None:
        self.items.extend(new_items)


class CountingStorage:
    def __init__(self) -> None:
        self._storage = Storage()
        self.added_count = 0

    def add_many(self, new_items: list[str]) -> None:
        self._storage.add_many(new_items)
        self.added_count += len(new_items)


storage = CountingStorage()
storage.add_many(["first", "second"])
print(storage.added_count)  # 2
```

`CountingStorage` зависит только от публичного метода `add_many` и от того,
что он добавляет все переданные элементы. Как этот метод устроен внутри,
значения не имеет. Наследование выражает отношение «**является**» (is-a), а
композиция — отношение «**содержит**» (has-a).

### Общий интерфейс без наследования

Если объекты нужно передавать в одну и ту же функцию, общий родитель для этого
не обязателен. Благодаря утиной типизации достаточно, чтобы у независимых
классов были методы с одинаковыми именами и сигнатурами, как `get_area` в
[конспекте о полиморфизме](04-polymorphism.md).

---

## Вывод

- Наследование сильно связывает дочерний класс с родительским.
- **Хрупкий базовый класс:** переименование служебного метода или изменение
  внутреннего устройства родителя может сломать дочерние классы, причем иногда
  без всякого исключения.
- Дочерний класс получает все состояние родителя.
- Объект дочернего класса должен быть пригоден везде, где ожидается объект
  родительского (**принцип подстановки Лисков**).
- Для переиспользования кода часто лучше подходит **композиция**, а для
  полиморфизма достаточно **общего интерфейса**.

---

## Источники

- [Python documentation. The Python Tutorial — Private Variables](https://docs.python.org/3/tutorial/classes.html#private-variables);
- [PEP 8 — Designing for Inheritance](https://peps.python.org/pep-0008/#designing-for-inheritance);
- [Raymond Hettinger. Python's super() considered super!](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/).
