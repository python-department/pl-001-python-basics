# Методы класса: classmethod

> **Главная мысль.** **Метод класса (class method)** — функция в теле класса,
> к которой применен декоратор `@classmethod`. При вызове через объект класса
> Python автоматически передает этот объект в качестве значения **первого
> параметра**, который по соглашению называется **`cls`**. Через `cls` метод
> получает доступ к атрибутам класса и другим его методам, не указывая имя
> класса явно. Методы класса предназначены для операций над состоянием,
> которое принадлежит самому классу.

---

## 1. Мотивация

Расширим класс `Temperature` из предыдущего конспекта: коэффициент перевода и
смещение шкалы Фаренгейта сохраним в атрибутах класса, а методы преобразования
температуры будут использовать эти атрибуты:

```python
class Temperature:
    ratio = 5 / 9  # отношение градуса Фаренгейта к градусу Цельсия
    offset = 32    # температура замерзания воды по шкале Фаренгейта

    @staticmethod
    def celsius_to_fahrenheit(celsius: float) -> float:
        return celsius / Temperature.ratio + Temperature.offset

    @staticmethod
    def fahrenheit_to_celsius(fahrenheit: float) -> float:
        return (fahrenheit - Temperature.offset) * Temperature.ratio


print(Temperature.celsius_to_fahrenheit(100))  # 212.0
print(Temperature.fahrenheit_to_celsius(212))  # 100.0
```

Все данные, необходимые для преобразования температуры, находятся в одном
пространстве имен — в классе. Методам не нужны данные экземпляра, им требуются
только атрибуты класса. Однако статический метод не получает ссылку на класс,
поэтому вынужден обращаться к атрибутам через имя `Temperature`. При
переименовании класса придется изменить все методы. Метод класса решает эту
проблему: объект класса передается в него автоматически.

---

## 2. Декоратор classmethod и параметр cls

Чтобы функция стала методом класса (class method), к ней применяют декоратор
`@classmethod`. Первым параметром такого метода всегда является объект класса,
через который метод вызван. Python подставляет этот аргумент самостоятельно,
вызывающий код его не передает:

```python
class Temperature:
    ratio = 5 / 9
    offset = 32

    @classmethod
    def celsius_to_fahrenheit(cls, celsius: float) -> float:
        return celsius / cls.ratio + cls.offset


print(Temperature.celsius_to_fahrenheit(100))  # 212.0
```

Вызов `Temperature.celsius_to_fahrenheit(100)` выполняется так:

1. по имени `Temperature` находится объект класса, у него запрашивается
   атрибут `celsius_to_fahrenheit`;
2. поскольку атрибут является методом класса, возвращается метод, **связанный
   с объектом класса**;
3. при вызове объект `Temperature` автоматически подставляется в параметр
   `cls`, а значение `100` передается в параметр `celsius`.

Убедимся, что `cls` — это тот же самый объект, что и `Temperature`:

```python
class Temperature:
    @classmethod
    def show_cls(cls) -> None:
        print(cls)
        print(cls is Temperature)


Temperature.show_cls()
# <class '__main__.Temperature'>
# True
```

Имя `cls` — это обычный параметр, а не ключевое слово. Использовать другое
имя технически допустимо, но в Python-сообществе принято соглашение: первый
параметр метода класса называется `cls` (сокращение от `class`).

Аннотации типов для `cls` обычно не указывают: тип параметра очевиден из
контекста, а для остальных параметров и возвращаемого значения аннотации
записываются как обычно.

---

## 3. Доступ к атрибутам и методам класса через cls

Внутри метода класса через `cls` выполняются те же операции с атрибутами,
которые в предыдущих конспектах выполнялись через имя класса: чтение,
присваивание (запись и добавление), удаление.

```python
class Temperature:
    ratio = 5 / 9
    offset = 32
    precision = 2

    @classmethod
    def celsius_to_fahrenheit(cls, celsius: float) -> float:
        fahrenheit = celsius / cls.ratio + cls.offset  # чтение атрибутов
        return round(fahrenheit, cls.precision)

    @classmethod
    def set_precision(cls, new_precision: int) -> None:
        cls.precision = new_precision  # присваивание атрибуту класса


print(Temperature.celsius_to_fahrenheit(36.6))  # 97.88
Temperature.set_precision(0)
print(Temperature.celsius_to_fahrenheit(36.6))  # 98.0
```

Присваивание `cls.precision = new_precision` меняет атрибут именно в объекте
класса, то есть действует так же, как `Temperature.precision = new_precision`
вне метода. Все методы работают с общим состоянием класса, а доступ к нему
выполняется из методов самого класса.

Через `cls` можно также вызывать другие методы класса. В следующем примере
метод `describe` использует метод `celsius_to_fahrenheit`:

```python
class Temperature:
    ratio = 5 / 9
    offset = 32

    @classmethod
    def celsius_to_fahrenheit(cls, celsius: float) -> float:
        return celsius / cls.ratio + cls.offset

    @classmethod
    def describe(cls, celsius: float) -> str:
        fahrenheit = cls.celsius_to_fahrenheit(celsius)  # другой метод класса
        return f"{celsius} C = {fahrenheit} F"


print(Temperature.describe(100))  # 100 C = 212.0 F
```

Выражение `cls.celsius_to_fahrenheit(celsius)` в `describe` эквивалентно
`Temperature.celsius_to_fahrenheit(celsius)`, так как `cls` ссылается на объект
класса. Статический метод тоже можно вызвать через `cls`.

В отличие от статического метода, метод класса не зависит от имени класса. При
переименовании `Temperature` код методов менять не нужно.

---

## 4. Когда использовать статический метод и метод класса

- **Метод класса** — когда функции нужно читать или менять атрибуты класса
  либо вызывать другие его методы. Например, перевод температуры с
  коэффициентами, которые хранятся в атрибутах класса `Temperature`, а также
  настройки или реестр объектов.
- **Статический метод** — когда состояние класса для вычисления не требуется,
  но поведение тесно связано с объектом класса и определяется в нем. В этом
  случае можно обойтись `@staticmethod`. Например, перевод градусов Цельсия в
  градусы Фаренгейта по формуле, в которой коэффициенты заданы явно, как в
  предыдущем конспекте.

Если данные класса функции не нужны, делать ее методом класса только ради
параметра `cls` не требуется: статический метод точнее передает назначение
функции.

---

## Вывод

- **Метод класса** — функция в теле класса с декоратором `@classmethod`.
  Первым параметром, по соглашению `cls`, метод получает объект класса,
  через который вызван.
- Аргумент `cls` подставляется автоматически, вызывающий код его не передает:
  `Temperature.celsius_to_fahrenheit(100)` вызывает функцию с `cls`, равным
  `Temperature`.
- Через `cls` читаются и изменяются атрибуты класса (`cls.ratio`,
  `cls.precision = ...`), а также вызываются другие методы класса
  (`cls.other_method()`). Имя класса в коде метода указывать не нужно.
- Статический метод применяют для функций, которым не нужны данные класса.
  Метод класса — когда нужен доступ к атрибутам и методам класса.

---

## Источники

- [Python documentation. Built-in Functions — classmethod](https://docs.python.org/3/library/functions.html#classmethod);
- [Python documentation. The Python Tutorial — Class Objects](https://docs.python.org/3/tutorial/classes.html#class-objects).
