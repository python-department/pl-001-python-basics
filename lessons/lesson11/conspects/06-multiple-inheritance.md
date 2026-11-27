# Множественное наследование и поиск атрибутов

> **Главная мысль.** В Python у класса может быть **несколько родительских
> классов**. Атрибут ищется сначала в экземпляре, затем в объекте его класса,
> затем в родительских классах. Порядок обхода классов называется **порядком
> разрешения методов (method resolution order, MRO)**. Он вычисляется один раз
> при создании класса и хранится в атрибуте `__mro__`.

---

## 1. Синтаксис множественного наследования

В Python класс может наследоваться одновременно от нескольких классов. Это
называется **множественным наследованием**. Для него достаточно перечислить
идентификаторы родительских классов в скобках через запятую:

```python
class Flyer:
    def fly(self) -> str:
        return "Flying"

    def move(self) -> str:
        return "Moving through the air"


class Swimmer:
    def swim(self) -> str:
        return "Swimming"

    def move(self) -> str:
        return "Moving through the water"


class Duck(Flyer, Swimmer):
    pass


duck = Duck()
print(duck.fly())   # Flying
print(duck.swim())  # Swimming
print(duck.move())  # Moving through the air
```

Класс `Duck` получает методы обоих родителей. Метод `move` есть и в `Flyer`, и
в `Swimmer`; выбран вариант из `Flyer`, поскольку этот класс указан первым.
Если поменять родителей местами, результат изменится:

```python
class Penguin(Swimmer, Flyer):
    pass


print(Penguin().move())  # Moving through the water
```

---

## 2. Поиск атрибута

Ранее мы рассмотрели поиск атрибута
[в экземпляре и его классе](../../lesson10/conspects/08-instance-and-class-attributes.md).
С учетом наследования выражение `instance.name` вычисляется так:

1. атрибут ищется в пространстве имен **экземпляра**;
2. затем в пространстве имен **объекта класса** этого экземпляра;
3. затем в пространствах имен **родительских классов** в определенном порядке,
   причем последним из них всегда является `object`;
4. если атрибут не найден нигде, возникает `AttributeError`.

Поиск прекращается на первом найденном атрибуте. Как было сказано в
[конспекте о свойствах](03-property.md), исключение составляют свойства:
`property`, найденное в классе или его родителях, имеет приоритет над
атрибутом экземпляра.

Проследим за поиском на следующем примере:

```python
class Device:
    vendor = "Unknown vendor"
    status = "off"
    model = "base model"


class Printer(Device):
    status = "ready"
    model = "printer model"


printer = Printer()
printer.model = "custom model"
print(printer.model)   # custom model   -- найден в экземпляре
print(printer.status)  # ready          -- найден в классе Printer
print(printer.vendor)  # Unknown vendor -- найден в родительском классе Device
print(printer.serial_number)
# AttributeError: 'Printer' object has no attribute 'serial_number'
```

Через объект класса поиск выполняется так же, но без первого шага: сначала в
самом классе, затем в его родителях:

```python
print(Printer.model)   # printer model
print(Printer.vendor)  # Unknown vendor
```

Наследование не меняет правила присваивания: `instance.name = value`
по-прежнему записывает атрибут в сам экземпляр и не изменяет ни его класс, ни
родительские классы.

---

## 3. Порядок разрешения методов (MRO)

Когда родителей несколько, а у них есть свои родители, нужно однозначно
определить порядок обхода всех этих классов. Такой порядок называется
**порядком разрешения методов (method resolution order, MRO)**. Несмотря на
слово «методов» в названии, он используется для поиска любых атрибутов.

MRO класса можно получить из атрибута `__mro__` (кортеж) или в результате
вызова метода `mro()` (список). Рассмотрим следующую иерархию классов:

```python
class Device:
    def get_info(self) -> str:
        return "Device"


class Printer(Device):
    pass


class Scanner(Device):
    def get_info(self) -> str:
        return "Scanner"


class Copier(Printer, Scanner):
    pass
```

Получим MRO класса `Copier`. Если вывести `Copier.__mro__` напрямую, каждый
класс будет представлен строкой вида `<class '__main__.Copier'>`. Чтобы
результат было проще читать, составим список из значений атрибута `__name__`
этих классов:

```python
print([parent.__name__ for parent in Copier.__mro__])
# ['Copier', 'Printer', 'Scanner', 'Device', 'object']
```

Классы записаны в MRO в том порядке, в котором в них выполняется поиск
атрибутов. При обращении к атрибуту через `Copier` сначала проверяется сам класс
`Copier`, затем `Printer`, `Scanner`, `Device` и в последнюю очередь `object`.
Поиск прекращается на первом классе, в пространстве имен которого найден
атрибут. Если обращение выполняется через экземпляр `Copier`, то перед обходом
этих классов проверяется пространство имен самого экземпляра.

Структура, в которой два родителя класса имеют общего предка, как в этой
иерархии, называется **ромбовидным наследованием (diamond inheritance)**:

```
        Device
       /      \
  Printer    Scanner
       \      /
        Copier
```

Вызовем метод `get_info` у экземпляра `Copier`:

```python
print(Copier().get_info())  # Scanner
```

В `Printer` метода нет, но поиск **не переходит** сразу к родителю `Printer`,
классу `Device`. В соответствии с MRO сначала проверяется `Scanner`, и только
после него — общий предок.

### Правила построения MRO

Python строит MRO по алгоритму **C3-линеаризации**. Для практической работы
достаточно знать его свойства:

1. класс всегда расположен **раньше своих родителей**;
2. родители сохраняют **порядок, в котором перечислены** в заголовке класса;
3. каждый класс входит в MRO **ровно один раз**;
4. взаимный порядок классов из MRO любого родителя **сохраняется** в MRO
   дочернего класса.

---

## Вывод

- Класс может иметь **несколько родителей**: `class Child(First, Second):`.
- `instance.name` ищется в экземпляре, затем в классе экземпляра, затем в
  родительских классах; последним проверяется `object`.
- Порядок обхода классов задает **MRO**, доступный через `__mro__` и `mro()`.
- В MRO класс расположен раньше своих родителей, порядок родителей из
  заголовка сохраняется, и каждый класс встречается один раз.
- При **ромбовидном наследовании** общий предок проверяется после всех своих
  наследников, входящих в MRO.

---

## Источники

- [Python documentation. The Python Tutorial — Multiple Inheritance](https://docs.python.org/3/tutorial/classes.html#multiple-inheritance);
- [Python documentation. The Python 2.3 Method Resolution Order](https://docs.python.org/3/howto/mro.html);
- [Python documentation. The Python Language Reference — Custom classes](https://docs.python.org/3/reference/datamodel.html#custom-classes).
