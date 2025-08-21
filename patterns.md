- [RAII](#raii)
- [SOLID](#solid)
  - [S — Single Responsibility Principle (Принцип единственной ответственности)](#s--single-responsibility-principle-принцип-единственной-ответственности)
  - [O — Open/Closed Principle (Принцип открытости/закрытости)](#o--openclosed-principle-принцип-открытостизакрытости)
  - [L — Liskov Substitution Principle (Принцип подстановки Лисков)](#l--liskov-substitution-principle-принцип-подстановки-лисков)
  - [I — Interface Segregation Principle (Принцип разделения интерфейса)](#i--interface-segregation-principle-принцип-разделения-интерфейса)
  - [D — Dependency Inversion Principle (Принцип инверсии зависимостей)](#d--dependency-inversion-principle-принцип-инверсии-зависимостей)
- [Фабрики (factory)](#фабрики-factory)
  - [Пример 1. GUI-элементы](#пример-1-gui-элементы)
  - [Пример 2. Сериализация / загрузка объектов](#пример-2-сериализация--загрузка-объектов)

# RAII
**Resource Acquisition Is Initialization** это паттерн управления ресурсами через объекты.

RAII использует жизненный цикл объекта для управления ресурсом. Когда объект создаётся (конструктор вызывается) → ресурс захватывается. Когда объект уничтожается (деструктор вызывается) → ресурс освобождается.

Типичные ресурсы:
- Память (new/delete, std::unique_ptr)
- Файлы (std::fstream)
- Мьютексы, блокировки (std::lock_guard)
- Сокеты и дескрипторы

Преимущества:
- Нет утечек ресурсов: деструкторы освобождают всё корректно.
- Безопасно при исключениях: даже если функция вылетела с ошибкой, ресурс будет освобождён.
- Код становится чище: не нужно вручную писать `free()`, `delete`, `close()`.

# SOLID
Набор пяти базовых принципов объектно-ориентированного программирования, которые помогают писать читаемый, расширяемый и поддерживаемый код.

## S — Single Responsibility Principle (Принцип единственной ответственности)
Класс должен иметь только одну причину для изменения. То есть каждый класс отвечает за одну конкретную задачу.

Пример: система заказов в интернет-магазине.

Как не надо (класс отвечает за хранение данных и за их вывод.):
```
class Invoice {
public:
    void addItem(const std::string& item, double price) {
        items.push_back({item, price});
    }

    void printInvoice() { // Печать здесь не нужна
        for (auto& item : items) {
            std::cout << item.first << ": " << item.second << "\n";
        }
    }
private:
    std::vector<std::pair<std::string, double>> items;
};
```
Как надо:
```
class Invoice {
public:
    void addItem(const std::string& item, double price) {
        items.push_back({item, price});
    }
    const auto& getItems() const { return items; }
private:
    std::vector<std::pair<std::string, double>> items;
};

class InvoicePrinter {
public:
    void print(const Invoice& invoice) {
        for (auto& item : invoice.getItems()) {
            std::cout << item.first << ": " << item.second << "\n";
        }
    }
};
```


## O — Open/Closed Principle (Принцип открытости/закрытости)
Программные сущности должны быть открыты для расширения, но закрыты для изменения. То есть мы можем добавлять новую функциональность без изменения существующего кода.

Как не надо:
```
class Shape {
public:
    enum Type { Circle, Rectangle };
    Type type;
    double radius;
    double width, height;
};

double calculateArea(Shape& s) {
    if (s.type == Shape::Circle)
        return 3.14 * s.radius * s.radius;
    else
        return s.width * s.height;
}
```
Как надо:
```
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14 * radius * radius; }
private:
    double radius;
};

class Rectangle : public Shape {
public:
    Rectangle(double w, double h) : width(w), height(h) {}
    double area() const override { return width * height; }
private:
    double width, height;
};

double calculateArea(const Shape& s) {
    return s.area(); // Работает с любым новым Shape
}
```

## L — Liskov Substitution Principle (Принцип подстановки Лисков)
Объекты подкласса должны быть заменимы объектами базового класса без изменения поведения программы.

Как не надо:
```
class Bird {
public:
    virtual void fly() {}
};

class Penguin : public Bird {
public:
    void fly() override { throw std::logic_error("Penguins can't fly"); }
};
```
Как надо:
```
class Bird {
public:
    virtual void move() = 0;
};

class Sparrow : public Bird {
public:
    void move() override { std::cout << "Sparrow flies\n"; }
};

class Penguin : public Bird {
public:
    void move() override { std::cout << "Penguin walks\n"; }
};
```

## I — Interface Segregation Principle (Принцип разделения интерфейса)
Лучше иметь много узкоспециализированных интерфейсов, чем один большой «толстый». То есть класс не должен реализовывать методы, которые ему не нужны.

Как не надо:
```
class MultiFunctionPrinter {
public:
    virtual void print() = 0;
    virtual void scan() = 0;
    virtual void fax() = 0;
};
```
Проблема: Не все принтеры умеют сканировать и отправлять факсы.

Как надо:
```
class IPrinter { public: virtual void print() = 0; };
class IScanner { public: virtual void scan() = 0; };
class IFax { public: virtual void fax() = 0; };

class OldPrinter : public IPrinter {
public:
    void print() override { std::cout << "Printing...\n"; }
};
```

## D — Dependency Inversion Principle (Принцип инверсии зависимостей)
Модули верхнего уровня не должны зависеть от модулей нижнего уровня — оба должны зависеть от абстракций (интерфейсов).

Как не надо:
```
class FileLogger {
public:
    void log(const std::string& msg) { std::cout << msg << "\n"; }
};

class OrderProcessor {
    FileLogger logger;
public:
    void process() { logger.log("Processing order"); }
};
```

Как надо:
```
class ILogger {
public:
    virtual void log(const std::string& msg) = 0;
    virtual ~ILogger() = default;
};

class FileLogger : public ILogger {
public:
    void log(const std::string& msg) override { std::cout << msg << "\n"; }
};

class OrderProcessor {
    ILogger& logger;
public:
    OrderProcessor(ILogger& log) : logger(log) {}
    void process() { logger.log("Processing order"); }
};
```
Теперь можно легко менять реализацию логирования (например, на `ConsoleLogger` или `NetworkLogger`) без изменения `OrderProcessor`.

# Фабрики (factory)
 Для создания объектов полиморфно обычно используют один из двух подходов, и оба связаны с идеей фабрики:
 ```
 class Base {
public:
    virtual ~Base() {}
    virtual Base* create() const = 0; // "виртуальный конструктор"
};

class Derived : public Base {
public:
    Base* create() const override { return new Derived(); }
};

void example() {
    Base* obj = new Derived();
    Base* copy = obj->create(); // создаём полиморфно объект типа Derived
    delete obj;
    delete copy;
}
```
`create()` действует как виртуальный конструктор — вызывается в зависимости от реального типа объекта.

Отдельная фабрика (Factory Class / Function). Создаёт объекты нужного типа по условию или параметру:
```
enum Type { BASE, DERIVED };

Base* Factory(Type t) {
    switch(t) {
        case BASE: return new Base();
        case DERIVED: return new Derived();
    }
    return nullptr;
}
```

🔹 Смысл фабрик появляется не в месте первого создания объекта, а в моменте, когда нам нужно **создать новый объект, не зная точного типа**.

Без "виртуального конструктора":
```
std::vector<Base*> vec;
vec.push_back(new Derived());
vec.push_back(new AnotherDerived());

// Теперь хотим скопировать объекты:
std::vector<Base*> copy;
for (Base* obj : vec) {
    copy.push_back(new Base(*obj)); // ❌ не скомпилится или создастся не тот тип
}
```
Проблема: у нас есть только `Base*`, и мы не знаем, какой конкретно тип скрывается за указателем (`Derived`, `AnotherDerived`, ещё какой-то), поэтому:
```
class Base {
public:
    virtual ~Base() {}
    virtual Base* clone() const = 0; // "виртуальный конструктор"
};

class Derived : public Base {
public:
    Base* clone() const override {
        return new Derived(*this);
    }
};

class AnotherDerived : public Base {
public:
    Base* clone() const override {
        return new AnotherDerived(*this);
    }
};

std::vector<Base*> vec;
vec.push_back(new Derived());
vec.push_back(new AnotherDerived());

// Теперь копируем полиморфно:
std::vector<Base*> copy;
for (Base* obj : vec) {
    copy.push_back(obj->clone()); // ✅ вызовется правильный конструктор
}
```
Теперь мы можем дублировать/создавать объекты правильного типа не зная их реального типа на этапе компиляции — это и есть имитация «виртуального конструктора». При первом создании объекта, конечно, нужно явно знать, что создаёшь (`new Derived()`). Но при последующих операциях (копирование, клонирование, фабричное создание по конфигурации) может быть доступен только `Base*`, и именно тогда нужен "виртуальный конструктор" (clone или фабрика).

## Пример 1. GUI-элементы
Есть базовый класс `Widget` и куча наследников: `Button`, `Label`, `TextBox` и т.д.
```
class Widget {
public:
    virtual ~Widget() {}
    virtual void draw() = 0;
    virtual Widget* clone() const = 0;
};

class Button : public Widget {
public:
    void draw() override { std::cout << "Button\n"; }
    Widget* clone() const override { return new Button(*this); }
};

class Label : public Widget {
public:
    void draw() override { std::cout << "Label\n"; }
    Widget* clone() const override { return new Label(*this); }
};
```
Есть форма, которая хранит список виджетов:
```
std::vector<Widget*> widgets;
widgets.push_back(new Button());
widgets.push_back(new Label());
```
Если пользователь нажимает «Дублировать виджет», то есть только `Widget*`. Как узнать, что это `Button` или `Label`? С `clone()` всё просто:
```
std::vector<Widget*> copied;
for (Widget* w : widgets) {
    copied.push_back(w->clone()); // вызывает правильный конструктор
}
```
## Пример 2. Сериализация / загрузка объектов
Есть игра, где есть `Enemy`, `Player`, `Item` — все наследники `GameObject`. Если просто хранить `GameObject*`, то при загрузке непонятно, как создать правильный объект. Тут удобно сделать фабрику:
```
class GameObject {
public:
    virtual ~GameObject() {}
    virtual void load(std::istream& in) = 0;
};

class Enemy : public GameObject {
public:
    void load(std::istream& in) override { /* читаем Enemy */ }
};

class Player : public GameObject {
public:
    void load(std::istream& in) override { /* читаем Player */ }
};

GameObject* create(const std::string& type) {
    if (type == "Enemy") return new Enemy();
    if (type == "Player") return new Player();
    return nullptr;
}
```
Теперь при чтении файла:
```
std::string type;
while (in >> type) {
    GameObject* obj = create(type); // фабрика выбирает правильный класс
    obj->load(in);
    world.push_back(obj);
}
```
