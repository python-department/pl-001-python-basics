# Методы экземпляров

> **Главная мысль.** **Метод экземпляра (instance method)** — функция в теле
> класса, которая вызывается через экземпляр: `instance.method(...)`.
> Python автоматически передает этот экземпляр в качестве **первого параметра**,
> который по соглашению называется **`self`**. Через `self` метод может читать и
> изменять состояние конкретного экземпляра. Для задания начального состояния
> используется специальный метод **`__init__`**: интерпретатор вызывает его
> автоматически во время создания экземпляра, передавая ему аргументы,
> указанные при этом.

---

## 1. Методы экземпляров и параметр self

Поведение, общее для всех экземпляров класса, оформляется **методами
экземпляров** — функциями, определенными в теле класса. Первым параметром
такого метода является `self`. В примере метод `describe` обращается через
`self` к атрибутам экземпляра, которые пока задаются присваиваниями вне
класса, как в предыдущем конспекте:

```python
class BankAccount:
    def describe(self) -> str:
        return f"{self.owner}: {self.balance}"


account = BankAccount()
account.owner = "Alice"
account.balance = 100
print(account.describe())  # Alice: 100
```

Функция `describe` определена один раз, в классе, но доступна каждому
экземпляру. При вызове `account.describe()`:

1. интерпретатор ищет атрибут `describe` у экземпляра `account`; в его
   пространстве имен атрибута нет, поэтому поиск продолжается в классе, где
   функция `describe` найдена;
2. поскольку найденная в классе функция вызывается через экземпляр, создается
   **связанный метод (bound method)**, который хранит функцию и экземпляр;
3. при вызове экземпляр `account` автоматически подставляется в параметр
   `self`.

Поэтому следующие два вызова эквивалентны:

```python
print(account.describe())             # Alice: 100
print(BankAccount.describe(account))  # Alice: 100 -- экземпляр передан явно
```

Вызов через экземпляр — это сокращенная запись: экземпляр указывается перед
точкой, и Python подставляет его в первый параметр сам.

Параметр `self` — это обычное имя, а не ключевое слово. Использовать
другое имя технически допустимо, но в сообществе Python принято соглашение:
первый параметр метода экземпляра называется `self`. Аннотацию типа
для него не указывают.

Каждый вызов метода получает тот экземпляр, через который он был вызван,
поэтому один и тот же метод работает с разными данными, соответствующими
состояниям тех экземпляров, через которые он вызван:

```python
first_account = BankAccount()
first_account.owner = "Alice"
first_account.balance = 100
second_account = BankAccount()
second_account.owner = "Bob"
second_account.balance = 250
print(first_account.describe())   # Alice: 100
print(second_account.describe())  # Bob: 250
```

### Типичная ошибка: нет параметра для экземпляра

Метод экземпляра должен иметь первый параметр, через который неявно
передается экземпляр (по соглашению `self`). Если в определении метода
параметров нет, при вызове через экземпляр возникнет `TypeError`: экземпляр
передается автоматически, а параметра для его приема в функции нет:

```python
class BankAccount:
    def describe() -> str:
        return "Bank account"


account = BankAccount()
print(account.describe())
# TypeError: BankAccount.describe() takes 0 positional arguments but 1 was given
```

В сообщении указано, что функция принимает 0 аргументов, но получила 1: этим
аргументом оказался сам экземпляр.

Если функции в теле класса не нужен экземпляр, ее можно оформить как
статический метод с помощью декоратора `@staticmethod` (или как метод класса
с помощью `@classmethod`). Такой метод можно вызывать через экземпляр, и
экземпляр в него автоматически не передается:

```python
class BankAccount:
    @staticmethod
    def describe() -> str:
        return "Bank account"


account = BankAccount()
print(account.describe())      # Bank account
print(BankAccount.describe())  # Bank account
```

---

## 2. Специальный метод __init__

Чтобы не задавать атрибуты вручную после каждого создания экземпляра, в классе
определяют **метод инициализации** с именем `__init__`. Это **специальный
метод (special method)**: его имя начинается и заканчивается двумя
подчеркиваниями, а вызывает его не программист, а интерпретатор — автоматически
при создании экземпляра.

```python
class BankAccount:
    def __init__(self, owner: str, balance: int) -> None:
        self.owner = owner
        self.balance = balance

    def describe(self) -> str:
        return f"{self.owner}: {self.balance}"


account = BankAccount("Alice", 100)
print(account.describe())  # Alice: 100
```

Вызов `BankAccount("Alice", 100)` выполняется в такой последовательности:

1. создается новый пустой экземпляр класса `BankAccount`;
2. для него вызывается `__init__`: первым аргументом (`self`) передается
   созданный экземпляр, а остальными — аргументы, указанные при вызове класса
   (`"Alice"` и `100`);
3. вызов класса возвращает созданный экземпляр.

К моменту вызова `__init__` экземпляр уже создан: метод **инициализирует** его,
а не создает.

### Сигнатура __init__

Специальные методы (их также называют **магическими**, magic methods) имеют
фиксированную сигнатуру: интерпретатор вызывает их по определенному
соглашению и рассчитывает на определенные параметры и возвращаемое значение.
Для `__init__` соглашение состоит в том, что метод **ничего не возвращает**,
то есть возвращает `None`. Это следует из назначения метода: `__init__` только
инициализирует уже существующий экземпляр и не создает новый объект, поэтому
возвращаемое значение ему не нужно. Если из `__init__` вернуть значение,
отличное от `None`, будет возбуждено исключение `TypeError`:

```python
class BankAccount:
    def __init__(self, owner: str, balance: int) -> None:
        self.owner = owner
        self.balance = balance
        return self


account = BankAccount("Alice", 100)
# TypeError: __init__() should return None, not 'BankAccount'
```

### Атрибуты экземпляра создаются через self

Присваивание `self.owner = owner` создает атрибут `owner` в пространстве имен
экземпляра. Слева от `=` стоит атрибут, справа — параметр метода.
Это разные объекты, несмотря на одинаковые имена. Параметры, как и любые
локальные переменные, существуют только во время выполнения тела метода, а
атрибуты остаются у экземпляра:

```python
class BankAccount:
    def __init__(self, owner: str, balance: int) -> None:
        self.owner = owner  # атрибут экземпляра
        balance = balance   # присваивание локальной переменной: атрибут не создается


account = BankAccount("Alice", 100)
print(account.__dict__)  # {'owner': 'Alice'}
```

### Особенности __init__

- Метод `__init__` — обычная функция, поэтому в нем допустимы значения по
  умолчанию, `*args`, `**kwargs` и вызовы других функций.
- Если аргументов, переданных при создании экземпляра, не хватает (или их
  больше, чем параметров `__init__`), возникает `TypeError`, как при вызове обычной
  функции.
- В `__init__` можно проверять допустимость значений и возбуждать исключения:
  в этом случае экземпляр не будет создан.

Значения по умолчанию в `__init__` задаются так же, как в обычной функции.
Параметры со значениями по умолчанию можно не передавать при создании
экземпляра или передавать по имени:

```python
class BankAccount:
    def __init__(
        self,
        owner: str,
        balance: int = 0,
        currency: str = "USD",
    ) -> None:
        self.owner = owner
        self.balance = balance
        self.currency = currency


first_account = BankAccount("Alice")
second_account = BankAccount("Bob", 100)
third_account = BankAccount("Carol", 100, "EUR")
fourth_account = BankAccount("Dave", currency="EUR")
print(first_account.balance, first_account.currency)    # 0 USD
print(second_account.balance, second_account.currency)  # 100 USD
print(third_account.balance, third_account.currency)    # 100 EUR
print(fourth_account.balance, fourth_account.currency)  # 0 EUR
```

Проверка значений и ошибки при неверном вызове:

```python
class BankAccount:
    def __init__(self, owner: str, balance: int = 0) -> None:
        if balance < 0:
            raise ValueError("balance must not be negative")
        self.owner = owner
        self.balance = balance


first_account = BankAccount("Alice", 100)
second_account = BankAccount("Bob")  # balance принимает значение по умолчанию
print(first_account.balance)         # 100
print(second_account.balance)        # 0
BankAccount("Carol", -5)             # ValueError: balance must not be negative
BankAccount()                        # TypeError: ... missing 1 required positional argument: 'owner'
```

---

## 3. Вызов методов экземпляра

Метод экземпляра вызывается через экземпляр: `instance.method(arguments)`.
Остальные аргументы передаются после автоматически подставленного `self`.
Метод может читать и менять состояние экземпляра через `self`, а также
возвращать значение:

```python
class BankAccount:
    def __init__(self, owner: str, balance: int = 0) -> None:
        self.owner = owner
        self.balance = balance

    def deposit(self, amount: int) -> None:
        self.balance += amount

    def withdraw(self, amount: int) -> None:
        if amount > self.balance:
            raise ValueError("insufficient funds")
        self.balance -= amount

    def describe(self) -> str:
        return f"{self.owner}: {self.balance}"


first_account = BankAccount("Alice", 100)
second_account = BankAccount("Bob")
first_account.deposit(50)
first_account.withdraw(30)
second_account.deposit(10)
print(first_account.describe())   # Alice: 120
print(second_account.describe())  # Bob: 10
```

Методы одного экземпляра могут вызывать друг друга. Для этого вызов также
выполняется через `self`:

```python
class BankAccount:
    def __init__(self, owner: str, balance: int = 0) -> None:
        self.owner = owner
        self.balance = balance

    def describe(self) -> str:
        return f"{self.owner}: {self.balance}"

    def print_description(self) -> None:
        print(self.describe())  # вызов другого метода того же экземпляра


BankAccount("Alice", 100).print_description()  # Alice: 100
```

Операции над одним экземпляром не влияют на остальные: методы изменяют только
атрибуты того экземпляра, который передан в `self`. В примере выше
пополнение `first_account` не изменило баланс `second_account`.

---

## 4. Статические методы и методы класса при вызове через экземпляр

Статические методы и методы класса можно вызывать не только через объект
класса, но и через экземпляр. При этом:

- статический метод по-прежнему **не получает** неявных аргументов;
- метод класса получает в `cls` **класс экземпляра**, а не сам экземпляр.

Именно здесь проявляется разница между статическим методом и обычной функцией
в теле класса: при вызове через экземпляр обычная функция получит экземпляр
первым аргументом, как метод экземпляра.

```python
class Converter:
    def plain(value: int) -> int:
        return value * 2

    @staticmethod
    def static(value: int) -> int:
        return value * 2

    @classmethod
    def get_class_name(cls) -> str:
        return cls.__name__


converter = Converter()
print(Converter.plain(5))          # 10 -- через класс работает
print(converter.static(5))         # 10 -- через экземпляр работает
print(converter.get_class_name())  # Converter -- в cls передан класс экземпляра
print(converter.plain(5))
# TypeError: Converter.plain() takes 1 positional argument but 2 were given
```

Это объясняет, зачем нужен декоратор `@staticmethod`: он гарантирует, что
функция будет работать одинаково при вызове через класс и через экземпляр.

Итоговое сравнение функций, определенных в теле класса:

| Вид | Декоратор | Первый параметр | Что подставляется при вызове через класс | Что подставляется при вызове через экземпляр |
|---|---|---|---|---|
| Метод экземпляра | нет | `self` | ничего (экземпляр нужно передать вручную) | экземпляр |
| Статический метод | `@staticmethod` | любой | ничего | ничего |
| Метод класса | `@classmethod` | `cls` | класс | класс экземпляра |

---

## Вывод

- **Метод экземпляра** — функция в теле класса, у которой первый параметр
  (по соглашению `self`) получает экземпляр, через который вызван метод.
  Запись `instance.method(arguments)` эквивалентна
  `Class.method(instance, arguments)`.
- Функция метода определяется один раз в теле класса и доступна всем его
  экземплярам. При вызове через экземпляр Python создает связанный метод
  (bound method), который хранит эту функцию и экземпляр.
- Метод экземпляра должен иметь первый параметр для неявно передаваемого
  экземпляра (по соглашению `self`). Если такого параметра нет, вызов через
  экземпляр завершится `TypeError`.
- **`__init__`** — специальный метод инициализации, который интерпретатор
  вызывает автоматически во время создания экземпляра. Он получает `self` и
  аргументы, переданные при создании экземпляра, создает атрибуты через
  `self.name = value` и должен возвращать `None`.
- Атрибуты экземпляра создаются присваиванием через `self`; обычные
  переменные и параметры метода существуют только во время его выполнения.
- Внутри метода экземпляра через `self` можно обращаться к состоянию этого
  экземпляра (`self.name`) и к его методам (`self.method()`).
- При вызове через экземпляр статический метод не получает неявных аргументов,
  а метод класса получает в `cls` класс экземпляра.

---

## Источники

- [Python documentation. The Python Tutorial — Method Objects](https://docs.python.org/3/tutorial/classes.html#method-objects);
- [Python documentation. The Python Tutorial — Class and Instance Variables](https://docs.python.org/3/tutorial/classes.html#class-and-instance-variables);
- [Python documentation. The Python Language Reference — object.\_\_init\_\_](https://docs.python.org/3/reference/datamodel.html#object.__init__);
- [Python documentation. The Python Language Reference — Instance methods](https://docs.python.org/3/reference/datamodel.html#instance-methods).
