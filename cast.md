- [`static_cast`](#static_cast)
- [`const_cast`](#const_cast)
- [`reinterpret_cast`](#reinterpret_cast)
- [`dynamic_cast`](#dynamic_cast)
- [`bit_cast`](#bit_cast)
- [`static_cast` vs `dynamic_cast`](#static_cast-vs-dynamic_cast)
- [`bit_cast` vs `reinterpret_cast`](#bit_cast-vs-reinterpret_cast)


# `static_cast`

Безопасное (во время компиляции) приведение между совместимыми типами.

Используется для:
- Преобразование чисел: `double` → `int`, `int` → `float`.
- Преобразование указателей/ссылок в пределах иерархии наследования (если известно, что объект реально того типа).
- Явное преобразование конструкторов/операторов приведения.

```
double x = 3.14;
int n = static_cast<int>(x);  // n = 3

Base* b = new Derived();
Derived* d = static_cast<Derived*>(b);  // допустимо, если реально Derived
```

Особенности:
- Проверка на этапе компиляции.
- Не меняет `const/volatile`.
- Не выполняет `reinterpret` (т.е. не меняет представление памяти).

# `const_cast`

Удаляет или добавляет квалификаторы `const` или `volatile`.

Используется для:
- Вызова функции, которая по ошибке не принимает `const`, хотя не должна менять объект.
- Пример допустимого применения - совместимость со старым API.

```
void print(char* s);
void foo(const char* msg) {
    print(const_cast<char*>(msg));  // легально, если print не меняет строку
}
```

***Опасность:*** Изменение объекта, который изначально был `const`, - неопределённое поведение.

# `reinterpret_cast`

Побитовая переинтерпретация указателей или целых чисел.

Используется для:
- Низкоуровневого доступа к памяти.
- Преобразования указателей в целые и обратно.
- Преобразования разных указателей (например `void*` → `int*`).

```
int x = 1;
char* p = reinterpret_cast<char*>(&x);  // доступ к байтам int
uintptr_t addr = reinterpret_cast<uintptr_t>(p);  // указатель в число
```

Особенности:
- Не изменяет байты, только трактовку.
- ***Опасен***, может нарушить [strict aliasing](low_level.md#strict-aliasing) или [alignment](low_level.md#alignment).

# `dynamic_cast`
Проверяет корректность приведения указателя или ссылки между типами в иерархии классов **во время выполнения**.
Работает только при наличии хотя бы одной **виртуальной функции** в базовом классе.

Применяется:
- Для "безопасного спуска" по иерархии (downcast).
- Для проверки реального типа объекта при работе через базовый указатель.

```
struct Base { virtual ~Base() = default; };
struct Derived : Base {};
struct Other : Base {};

Base* b = new Derived();
Derived* d = dynamic_cast<Derived*>(b);  // OK, d != nullptr
Other* o = dynamic_cast<Other*>(b);      // nullptr, тип не совпадает
```

```
Base& br = *b;
try {
    Derived& dr = dynamic_cast<Derived&>(br);
} catch (const std::bad_cast&) {
    // если тип не подходит
}
```

Особенности:
- Проверка типа выполняется через таблицу виртуальных функций (RTTI).
- Работает только при полиморфизме.
- Медленнее `static_cast`, но безопаснее.

# `bit_cast`

Безопасное побитовое копирование значений между типами одинакового размера и trivially copyable. Интерпретация представления одного типа как другого без UB.

```
#include <bit>
float f = 3.14f;
uint32_t bits = std::bit_cast<uint32_t>(f);  // получить IEEE 754 представление
float f2 = std::bit_cast<float>(bits);       // восстановить
```

# `static_cast` vs `dynamic_cast`

`static_cast` и `dynamic_cast` оба могут использоваться для приведения в иерархии классов, но их механизмы и гарантии разные.

| Свойство          | `static_cast`                                                  | `dynamic_cast`                              |
| ----------------- | -------------------------------------------------------------- | ------------------------------------------- |
| Проверка          | Во время **компиляции**                                        | Во время **выполнения**                     |
| Требует `virtual` | Нет                                                            | Да (для RTTI)                               |
| Безопасность      | Не проверяет реальный тип, возможен UB                         | Проверяет реальный тип, безопасен           |
| Скорость          | Быстро (в compile-time)                                        | Медленнее (через RTTI)                      |
| Пример ошибки     | Приведение `Base*` к `Derived*`, если объект не `Derived` → UB | То же приведение → `nullptr` или исключение |
| Применение        | Когда точно известен реальный тип                              | Когда тип может быть неизвестен             |

```
struct Base { virtual ~Base() = default; };
struct Derived : Base {};
struct Other : Base {};

Base* b = new Derived();

// static_cast — без проверки:
Derived* d1 = static_cast<Derived*>(b);  // OK, но компилятор не проверяет
Other* o1 = static_cast<Other*>(b);      // UB, если вызвать методы

// dynamic_cast — с проверкой:
Derived* d2 = dynamic_cast<Derived*>(b); // d2 != nullptr
Other* o2 = dynamic_cast<Other*>(b);     // o2 == nullptr
```

# `bit_cast` vs `reinterpret_cast`

- `reinterpret_cast` не разрешён в `constexpr`
- `reinterpret_cast` не даёт гарантий о безопасном копировании представлений битов между различными trivially copyable типами. Он даёт «переинтерпретацию» указателя или ссылки, что может нарушать правила aliasing. `bit_cast` обеспечивает чёткий контракт: результат — новый объект, побитовая копия.
- С точки зрения дизайна языка: разные операции с разными намерениями должны быть выражены разными кастами. `reinterpret_cast` — «опасный», `bit_cast` — «легально побитовое копирование».
- `bit_cast` не нарушает strict aliasing, поскольку не переинтерпретирует существующий объект, а создаёт новый объект (копией бит).
- `reinterpret_cast` при переинтерпретации может нарушать strict aliasing или alignment, что делает поведение неопределённым.
