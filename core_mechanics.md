- [Перегрузка операторов](#перегрузка-операторов)
  - [`operator<<`](#operator)
    - [Полиморфизм](#полиморфизм)
- [copy elision](#copy-elision)
  - [C++17 и позже](#c17-и-позже)
  - [Когда RVO ломается](#когда-rvo-ломается)
- [SFINAE (Substitution Failure Is Not An Error)](#sfinae-substitution-failure-is-not-an-error)
  - [trailing return type](#trailing-return-type)
  - [`decltype(...)`](#decltype)
  - [Что означает `std::cout << value, void()`?](#что-означает-stdcout--value-void)

# Перегрузка операторов
## `operator<<`
```
friend std::ostream& operator<<(std::ostream& os, const Base& b)
```
Возвращается `std::ostream&` потому что нужен чейнинг:
```
std::cout << b1 << b2 << 42;
```

Каждый `operator<<` возвращает тот же поток, чтобы следующий вызов работал с ним же.
Если вернуть `void` — цепочка невозможна.
Если вернуть поток по значению — будет копирование, а `std::ostream` вообще некопируемый.
Поэтому строго **возвращаем ссылку на поток**.

Первый параметр — `std::ostream&`, так как оператор вызывается как:
```
std::cout << b;
```
Это синтаксический сахар для:
```
operator<<(std::cout, b);
```
Левый операнд — поток.
Если бы оператор был методом `Base`, он бы выглядел так:
```
b.operator<<(std::cout);
```
А это не то поведение, которое нам нужно. Оператор `<<` для потоков всегда принимает поток первым параметром.

Почему это не может быть обычным методом? Если сделать так:
```
struct Base {
    std::ostream& operator<<(std::ostream& os) const;
};
```
Тогда писать придётся:
```
b << std::cout;   // а не std::cout << b
```
Это переворачивает семантику оператора. Поток должен быть слева — это устоявшаяся идиома C++. Поэтому оператор обязан быть свободной функцией, а не методом класса.

### Полиморфизм
При:
```
Base* p = new Derived(...);
std::cout << *p;
```
Вызовется версия для `Base`, а не для `Derived`.

Правильный паттерн:
```
class Base {
public:
    virtual void print(std::ostream& os) const {
        os << s;
    }

    friend std::ostream& operator<<(std::ostream& os, const Base& b) {
        b.print(os);
        return os;
    }

    virtual ~Base() = default;

private:
    std::string s;
};
```


# copy elision
Это оптимизация, при которой компилятор избегает создания временных объектов и пропускает вызовы копирующих/перемещающих конструкторов, напрямую создавая объект в конечном месте.

```
T foo() {
    return T();   // без copy elision: создание временного, потом копия
}
```
С elision объект `T()` создаётся сразу в месте, куда будет присвоен результат `foo()`, поэтому копирования нет.

Используется при:

**Return Value Optimization (RVO).**
Оптимизация возврата локального объекта:
```
T foo() {
    T x;
    return x;   // RVO
}
```

**Named Return Value Optimization (NRVO).**
Когда возвращается именованная переменная.

**Элиминация при инициализации:**
```
T a = T();      // временный не создаётся
```

## C++17 и позже

Стандарт **обязывает** компилятор выполнять copy elision в ряде случаев (например, при возврате временного или локального объекта), даже если копирующие/перемещающие конструкторы удалены.

Например, это допустимо:
```
struct T {
    T(const T&) = delete;
    T(T&&) = delete;
};

T foo() {
    return T();    // работает: обязательный elision
}
```

## Когда RVO ломается
- Есть несколько return-путей, возвращающих разные локальные переменные
  ```
  T f(bool b) {
    T a;
    T c;
    if (b) return a;
    else  return c; // NRVO не применим: какой local выбрать?
  }
  ```
- Возврат из вложенных блоков или через `goto`/`setjmp`/`longjmp`.
- Возвращается локал с разным типом, требующий преобразования в возвращаемый тип (`Derived` -> `Base`), тогда может потребоваться явная конверсия/копирование.

# SFINAE (Substitution Failure Is Not An Error)
Правило, согласно которому ошибка при подстановке шаблонных параметров не считается ошибкой компиляции, если существует другая подходящая перегрузка.

Иначе: если при инстанцировании шаблона выражение внутри сигнатуры оказывается некорректным, компилятор просто исключает этот вариант из перегрузки и продолжает поиск.

**Классический пример с `std::enable_if`**
```
#include <type_traits>
#include <iostream>

template<typename T>
typename std::enable_if<std::is_integral<T>::value>::type
foo(T value)
{
    std::cout << "Integral: " << value << "\n";
}

template<typename T>
typename std::enable_if<!std::is_integral<T>::value>::type
foo(T value)
{
    std::cout << "Not integral\n";
}
```

Как это работает:
```
std::enable_if<condition>::type
```
если `condition == true` → существует `type`

если `false` → `type` не существует

Если `type` не существует — подстановка проваливается, но это не ошибка, просто эта перегрузка удаляется.

**Базовая идея на примере перегрузки**
```
#include <iostream>

template<typename T>
auto print(T value) -> decltype(std::cout << value, void())
{
    std::cout << value << "\n";
}

void print(...)
{
    std::cout << "Not printable\n";
}
```

Для типа `T` компилятор проверяет `std::cout << value`. Если выражение корректно → шаблон участвует в перегрузке. Если нет → шаблон просто исключается. Тогда вызывается `print(...)`:
```
print(42);        // печатает 42
print("hello");   // печатает hello
```

Если тип не поддерживает `operator<<`, шаблон будет исключён.

## trailing return type
```
auto ... -> type?
```

Обычная запись:
```
int foo();
```

Аналог с trailing return:
```
auto foo() -> int;
```

Тип возврата зависит от параметров функции. В нашем случае внутри `decltype` используется `value`, а его тип — `T`.
До C++11 так писать было нельзя.

## `decltype(...)`

Возвращает тип выражения expr, не вычисляя его.
```
int x = 5;
decltype(x) y = 10;  // y имеет тип int
```

**Важно:** выражение внутри `decltype` проверяется на корректность, но не выполняется.

## Что означает `std::cout << value, void()`?

Здесь используется оператор запятая. Оператор запятая работает так: `(expr1, expr2)` вычисляется `expr1`,  затем `expr2`, результатом всего выражения становится `expr2`.

Пример:
```
int x = (1, 2);  // x = 2
```

В случае `decltype(std::cout << value, void())` проверяется корректность выражения `std::cout << value`. Если корректно — вернуть тип `void`
(потому что результатом запятой является `void()`). Если `operator<<` существует → тип функции = `void`. Если `operator<<` не существует → подстановка ломается

И вот тут срабатывает SFINAE: ошибка подстановки → шаблон просто исключается из перегрузки.


**Более интересный пример — проверка наличия метода**
```
#include <type_traits>
#include <iostream>

template<typename T>
auto has_size_impl(int) -> decltype(std::declval<T>().size(), std::true_type());

template<typename>
std::false_type has_size_impl(...);

template<typename T>
using has_size = decltype(has_size_impl<T>(0));
```

Теперь:
```
#include <vector>

std::cout << has_size<std::vector<int>>::value; // 1
std::cout << has_size<int>::value;              // 0
```

Первая перегрузка проверяет существование `T::size()`. Если выражение корректно → возвращается `std::true_type`. Если нет → подстановка ломается → выбирается `...` версия.