# Свойства: property

> **Главная мысль.** **`property`** — встроенный класс, который позволяет
> настроить доступ к атрибуту, указав, какие функции будут вызываться при его
> чтении, записи и удалении. Эти функции называются **getter**, **setter** и
> **deleter**. Для пользователя класса принцип работы с атрибутом никак не
> меняется. Если задан только getter, атрибут доступен **только для чтения**.

---

## 1. Мотивация

Пусть у товара есть цена, которая не может быть отрицательной. Если хранить ее
в публичном атрибуте, любой код может присвоить ему недопустимое значение:

```python
class Product:
    def __init__(self, price: int) -> None:
        self.price = price


product = Product(100)
product.price = -50   # ошибки нет
print(product.price)  # -50
```

Можно сделать атрибут служебным и добавить методы для чтения и записи:

```python
class Product:
    def __init__(self, price: int) -> None:
        self._price = 0
        self.set_price(price)

    def get_price(self) -> int:
        return self._price

    def set_price(self, new_price: int) -> None:
        if new_price < 0:
            raise ValueError("price must not be negative")
        self._price = new_price


product = Product(100)
product.set_price(product.get_price() + 50)
print(product.get_price())  # 150
```

Проверка появилась, но у решения есть недостатки:

- вместо `product.price += 50` приходится писать вложенные вызовы методов;
- если класс уже использовался с публичным атрибутом `price`, замена атрибута
  на методы изменяет публичный интерфейс и ломает весь код, в котором есть
  обращения `product.price`.

Нужен атрибут, синтаксис работы с которым остается обычным, но при чтении и
записи автоматически вызываются заданные в классе функции. Например, функция
записи может проверить новое значение и отклонить недопустимое. Для этого
предназначен `property`.

---

## 2. Что такое property

`property` — встроенный класс. Его экземпляр создается в теле класса и
сохраняется в атрибуте класса. Атрибут, определенный через `property`,
называется **свойством**. Конструктор `property` имеет следующие параметры:

```
property(fget=None, fset=None, fdel=None, doc=None)
```

| Параметр | Что это | Когда вызывается |
|---|---|---|
| `fget` | функция чтения (**getter**) | `instance.name` |
| `fset` | функция записи (**setter**) | `instance.name = value` |
| `fdel` | функция удаления (**deleter**) | `del instance.name` |
| `doc` | строка документации свойства | не вызывается |

Все параметры необязательны.

---

## 3. Прямой вызов property

В следующем примере сначала определяются функции чтения и записи, а затем они
передаются в `property`. Сами функции объявлены служебными, поскольку
пользователь должен обращаться только к свойству `price`:

```python
class Product:
    def __init__(self, price: int) -> None:
        self.price = price  # вызывается _set_price

    def _get_price(self) -> int:
        print("Getter is called")
        return self._price

    def _set_price(self, new_price: int) -> None:
        print("Setter is called")

        if new_price < 0:
            raise ValueError("price must not be negative")
        self._price = new_price

    price = property(fget=_get_price, fset=_set_price)


product = Product(100)  # Setter is called
print(product.price)
# Getter is called
# 100
product.price = 150     # Setter is called
print(product.price)
# Getter is called
# 150
product.price = -50
# Setter is called
# ValueError: price must not be negative
```

Обратите внимание:

- при чтении `product.price` скобки не нужны: getter вызывается автоматически;
- при присваивании `product.price = 150` вызывается setter, а присваиваемое
  значение передается через параметр `new_price`;
- значение хранится в служебном атрибуте `_price`, а `price` только управляет
  доступом к нему;
- присваивание `self.price = price` в `__init__` тоже вызывает setter, поэтому
  проверка выполняется уже при создании экземпляра, и дублировать ее не нужно.

### Где находится свойство

Свойство является атрибутом **класса**, а данные хранятся в **экземпляре**:

```python
print(Product.price)     # <property object at 0x...>
print(product.__dict__)  # {'_price': 150}
```

Атрибута `price` в пространстве имен экземпляра нет. Мы знаем, что атрибут
ищется сначала в экземпляре, а затем в классе. Для свойств действует
исключение: если в классе есть `property` с запрошенным именем, он имеет
**приоритет** над пространством имен экземпляра. Именно поэтому присваивание
`product.price = 150` не создает атрибут экземпляра, а вызывает setter.

---

## 4. property в качестве декоратора

При прямом вызове в пространстве имен класса остаются лишние функции
`_get_price` и `_set_price`, а определение свойства находится отдельно от них.
`property` принимает функцию чтения первым аргументом, поэтому его можно
применять как [декоратор](../../lesson08/conspects/05-decorators-introduction.md).
Это наиболее распространенный способ создания свойств.

### Определение getter

```python
class Product:
    def __init__(self, price: int) -> None:
        self._price = price

    @property
    def price(self) -> int:
        return self._price


product = Product(100)
print(product.price)  # 100
```

Запись с декоратором эквивалентна присваиванию `price = property(price)`: имя
`price` в классе ссылается уже не на функцию, а на объект `property`, у
которого эта функция сохранена в качестве getter. Имя функции становится именем
свойства, поэтому ее называют по имени атрибута, без префикса `get_`.

### Определение setter

У объекта `property` есть метод `setter`, который также используется в
качестве декоратора:

```python
class Product:
    def __init__(self, price: int) -> None:
        self.price = price

    @property
    def price(self) -> int:
        return self._price

    @price.setter
    def price(self, new_price: int) -> None:
        if new_price < 0:
            raise ValueError("price must not be negative")
        self._price = new_price


product = Product(100)
product.price += 50   # вызывается getter, затем setter
print(product.price)  # 150
product.price = -50   # ValueError: price must not be negative
```

Декоратор `@price.setter` создает **новый** объект `property`, который содержит
прежний getter и переданную функцию в качестве setter. Этот объект связывается
с именем декорируемой функции. Поэтому функция записи **должна называться так
же, как свойство**. Если назвать ее иначе, свойство с getter и setter окажется
под другим именем, а исходное останется без setter:

```python
class Product:
    def __init__(self, price: int) -> None:
        self._price = price

    @property
    def price(self) -> int:
        return self._price

    @price.setter
    def new_price(self, value: int) -> None:  # неверное имя
        self._price = value


product = Product(100)
product.new_price = 150  # запись доступна только под этим именем
print(product.price)     # 150
product.price = 200
# AttributeError: property 'price' of 'Product' object has no setter
```

Аналогично с помощью декоратора `@price.deleter` задается функция удаления. На
практике она требуется редко.

---

## 5. Свойства только для чтения

Если определен только getter, присваивание свойству невозможно:

```python
class BankAccount:
    def __init__(self, balance: int) -> None:
        self._balance = balance

    @property
    def balance(self) -> int:
        return self._balance

    def deposit(self, amount: int) -> None:
        self._balance += amount


account = BankAccount(100)
print(account.balance)  # 100
account.balance = 1000
# AttributeError: property 'balance' of 'BankAccount' object has no setter
```

Удалить такое свойство тоже нельзя, поскольку не задан deleter:

```python
del account.balance
# AttributeError: property 'balance' of 'BankAccount' object has no deleter
```

Значение при этом может изменяться методами класса:

```python
account.deposit(50)
print(account.balance)  # 150
```

Таким образом, свойство только для чтения разрешает получать состояние, но
оставляет его изменение под контролем класса.

### Границы ограничения

Ограничение распространяется только на присваивание самому свойству. Служебный
атрибут `_balance` по-прежнему доступен: от обращения к нему пользователя
удерживает лишь соглашение об именовании.

Кроме того, если getter возвращает **изменяемый объект**, вызывающая сторона
может изменить его с помощью модифицирующих методов:

```python
class Team:
    def __init__(self) -> None:
        self._members = []

    @property
    def members(self) -> list[str]:
        return self._members


team = Team()
team.members.append("Alice")  # присваивания свойству не происходит
print(team.members)           # ['Alice']
```

Чтобы защитить внутреннее состояние, getter может возвращать копию или
неизменяемый объект:

```python
class Team:
    def __init__(self) -> None:
        self._members = []

    @property
    def members(self) -> tuple[str, ...]:
        return tuple(self._members)

    def add_member(self, name: str) -> None:
        self._members.append(name)


team = Team()
team.add_member("Alice")
print(team.members)  # ('Alice',)
team.members.append("Bob")
# AttributeError: 'tuple' object has no attribute 'append'
```

---

## 6. Вычисляемые свойства

Свойство не обязано соответствовать служебному атрибуту. Его значение может
вычисляться при каждом обращении:

```python
class Rectangle:
    def __init__(self, width: int, height: int) -> None:
        self.width = width
        self.height = height

    @property
    def area(self) -> int:
        return self.width * self.height


rectangle = Rectangle(3, 4)
print(rectangle.area)  # 12
rectangle.width = 10
print(rectangle.area)  # 40 -- значение всегда соответствует сторонам
```

Если хранить площадь в обычном атрибуте, ее пришлось бы пересчитывать при
каждом изменении сторон.

---

## 7. Рекомендации

### Не обращайтесь к свойству из его же setter

Внутри getter и setter нужно работать со служебным атрибутом. Обращение к
самому свойству приводит к повторному вызову той же функции и бесконечной
рекурсии:

```python
class Product:
    @property
    def price(self) -> int:
        return self._price

    @price.setter
    def price(self, new_price: int) -> None:
        self.price = new_price  # снова вызывает этот же setter


product = Product()
product.price = 100  # RecursionError: maximum recursion depth exceeded
```

### Не создавайте свойства без необходимости

Если при чтении и записи не нужны ни проверки, ни вычисления, ни ограничения,
используйте обычный публичный атрибут. Заменить его свойством можно позже:
синтаксис обращения не изменится.

### Свойство должно вести себя как атрибут

Пользователь ожидает, что обращение к атрибуту выполняется быстро и не имеет
неожиданных побочных эффектов. Длительные вычисления, сетевые запросы и
действия, изменяющие состояние других объектов, оформляйте обычными методами:
скобки вызова показывают, что выполняется операция.

---

## Вывод

- **`property`** — встроенный класс; его экземпляр сохраняется в атрибуте
  класса и вызывает функции при чтении, записи и удалении атрибута.
- Свойство можно создать прямым вызовом `property(fget=..., fset=...)` или
  декораторами `@property` и `@name.setter`. Вариант с декораторами
  используется чаще.
- Функции, декорированные `@property` и `@name.setter`, должны иметь **одно и
  то же имя** — имя свойства.
- Свойство с одним getter доступно **только для чтения**: присваивание
  вызывает `AttributeError`. Изменяемый объект, который возвращает getter,
  этим не защищен.
- Внутри getter и setter используется служебный атрибут, а не само свойство.
- Свойства позволяют добавить проверки и вычисления, **не изменяя публичный
  интерфейс** класса.

---

## Источники

- [Python documentation. Built-in Functions — property](https://docs.python.org/3/library/functions.html#property);
- [Python documentation. Descriptor Guide — Properties](https://docs.python.org/3/howto/descriptor.html#properties).
