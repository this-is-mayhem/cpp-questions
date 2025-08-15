## Статический и динамический полиморфизм
### Статический полиморфизм (compile-time polymorphism)
Перегрузка функций:
```
void print(int x)    { std::cout << "int\n"; }
void print(double x) { std::cout << "double\n"; }

int main() {
    print(42);
    print(3.14);
}
```
Перегрузка операторов:
```
struct Vec {
    int x;
    Vec operator+(const Vec& other) const {
        return {x + other.x};
    }
};
```
[Шаблоны (templates)](templates.md):
```
template<typename T>
T add(T a, T b) { return a + b; }

int main() {
    add(1, 2);       // T = int
    add(1.5, 2.3);   // T = double
}

```

### Динамический полиморфизм (run-time polymorphism)
В основе механизм виртуальных функций и таблиц виртуальных методов (vtable).
```
#include <iostream>

struct Shape {
    virtual void draw() { std::cout << "Shape\n"; }
};

struct Circle : public Shape {
    void draw() override { std::cout << "Circle\n"; }
};

int main() {
    Circle c;
    Shape& ref = c;   // ссылка на базовый класс
    Shape* ptr = &c;  // указатель на базовый класс

    ref.draw(); // динамический вызов → Circle::draw
    ptr->draw(); // динамический вызов → Circle::draw
}
```

```
 ┌───────────────────┐
 │   Object c (Circle)│
 ├───────────────────┤
 │ vptr ─────────────┼──► vtable_Circle
 │ (указатель на VMT) │
 ├───────────────────┤
 │ поля Shape         │
 ├───────────────────┤
 │ поля Circle        │
 └───────────────────┘

vtable_Shape:
[0] → &Shape::draw

vtable_Circle:
[0] → &Circle::draw
```
Что происходит при вызове:
- Берём `vptr` из объекта по адресу `ptr`.
- Достаём адрес функции из `vtable`.
- Вызываем её.
```
ref ──────────────┐
                   │
ptr ────────────┐ │
                 │ │
       ┌─────────▼─▼─────────┐
       │   Circle object     │
       │ vptr ─────────────┐ │
       └───────────────────┘ │
                              │
          vtable_Circle ◄─────┘
          [0] → Circle::draw
```
⚠️ **Object slicing** со ссылками:
```
struct Shape { void virtual draw() = 0; };

struct Circle : public Shape {
   void draw() override { std::cout << "ooo\n"; }
};

struct Triangle : public Shape {
   void draw() override { std::cout << "^^^\n"; }
};

int main() {
   Circle c{};
   Triangle t{};
   Shape& ps = c;
   ps.draw(); // вывод "ooo"
   ps = t;
   ps.draw(); // вывод "^^^"
   return 0;
   }
```
`ps` — это ссылка на `Shape`, но уже привязанная к объекту `c (Circle)`. Когда ты пишешь `ps = t;`, это не меняет объект, на который указывает ссылка. Ссылки в C++ неизменяемы — их нельзя "переназначить". Вместо этого вызывается `Shape::operator=`, который копирует только часть `Shape` из `t` в `c`.

В момент создания `ps` у нас в памяти есть объект c типа `Circle` с правильным `vptr` (указателем на таблицу виртуальных методов для `Circle`). При `ps = t;` в объект c копируется только базовая часть `Shape` из `t` (а `t` — это `Triangle`). Но `vptr` не меняется, потому что у `Shape` нет своего `operator=` (и виртуальный оператор= не объявлен). В итоге `c` остаётся объектом `Circle` с `vptr`, указывающим на `Circle::draw`.