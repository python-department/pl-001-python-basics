# Объект super()

> **Главная мысль.** Вызов **`super()`** возвращает объект для поиска
> атрибутов по MRO. Для поиска `super()` использует **MRO того объекта, через
> вызов которого был инициирован поиск**, и начинает с класса, **следующего за
> текущим в порядке MRO**. Чаще всего `super()` используют, чтобы внутри
> переопределенного метода вызвать реализацию этого же метода из родительского
> класса и **дополнить** ее, а не заменить полностью.

---

## 1. Мотивация

Переопределенный метод полностью заменяет родительский: при поиске находится
метод дочернего класса, и родительский не выполняется. Часто же требуется
сохранить поведение родителя и добавить к нему новое.

```python
class Employee:
    def __init__(self, name: str) -> None:
        self.name = name

    def describe(self) -> str:
        return f"Employee {self.name}"


class Manager(Employee):
    def describe(self) -> str:
        return f"Employee {self.name}, role: manager"  # дублирование
```

Метод `Manager.describe` повторяет код родителя. Если изменить формат строки в
`Employee.describe`, метод `Manager.describe` по-прежнему будет возвращать
строку в старом формате.

Родительскую функцию можно вызвать через объект класса, передав экземпляр
явно:

```python
class Manager(Employee):
    def describe(self) -> str:
        parent_description = Employee.describe(self)
        return f"{parent_description}, role: manager"


print(Manager("Bob").describe())  # Employee Bob, role: manager
```

Дублирования больше нет, но имя родителя записано внутри метода. При смене
родительского класса придется исправлять все такие обращения. Кроме того, при
множественном наследовании явный вызов нарушает порядок MRO, что показано в
разделе 3.

---

## 2. Использование super()

Для решения обозначенных проблем используют объект `super`. Он позволяет
обратиться к реализации родителя, не указывая имя родительского класса, и
выполняет поиск в соответствии с MRO.

`super` — встроенный класс. Внутри методов его вызывают без аргументов:

```python
class Manager(Employee):
    def describe(self) -> str:
        parent_description = super().describe()
        return f"{parent_description}, role: manager"


print(Manager("Bob").describe())  # Employee Bob, role: manager
```

`super()` возвращает **не родительский класс** и не его экземпляр, а
объект-посредник типа `super`. При обращении к его атрибуту выполняется
поиск в родительских классах. Если найден метод экземпляра, он связывается с
текущим экземпляром, поэтому явно передавать `self` не нужно: в вызове
`super().describe()` экземпляр передается автоматически, а в вызове
`Employee.describe(self)` из предыдущего раздела он передавался явно.

Запись `super()` является сокращением для `super(CurrentClass, self)`:

- первый аргумент — класс, **после которого** в MRO нужно начинать поиск;
- второй аргумент — объект, **по MRO класса которого** выполняется поиск и с
  которым связываются найденные методы.

Форма без аргументов работает только внутри функций, определенных в теле
класса: интерпретатор сам подставляет текущий класс и первый параметр метода.

Вызов через `super()` можно разместить в любом месте тела метода, поэтому
собственный код может выполняться до вызова родительского метода, после него
или с обеих сторон.

### Поиск выполняется по всем родителям

`super()` не ограничивается непосредственным родителем. Если в нем атрибута
нет, поиск продолжается дальше по MRO:

```python
class Employee:
    def get_role(self) -> str:
        return "employee"


class Manager(Employee):
    pass


class Director(Manager):
    def get_role(self) -> str:
        parent_role = super().get_role()  # найден в Employee
        return f"director, {parent_role}"


print(Director().get_role())  # director, employee
```

---

## 3. super() при множественном наследовании

`super()` ищет атрибут **в классах, которые следуют за текущим в MRO объекта,
с которым работает метод**. При одиночном наследовании следующим всегда
является родитель текущего класса, а при множественном это не обязательно.

```python
class Device:
    def start(self) -> None:
        print("Device started")


class Printer(Device):
    def start(self) -> None:
        print("Printer started")
        super().start()


class Scanner(Device):
    def start(self) -> None:
        print("Scanner started")
        super().start()


class Copier(Printer, Scanner):
    def start(self) -> None:
        print("Copier started")
        super().start()


print([parent.__name__ for parent in Copier.__mro__])
# ['Copier', 'Printer', 'Scanner', 'Device', 'object']
Copier().start()
# Copier started
# Printer started
# Scanner started
# Device started
```

Вызов `super().start()` в классе `Printer` передает управление методу класса
`Scanner`, хотя `Scanner` не является родителем `Printer`. Причина в том, что
`self` — экземпляр `Copier`, и поиск идет по MRO класса `Copier`, где после
`Printer` находится `Scanner`.

Для экземпляра самого класса `Printer` тот же вызов в том же методе приводит к
`Device`:

```python
Printer().start()
# Printer started
# Device started
```

Следовательно, результат выполнения метода, вызванного через `super()`,
зависит от того, через какой объект был выполнен вызов, а также от структуры
MRO.

### Сравнение с явными вызовами

Заменим в той же иерархии каждый вызов `super().start()` явным вызовом метода
родительского класса. В `Copier.start` будем вызывать `Printer.start`:

```python
class Device:
    def start(self) -> None:
        print("Device started")


class Printer(Device):
    def start(self) -> None:
        print("Printer started")
        Device.start(self)


class Scanner(Device):
    def start(self) -> None:
        print("Scanner started")
        Device.start(self)


class Copier(Printer, Scanner):
    def start(self) -> None:
        print("Copier started")
        Printer.start(self)


Copier().start()
# Copier started
# Printer started
# Device started
```

Метод `Scanner.start` не выполнился. Вызов `Device.start(self)` в классе
`Printer` всегда обращается к методу класса `Device`: `self` по-прежнему
является экземпляром `Copier`, но MRO этого класса при выборе метода не
учитывается. Вызов `super().start()` на том же месте выполняет поиск по MRO
класса `Copier` и поэтому приводит к `Scanner.start`.

### Условия работы цепочки вызовов

Классы, рассчитанные на совместную работу через `super()`, называют
**кооперативными**. Чтобы цепочка работала:

- каждый переопределенный метод должен вызывать `super()`. Если один из них
  этого не сделает, методы следующих в MRO классов не выполнятся;
- сигнатуры методов должны быть совместимыми, поскольку неизвестно, какой
  метод окажется следующим;
- в цепочке должен быть класс, который завершает ее и `super()` не вызывает. В
  примере это `Device`: в `object` метода `start` нет.

---

## Вывод

- **`super()`** возвращает объект, который ищет атрибуты в классах,
  следующих в MRO за текущим, и связывает найденные методы с экземпляром.
- Внутри методов используется форма без аргументов; она эквивалентна
  `super(CurrentClass, self)`.
- Вызов `super().method()` позволяет **дополнить** родительский метод своими
  действиями вместо дублирования его кода.
- При множественном наследовании следующим в MRO может быть класс, не
  являющийся родителем текущего.
- Результат выполнения метода, вызванного через `super()`, зависит от того,
  через какой объект был выполнен вызов, а также от структуры MRO.
- Явный вызов через имя класса обращается к указанному классу независимо от
  MRO объекта.

---

## Источники

- [Python documentation. Built-in Functions — super](https://docs.python.org/3/library/functions.html#super);
- [Raymond Hettinger. Python's super() considered super!](https://rhettinger.wordpress.com/2011/05/26/super-considered-super/).
