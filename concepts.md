- [Зачем?](#зачем)
- [Базовый пример](#базовый-пример)
- [`requires`](#requires)
- [Стандартные концепты](#стандартные-концепты)
- [Формы записи](#формы-записи)


Concepts (концепты) в - это механизм языка (C++20), позволяющий задавать требования к шаблонным параметрам на уровне интерфейса, а не в виде неявных ошибок компиляции. Формально: concept — это именованное логическое ограничение (bool-предикат времени компиляции) на тип или набор типов.

# Зачем?

До C++20 шаблоны принимали любой тип, а ошибки возникали глубоко внутри инстанцирования:

```
template<class T>
void f(T x) {
    x.push_back(1); // ошибка где-то тут
}
```

Если `T` не имеет `push_back`, компилятор выдаёт длинную и плохо читаемую диагностику.

Concepts решают три проблемы:
1. Явные требования к типам
2. Понятные сообщения об ошибках
3. Контроль перегрузок шаблонов

# Базовый пример

```
#include <concepts>

template<typename T>
concept Addable = requires(T a, T b) {
    a + b;
};

template<Addable T>
T sum(T a, T b) {
    return a + b;
}

// sum(1, 2) — OK
// sum(std::string("a"), "b") — OK
// sum(std::vector<int>{}, {}) — ошибка на границе интерфейса
```

# `requires`

Выражение для проверки существования и свойств операций

```
template<typename T>
concept Incrementable = requires(T x) {
    ++x;        // операция существует
    x++;        // и эта тоже
};
```

Можно проверять типы возвращаемых значений:

```
template<typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};
```

# Стандартные концепты

В `<concepts>` уже есть базовые:
- `std::integral`
- `std::floating_point`
- `std::copyable`
- `std::movable`
- `std::totally_ordered`
- `std::same_as<T, U>`

Пример:

```
template<std::integral T>
T gcd(T a, T b);
```

Это строже и яснее, чем `std::enable_if`.

# Формы записи

```
template<typename T>
requires std::integral<T>
T f(T);
```

```
template<std::integral T>
T f(T);
```

```
auto f(std::integral auto x);
```