- [Как запретить наследование?](#как-запретить-наследование)
- [Ромбовидное наследование](#ромбовидное-наследование)
- [Чисто виртуальный метод](#чисто-виртуальный-метод)

### Как запретить наследование?
1. `final` в классе:
   ```
   class MyClass final {
    public:
    void foo();
    };
   ```
2. `final` в методе:
   ```
   class Base {
   public:
   virtual void foo() final;
   };
   ```
3. `private` конструктор:
   ```
   class MyClass {
   private:
   MyClass() {}
   friend MyClass create();
   };
   ```

### Ромбовидное наследование
Проблема:
```
struct A { int x; };
struct B : public A {};
struct C : public A {};
struct D : public B, public C {};

int main() {
    D d;
    d.x = 42; // ❌ Ошибка: неоднозначно, откуда x — из B или из C
}
```
Решение с `virtual`:
```
struct A { int x; };
struct B : public virtual A {};
struct C : public virtual A {};
struct D : public B, public C {};

int main() {
    D d;
    d.x = 42; // ✅ Одна общая копия A
}
```
При обычном наследовании каждая цепочка получает свою копию базового класса. При виртуальном наследовании компилятор хранит одну копию и добавляет «указатели смещения» (vtable-like механизм) для доступа к ней.

Другие способы:
1. Явно указывать, через какую ветку обращаться:
   ```
   D d;
   d.B::x = 10;
   d.C::x = 20;
   ```
2. Композиция вместо наследования:
   ```
   struct A { int x; };
   struct B { A* a; };
   struct C { A* a; };

   struct D : public B, public C {
    A a_obj;
    D() {
        B::a = &a_obj;
        C::a = &a_obj;
      }
   };
   ```

### Чисто виртуальный метод
Нет реализации в базовом классе. Обязаны переопределить наследники, если хотят быть конкретными классами (instantiable):
```
struct Base {
    virtual void foo() = 0;
};
```
Вызов чисто виртуальной функции (pure virtual call) в конструкторе или деструкторе базового класса:
```
struct Base {
    Base() { foo(); } // опасно!
    virtual void foo() = 0;
};

struct Derived : Base {
    void foo() override { /* ... */ }
};

int main() {
    Derived d; // ❌ pure virtual call при вызове foo() из Base()
}
```
