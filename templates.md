- [Реализация в `.cpp`](#реализация-в-cpp)
  - [Явная инстанциация (explicit instantiation)](#явная-инстанциация-explicit-instantiation)
  - [Полная специализация](#полная-специализация)
- [Шаблонный класс и шаблонные методы](#шаблонный-класс-и-шаблонные-методы)
  - [Шаблонный класс](#шаблонный-класс)
  - [Шаблонный класс и шаблонные методы](#шаблонный-класс-и-шаблонные-методы-1)
  - [`typename` vs `class`](#typename-vs-class)
  - [Шаблон шаблонов (template template parameter)](#шаблон-шаблонов-template-template-parameter)
- [variadic templates](#variadic-templates)
  - [Как получить первый аргумент:](#как-получить-первый-аргумент)
  - [pack expansion](#pack-expansion)
  - [Fold expression (C++17)](#fold-expression-c17)
- [type traits](#type-traits)
  - [`std::enable_if`](#stdenable_if)
  - [`std::is_integral`](#stdis_integral)
  - [type traits vs concepts](#type-traits-vs-concepts)


# Реализация в `.cpp`

## Явная инстанциация (explicit instantiation)

Если  известно, какие специализации шаблона нужны:
```
// .h

template <typename T>
class Foo {
public:
    void bar();
};
extern template class Foo<int>;  // объявление
```

```
// .cpp

template <typename T>
void Foo<T>::bar() { /* ... */ }

template class Foo<int>;  // явная инстанциация
```

## Полная специализация

Полностью специализированный шаблон — это уже не шаблон, а обычный класс.

```
// .h:

template <>
class Foo<int> {
public:
    void bar();
};
```

```
// .cpp:

void Foo<int>::bar() { /* ... */ }
```

Работает, но только для одной конкретной специализации.



# Шаблонный класс и шаблонные методы

## Шаблонный класс

```
template <size_t N, typename T>
class Base {
    void multiply(const T&);
    void add(const T&);
};
```

Тип аргумента `T` — часть типа класса. Для `Base<3, int>` и `Base<3, double>` создаются совершенно разные классы. Методы не шаблонные; тип суммы/умножения определён заранее.

При использовании `Base<3, int>` создаётся класс, в котором методы принимают `const int&`.

**Что можно вынести в .cpp:**

Вынести реализацию можно при условии:

```
template class Base<3,int>;
template class Base<3,double>;
```

## Шаблонный класс и шаблонные методы

```
template <size_t N>
class Base {
    template <typename T> void multiply(const T&);
    template <typename T> void add(const T&);
};
```

У класса всего один параметр: `N`. Методы - шаблоны сами по себе, независимы от шаблонных параметров класса. Тип аргумента выбирается при каждом вызове.

```
Base<3> b;
b.add(2) → инстанцируется add<int>()
b.add(2.5) → инстанцируется add<double>()
```

Инстанцируется при каждом вызове с новым типом `T`.

**Что можно вынести в .cpp:**

Методы-шаблоны нельзя вынести в cpp, потому что для них нельзя заранее перечислить все возможные `T`. Explicit instantiation тут невозможна, если ты не знаешь список всех T.

## `typename` vs `class`
В списке параметров шаблона `template <typename T>` и `template <class T>` полностью эквивалентны.

В объявлении `template <template <typename> class Container>` и `template <template <class> class Container>` разницы тоже нет.

Внутри шаблонов (зависимые имена) `typename` обязателен, когда ты говоришь компилятору: *это имя — тип, а не поле или статическая переменная*

```
template <typename T>
void foo() {
    typename T::value_type x;
}
```

Если убрать `typename`:
```
T::value_type x;  // ошибка
```
Компилятор не знает, `value_type` — это тип или статический член.

## Шаблон шаблонов (template template parameter)

`template <template <typename> class Container>` - параметр шаблона, который сам является шаблоном класса.

Здесь объявляется шаблон, который принимает: `Container` — это шаблон, который принимает один параметр типа

```
#include <vector>
#include <list>
#include <iostream>

template <template <typename> class Container>
void print_container(const Container<int>& c) {
    for (const auto& x : c)
        std::cout << x << " ";
}
```
**Если нужна обёртка над контейнером произвольного типа**

***Сценарий 1*** — контейнер уже конкретного типа

```
std::vector<int>
std::list<std::string>
```

Тогда `template template parameter` вообще не нужен:
```
template <typename Container>
struct Wrapper {
    Container data;
};
```

Использование:
```
Wrapper<std::vector<int>> w;
Wrapper<std::list<double>> w2;
```

***Сценарий 2*** — ты хочешь выбрать семейство контейнеров и сам задавать тип

```
Wrapper<std::vector, int>
Wrapper<std::list, double>
```

Тогда нужно два параметра:
```
#include <vector>
#include <list>

template <
    template <typename, typename...> class Container,
    typename T
>
struct Wrapper {
    Container<T> data;
};
```

Использование:
```
Wrapper<std::vector, int> w1;
Wrapper<std::list, double> w2;
```
Обрати внимание на `template <typename, typename...>`. Это нужно, потому что у `std::vector` есть аллокатор:
```
template<class T, class Allocator = std::allocator<T>>
class vector;
```

***Сценарий 2*** нужен, когда ты хочешь сам формировать тип контейнера внутри. Абстрагироваться от вида контейнера, но управлять типом элемента.

Пример — фабрика контейнеров
```
#include <vector>
#include <list>

template <
    template <typename, typename...> class Container
>
struct Factory {
    template <typename T>
    static Container<T> create() {
        return Container<T>{};
    }
};
```

Использование:
```
auto v = Factory<std::vector>::create<int>();
auto l = Factory<std::list>::create<double>();
```
Здесь без `template template parameter` невозможно —
потому что ты создаёшь `Container<T>` внутри.


# variadic templates
```
template <typename... Args>
void f(Args... args);
```

- `Args...` — пакет типов
- `args...` — пакет параметров
- `...` в объявлении — создание пакета
- `...` в использовании — разворачивание пакета

## Как получить первый аргумент:
```
template <typename First, typename... Rest>
void f(First first, Rest... rest);
```
Через `std::tuple`:
```
auto t = std::forward_as_tuple(args...);
auto& first = std::get<0>(t);
```
Минусы:
- не работает при 0 аргументов
- лишняя абстракция
- усложнение без необходимости

## pack expansion
`foo(args...)` компилятор превращает в `foo(arg1, arg2, arg3)`.

## Fold expression (C++17)
Позволяет применить оператор ко всему пакету.
```
(std::cout << ... << args);
```
Разворачивается в цепочку:
```
((std::cout << arg1) << arg2) << arg3
```
Используется, когда:
- операция одинаковая для всех аргументов
- порядок важен
- не нужен доступ по индексу

# type traits

## `std::enable_if`

Шаблонный инструмент для условного включения кода на этапе компиляции.

```
template<bool B, class T = void>
struct enable_if {};

template<class T>
struct enable_if<true, T> {
    using type = T;
};
```

Если `B == true` → существует `enable_if<B, T>::type`.

Если `B == false` → type не существует, и шаблон считается неподходящим.

**Классический пример:**

```
#include <type_traits>

template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
abs_val(T x) {
    return x < 0 ? -x : x;
}
```

Если `T` не целочисленный, функция просто не участвует в разрешении перегрузки.

Упрощённая реализация выглядит так:
```
template<bool B, class T = void>
struct enable_if {};

template<class T>
struct enable_if<true, T> {
    using type = T;
};
```

***Случай 1:*** `abs_val(5)`

Компилятор выводит: `T = int`
Теперь сигнатура становится:
```
typename std::enable_if<std::is_integral<int>::value, int>::type

std::is_integral<int>::value == true
```
Значит: `std::enable_if<true, int>::type`

А это определено как: `using type = int;`

Итого сигнатура превращается в: `int abs_val(int x)`

Функция существует. Всё нормально.

***Случай 2:*** `abs_val(3.14)`

Теперь: `T = double`

Сигнатура:
```
typename std::enable_if<std::is_integral<double>::value, double>::type

std::is_integral<double>::value == false
```
Значит: `std::enable_if<false, double>`

Но у нас НЕТ специализации для `false`.

Следовательно: `enable_if<false, double>::type` НЕ СУЩЕСТВУЕТ.

## `std::is_integral`
Упрощённая версия выглядит так:
```
template<typename T>
struct is_integral : std::false_type {};
```
А затем идут специализации:
```
template<> struct is_integral<bool>               : std::true_type {};
template<> struct is_integral<char>               : std::true_type {};
template<> struct is_integral<signed char>        : std::true_type {};
template<> struct is_integral<unsigned char>      : std::true_type {};
template<> struct is_integral<wchar_t>            : std::true_type {};
template<> struct is_integral<char8_t>            : std::true_type {};
template<> struct is_integral<char16_t>           : std::true_type {};
template<> struct is_integral<char32_t>           : std::true_type {};
template<> struct is_integral<short>              : std::true_type {};
template<> struct is_integral<unsigned short>     : std::true_type {};
template<> struct is_integral<int>                : std::true_type {};
template<> struct is_integral<unsigned int>       : std::true_type {};
template<> struct is_integral<long>               : std::true_type {};
template<> struct is_integral<unsigned long>      : std::true_type {};
template<> struct is_integral<long long>          : std::true_type {};
template<> struct is_integral<unsigned long long> : std::true_type {};
```

`std::true_type` и `std::false_type` по сути:
```
struct true_type  { static constexpr bool value = true;  };
struct false_type { static constexpr bool value = false; };
```

## type traits vs concepts
Основные проблемы type traits:
1. Плохая читаемость
```
template<typename T>
std::enable_if_t<
    std::is_arithmetic_v<T> && !std::is_same_v<T, bool>,
    T>
f(T);
```
2. Диагностика. Ошибка где-то в `type_traits`, а не в сигнатуре функции.

3. Слабый контракт. Требования спрятаны в реализации, а не в интерфейсе.

Когда type traits всё ещё оправдан:
1. Поддержка C++11/14/17
2. Метапрограммирование типов
3. Частичная специализация, где concepts недоступны

**Эквивалент с concepts (для сравнения)**
```
template<std::integral T>
T abs_val(T x);
```

vs

```
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
abs_val(T x);
```
