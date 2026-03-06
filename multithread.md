Многопоточность — это выполнение нескольких потоков исполнения внутри одного процесса.

В C++ каждый поток делит память с другими потоками, но у каждого потока есть свой стек (локальные переменные).
```
int shared = 0; // общая память
void threadFunc() {
    int local = 0; // локальная переменная потока
    shared++;      // опасная операция без синхронизации
}
```

На уровне процессора `shared++` состоит из трёх шагов:
```
load — загрузить значение из памяти
add 1 — увеличить
store — записать обратно
```

Если два потока делают это одновременно, один результат затрёт другой. Каждый поток работает на ядре CPU, которое имеет свой L1/L2 кэш. Когда один поток меняет значение, кэш другого ядра может быть устаревшим. CPU автоматически синхронизирует кэши при атомарных операциях и блокировках.

Следствие: без атомарных операций или мьютексов результат будет непредсказуемым.




# `std::thread`

Создаёт поток.

```
#include <iostream>
#include <thread>

void work() {
    std::cout << "thread running\n";
}

int main() {
    std::thread t(work);
    t.join();
}
```

`join()` — ждать завершения потока.

Отделяет поток:
```
t.detach();
```

Поток продолжает работать независимо. Это опасно - поток может обратиться к уничтоженным данным.


# `std::mutex`

Защищает данные от одновременного доступа.

```
#include <mutex>

std::mutex m;

void work() {
    m.lock();
    // критическая секция
    m.unlock();
}
```

Обычно используют `std::lock_guard`:
```
#include <mutex>

std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;
}
```

lock_guard позволяет автоматически `unlock()` при выходе из области видимости. Уменьшает шанс deadlock и забыть разблокировать.


# `std::atomic`

Атомарные операции без блокировки.
```
#include <atomic>

std::atomic<int> counter;
counter++;
```

C++ стандарт позволяет управлять тем, как изменения видны другим потокам.
```
std::atomic<int> x(0);

x.store(1, std::memory_order_relaxed);  // быстрый, может видеться не сразу другим потокам
int y = x.load(std::memory_order_relaxed);
```

- `memory_order_relaxed` - не гарантирует порядок
- `memory_order_acquire / release` — гарантирует правильный порядок для флагов, сигналов
- `memory_order_seq_cst` — полная последовательность, самый безопасный вариант

**Современная практика:** начинаем с `seq_cst`, оптимизируем только если реально нужны миллисекунды.


# `std::condition_variable`

Позволяет потокам ждать события. Например: producer → кладёт данные, consumer → ждёт пока данные появятся.

```
#include <condition_variable>
#include <mutex>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> data;
bool finished = false;

void producer() {
    for(int i=0; i<10; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        data.push(i);
        cv.notify_one(); // сигнал потребителю
    }
    std::lock_guard<std::mutex> lock(mtx);
    finished = true;
    cv.notify_all();
}

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    while(!finished || !data.empty()) {
        cv.wait(lock, []{ return !data.empty() || finished; });
        while(!data.empty()) {
            int val = data.front();
            data.pop();
            // обрабатываем val
        }
    }
}
```



# `std::future` и `std::async`

Позволяют запускать задачи и получать результат.
```
#include <future>

int work() {
    return 42;
}

auto f = std::async(work);

int result = f.get();
```


# `std::promise`

Позволяет передать результат из потока.


# `std::shared_mutex`

Полезно, когда много читателей и 1 писатель. Полезно для read-heavy структур.


# Thread Local Storage

Переменная уникальна для каждого потока: `thread_local int x;`.


# Основные проблемы многопоточности

## Data race

Несколько потоков меняют данные без синхронизации.

## Deadlock
Два потока ждут друг друга.
```
thread1 lock A → ждёт B
thread2 lock B → ждёт A
```

## False sharing

Два потока меняют разные переменные, но они в одном cache line. Это резко замедляет программу.


# Типичная архитектура - Thread pool

Часто используют **thread pool**:
```
#include <thread>
#include <queue>
#include <functional>
#include <vector>
#include <mutex>
#include <condition_variable>

class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;

public:
    ThreadPool(size_t n) {
        for(size_t i=0;i<n;i++)
            workers.emplace_back([this]{ this->worker(); });
    }

    void worker() {
        while(true) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [this]{ return stop || !tasks.empty(); });
                if(stop && tasks.empty()) return;
                task = tasks.front(); tasks.pop();
            }
            task();
        }
    }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            tasks.push(task);
        }
        cv.notify_one();
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            stop = true;
        }
        cv.notify_all();
        for(auto &t : workers) t.join();
    }
};
```
