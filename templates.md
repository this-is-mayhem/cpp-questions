- [Реализация в `.cpp`](#реализация-в-cpp)
  - [Явная инстанциация (explicit instantiation)](#явная-инстанциация-explicit-instantiation)
  - [Полная специализация](#полная-специализация)
- [Шаблонный класс и шаблонные методы](#шаблонный-класс-и-шаблонные-методы)
  - [Шаблонный класс](#шаблонный-класс)
  - [Шаблонный класс и шаблонные методы](#шаблонный-класс-и-шаблонные-методы-1)
- [type traits](#type-traits)
  - [`std::enable_if`](#stdenable_if)


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