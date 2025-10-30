- [`const`](#const)
- [`constexpr`](#constexpr)
  - [`constexpr` конструкторы](#constexpr-конструкторы)
  - [`if constexpr`](#if-constexpr)
- [`consteval`](#consteval)


# `const`

Означает неизменяемость объекта после инициализации.

Может быть временной или глобальной переменной, указателем, ссылкой, параметром функции. Значение **не обязано быть известно на этапе компиляции**.

```
int f() { return std::rand(); }
const int y = f(); // допустимо, но не constexpr
```

# `constexpr`

Означает, что значение должно быть вычислимо во время компиляции, если возможно.

```
constexpr int a = 2 + 3; // вычисляется на этапе компиляции
```

Используется для:
- констант времени компиляции
- функций
- конструкторов
- if constexpr

```
constexpr int square(int n) { return n * n; }

constexpr int b = square(5);   // ОК
int c = square(std::rand());   // не constexpr, но допустимо
```

`constexpr` нельзя сделать виртуальным. Класс с виртуальными методами не может иметь `constexpr` конструктор (vtable формируется во время выполнения).
То есть `constexpr` и полиморфизм несовместимы.

С C++20 допускается `new/delete` внутри `constexpr`, если выделение и освобождение происходят в compile-time контексте и объект не “утекает”:
```
constexpr int f() {
    int* p = new int(5);
    int v = *p;
    delete p;
    return v;
}
constexpr int x = f(); // OK с C++20
```

Можно генерировать таблицы на этапе компиляции:
```
constexpr auto make_table() {
    std::array<int, 5> t{};
    for (int i = 0; i < 5; ++i) t[i] = i * i;
    return t;
}
constexpr auto table = make_table(); // compile-time генерация
```


## `constexpr` конструкторы
Позволяют создавать объекты, чьи значения известны во время компиляции. Это нужно, когда объект участвует в других `constexpr` выражениях.

```
struct Vec2 {
    double x, y;
    constexpr Vec2(double x_, double y_) : x(x_), y(y_) {}
    constexpr double length2() const { return x*x + y*y; }
};

constexpr Vec2 v(3.0, 4.0);     // создается на этапе компиляции
constexpr double len2 = v.length2(); // тоже вычисляется на этапе компиляции
```

Условия для `constexpr` конструктора:
1. Все члены должны быть инициализированы константно (через список инициализации).
2. Конструктор не должен содержать недопустимых действий: циклов с неопределённым количеством итераций, выделения памяти, виртуальных вызовов и т.д.
3. После C++20 допускаются более сложные конструкции, включая `if`, `for`, но всё должно быть вычислимо при компиляции, если аргументы — константы.

## `if constexpr`
Условие, вычисляемое во время компиляции.
Позволяет компилятору удалить неактуальные ветки кода, что делает возможным шаблонные конструкции без ошибок компиляции в неиспользуемых ветках.

```
template<typename T>
void print(T value) {
    if constexpr (std::is_integral_v<T>)
        std::cout << value << " — целое\n";
    else
        std::cout << value << " — нецелое\n";
}
```
Если `T` — целое, то код для второй ветки вообще не компилируется. Без `constexpr` компилятор попытался бы скомпилировать обе ветки и выдал ошибку, если бы во второй были недопустимые выражения.

Можно “разветвлять” compile-time логику внутри `constexpr` функции:
```
template<typename T>
constexpr int type_code() {
    if constexpr (std::is_integral_v<T>) return 1;
    else if constexpr (std::is_floating_point_v<T>) return 2;
    else return 0;
}
```

# `consteval`
Означает, что **функция обязательно вычисляется при компиляции**.
Если результат нельзя вычислить во время компиляции — ошибка.

```
consteval int cube(int x) { return x * x * x; }

constexpr int v = cube(3); // ОК
int n = 3;
int r = cube(n); // ошибка: n не constexpr
```