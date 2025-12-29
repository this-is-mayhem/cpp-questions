- [`std::unique_ptr`](#stdunique_ptr)
- [`std::shared_ptr`](#stdshared_ptr)
  - [`make_shared` vs `shared_ptr<T>(new T)`](#make_shared-vs-shared_ptrtnew-t)
- [`std::weak_ptr`](#stdweak_ptr)
  - [Типовые сценарии применения weak\_ptr](#типовые-сценарии-применения-weak_ptr)


# `std::unique_ptr`

Владеет объектом единолично.

Копировать нельзя, можно перемещать.

Память освобождается при уничтожении `unique_ptr`.

Используется, когда объект имеет единственного владельца. Ресурсы с однозначным владением - элементы контейнера, RAII-объекты, фабричные функции.

```
auto p = std::make_unique<Foo>();
p->bar();
```

# `std::shared_ptr`

Разделяет владение объектом между несколькими указателями.

Ведёт счётчик ссылок (reference count).

Объект удаляется, когда счётчик падает до нуля.

```
auto p1 = std::make_shared<Foo>();
auto p2 = p1; // оба владеют объектом
```

Если один объект логически принадлежит нескольким частям программы и нужно автоматическое освобождение памяти при исчезновении всех владельцев.

`std::shared_ptr` хранит два указателя:
1. На сам управляемый объект.
2. На контрольный блок (control block), где находятся счётчики и вспомогательные данные.

**Структура контрольного блока:**
```
struct control_block {
    std::atomic<long> shared_count; // число shared_ptr
    std::atomic<long> weak_count;   // число weak_ptr + shared_ptr
    void (*deleter)(void*);         // функция удаления
    void* ptr;                      // указатель на сам объект
};
```
**Алгоритм работы**

1. Создание
   
   При `std::make_shared<T>()` выделяется один кусок памяти, содержащий и объект T, и контрольный блок — экономит память и кэш.
   
   При `shared_ptr<T>(new T)` создаются два выделения: одно под объект, второе под блок управления.

2. Копирование
   Копия `shared_ptr` указывает на тот же объект и контрольный блок. `shared_count` увеличивается атомарно (`++shared_count`).

3. Разрушение
   При уничтожении `shared_ptr` выполняется `--shared_count`. Если счётчик стал 0 → вызывается deleter → объект удаляется. Контрольный блок остаётся до тех пор, пока `weak_count > 0`.

**Упрощённая реализация:**
```
template<typename T>
class SharedPtr {
    T* ptr;
    ControlBlock* ctrl;

    void release() {
        if (ctrl && --ctrl->shared_count == 0) {
            delete ptr;
            if (--ctrl->weak_count == 0)
                delete ctrl;
        }
    }

public:
    SharedPtr(T* p) : ptr(p), ctrl(new ControlBlock()) {
        ctrl->shared_count = 1;
        ctrl->weak_count = 1;
    }
    SharedPtr(const SharedPtr& other) : ptr(other.ptr), ctrl(other.ctrl) {
        ++ctrl->shared_count;
    }
    ~SharedPtr() { release(); }
};
```

## `make_shared` vs `shared_ptr<T>(new T)`

1. Одно выделение памяти вместо двух. меньше malloc, лучше locality cache, быстрее.

2. Исключения
   ```
   shared_ptr<T> p(new T, deleter); // если бросит исключение до конструктора shared_ptr — утечка
   auto p = make_shared<T>();      // exception-safe
   ```

**Когда make_shared НЕ подходит**
1. Нужен кастомный deleter
   ```
   shared_ptr<FILE> f(fopen(...), fclose); // make_shared не умеет
   ```

2. Жёсткий контроль времени жизни
   Объект удаляется позже, чем кажется:
   ```
   auto p = make_shared<T>();
   weak_ptr<T> w = p;
   p.reset(); // T НЕ уничтожен, пока жив weak_ptr
   ```

3. Очень большие объекты
Контрольный блок живёт дольше, чем сам объект → удерживается память.


# `std::weak_ptr`

Не владеет объектом, а лишь ссылается на него, не увеличивая счётчик ссылок.

Используется для разрыва циклов `shared_ptr` и проверки, жив ли объект. Применяется, когда нужен доступ к объекту, управляемому `shared_ptr`, без продления его жизни.

```
std::weak_ptr<Foo> w = p1;
if (auto s = w.lock()) { s->bar(); }
```

`weak_ptr` нужен, потому что `shared_ptr` ломает модель владения.

```
struct B;

struct A {
    shared_ptr<B> b;
};

struct B {
    shared_ptr<A> a;
};

auto a = make_shared<A>();
auto b = make_shared<B>();
a->b = b;
b->a = a; // ❌ Утечка памяти
Счётчики никогда не станут 0.
```

**Как надо:**
```
struct B {
    weak_ptr<A> a;
};
```
Теперь
- A владеет B
- B знает про A, но не владеет

Это и есть смысл `weak_ptr`.

## Типовые сценарии применения weak_ptr
1. Обратные ссылки (parent / child)
```
struct Node {
    weak_ptr<Node> parent;
    vector<shared_ptr<Node>> children;
};
```
2. Observer / callback
```
class Listener {
public:
    void onEvent();
};

weak_ptr<Listener> listener;

void emit() {
    if (auto l = listener.lock())
        l->onEvent(); // безопасно
}
```
3. Кэш
```
map<Key, weak_ptr<Resource>> cache;

shared_ptr<Resource> get(Key k) {
    if (auto r = cache[k].lock())
        return r;

    auto r = make_shared<Resource>();
    cache[k] = r;
    return r;
}
```

Ресурс живёт, пока кто-то реально его использует.