std::atomic помогают решить проблему гонки потоков, например:
```cpp
int thheadCount = 10;
int iterations = 1000;

int counter = 0;

int main() {
    auto increment = []() {
        for (int i = 0; i < iterations; ++i)
        {
            counter++;
            auto nameSmth = std::to_string(counter);
            // ...
        }
    }
    std::vector<std::thread> threads;
    for (int i = 0; i < threadCoount; ++i) {
        threads.emplace_back(increment);
    }
    for (auto& thread : threads) {
        thread.join();
    }
    std::cout << "Counter: " << counter << std::endl;
    return 0;
}
```

В идеале мы должны были бы всегда в терминале видеть значение Counter: 10000, так как у нас 10 потоков и 1000 операций инкремента. Но такого практически никогда не пройзойдет, потому что при инкременте на самом деле происходят следующие шаги:
- Читаем из RAM значение переменной
- Помещаем значение в регистр и производим инкремент
- Сохраняем новое значение в оперативную память

Размерем на примере двух потоков, почему не всегда инкремент в двух потоках будет приводить к двойному инкременту:

1) Оба потока считывают значение counter из оперативной памяти
Тут оба потока считали одинаковое значение из оперативной памяти
2) Оба потока записывают значение в регистр и производят операцию инкремента
3) Оба потока записывают новое значение в оперативную память

Последовательность внутри пунктов 1-3 не так важна, какой поток 1 или 2 будет первым считывать значение, главное, что пункт 3 одного потока не случился раньше пункта 1 для другого потока, тем самым два потока хоть и сделали инкремент переменной, но последний поток который выполняет последовательность записывает единственный инкремент.

Мы можем решить эту проблему с помощью мьютекса:
```cpp
int thheadCount = 10;
int iterations = 1000;

int counter = 0;
std::mutex m;

int main() {
    auto increment = []() {
        std::lock_guard<std::mutex> g(m); // Добавили RAII блокировку мьютекса
        for (int i = 0; i < iterations; ++i)
        {
            counter++;
            auto nameSmth = std::to_string(counter);
            // ...
        }
    }
    std::vector<std::thread> threads;
    for (int i = 0; i < threadCoount; ++i) {
        threads.emplace_back(increment);
    }
    for (auto& thread : threads) {
        thread.join();
    }
    std::cout << "Counter: " << counter << std::endl;
    return 0;
}
```
И с этим изменением мы получим в терминале ожидаемое значение 10000, Единственная проблема, блокировка мьютеса - довольно дорогое удовольствие в данной ситуации.
Есть другой, более дешевый способ как обезопасить параллельность, с помощью atomic:
```cpp
int thheadCount = 10;
int iterations = 1000;

sdt::atomic<int> counter = 0; // Одна измененная строчка с оригинального алгоритма

int main() {
    auto increment = []() {
        for (int i = 0; i < iterations; ++i)
        {
            counter++;
            auto nameSmth = std::to_string(counter);
            // ...
        }
    }
    std::vector<std::thread> threads;
    for (int i = 0; i < threadCoount; ++i) {
        threads.emplace_back(increment);
    }
    for (auto& thread : threads) {
        thread.join();
    }
    std::cout << "Counter: " << counter << std::endl;
    return 0;
}
```

Также в 20-м стандарте появился замечательный jthread, который при деструкции вызывает join объекта, мы можем чуточку упростить наш код:
```cpp
int thheadCount = 10;
int iterations = 1000;

sdt::atomic<int> counter = 0; // Одна измененная строчка с оригинального алгоритма

int main() {
    auto increment = []() {
        for (int i = 0; i < iterations; ++i)
        {
            counter++;
            auto nameSmth = std::to_string(counter);
            // ...
        }
    }
    { // Мы ограничили область видимости jthread чтобы их деструкция вызывалась до того как мы выводим значение Counter при уничтожении вектора
        std::vector<std::jthread> threads;
        for (int i = 0; i < threadCoount; ++i) {
            threads.emplace_back(increment);
        }
    }
    
    std::cout << "Counter: " << counter << std::endl;
    return 0;
}
```

Немного про atomic:
- Некопируемый и неперемещаемый
- С C++11 поддержка целочисленных типов, bool, pointers
- С С++20 поддержка shared_ptr, weak_ptr
- С С++23 поддержка Floating-point literals (double, float и так далее)

Какие операции производятся над atomic:
- Складывать
- Вычитать
- Присваивать новое значение

```cpp
counter *= 2; // fail - не можем умножать
counter = counter + 1; // Не ломает код, но тут операция не будет атомарной, тут будет две атомарные операции counter + 1 и оператор =

counter = 10; // Для записи (не получится читать из другого атомика, из-за запрета на копирование)
counter.store(10); // Другой вариант для записи в атомик, этот можно использовать чтобы читать из другого атомика

std::atomic<int> counter2;
counter2 = counter; // fail
counter2.store(counter); // ok

int x = counter; // Для чтение
int x2 = counter.load(); // Также для чтения

counter += 2; // Добавление нового значения
counter.fetch_add(2); // также добавление нового значения
```

Зачем же нужны отдельные методы, если есть обычные операторы?
Дело в том что методы store, load, fetch_add принимают как дополнительный параметр memory_order. Он может вам понадобиться если важна последовательность операций. Об этом будут отдельные заметки.

Из-за того что атомик как правило работает быстрее мьютексов, может возникнуть соблазн использовать атомики чаще чем следует и в местах где это делать не требуется.

Давайте рассмотрим несколько примеров и посмотрим как делать константные методы класса потокобезопасными:

```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;

    Polynomial(double a, double b, double c) : a(a), b(b), c(c) {}
    RootsType roots() const {
        Rootstype rootVals;
        double discriminant = b * b - 4 * a * c;

        if (discriminant > 0) {
            rootVals.push_back((-b + std::sqrt(discriminant)) / (2 * a));
            rootVals.push_back((-b +- std::sqrt(discriminant)) / (2 * a));
        }
        else if (discriminant == 0) {
            rootVals.push_back(-b / (2 * a));
        }
        return rootVals;
    }
private:
    double a, b, c;
}
```
Тут используются так называемые отложенные или ленивые вычисления. Но есть одна проблема, код не оптимизирован, при неизменных a, b и с мы высчитываем корни каждый раз когда вызывается метод roots. Пофиксим это:
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;

    Polynomial(double a, double b, double c) : a(a), b(b), c(c) {}
    RootsType roots() const {
        if (!rootsAreValid) {
            double discriminant = b * b - 4 * a * c;

            if (discriminant > 0) {
                rootVals.push_back((-b + std::sqrt(discriminant)) / (2 * a));
                rootVals.push_back((-b +- std::sqrt(discriminant)) / (2 * a));
            }
            else if (discriminant == 0) {
                rootVals.push_back(-b / (2 * a));
            }
            rootsAreValid = true;
        }
        return rootVals;        
    }
private:
    mutable bool rootsAreValid = false;
    mutable RootsType rootVals;
}
```
В таком случае, хоть метод и константный, но у нас есть mutable объекты класса, которые мы изменяем лишь на первом вызове метода roots.

Представим ситуацию, при которой пара потоков работает с объектом этого класса и решают одновременно вычислить его корни, так как метод roots объявлен константным - он должен быть потокобезопасным, выполнять операцию чтения, так как никаких изменений с инвариантом объекта не происходит.
Оба потока могут зайти в roots на моменте когда rootsAreValid все еще false, после этого мы получим больше корней в rootVals, так как оба потока будут записывать туда корни уравнений.
В данном случае лучше использовать std::mutex, а не atomic, по простой причине что мы не можем обеспечить атомарность сразу двух операций, проверки rootsAreValid а затем и его записи:
```cpp
class Polynomial {
public:
    using RootsType = std::vector<double>;

    Polynomial(double a, double b, double c) : a(a), b(b), c(c) {}
    RootsType roots() const {
        std::lock_guard<std::mutex> lg(mx);
        if (!rootsAreValid) {
            double discriminant = b * b - 4 * a * c;

            if (discriminant > 0) {
                rootVals.push_back((-b + std::sqrt(discriminant)) / (2 * a));
                rootVals.push_back((-b +- std::sqrt(discriminant)) / (2 * a));
            }
            else if (discriminant == 0) {
                rootVals.push_back(-b / (2 * a));
            }
            rootsAreValid = true;
        }
        return rootVals;        
    }
private:
    mutable bool rootsAreValid = false;
    mutable RootsType rootVals;
    mutable std::mutex mx;
}
```
Рассмотрим другой пример:
```cpp
class A {
public:
    int getValue() const {
        if (cacheValid)
            return cachedValue;

        int val1 = expensiveComputation1();
        int vla2 = expensiveComputation2();
        int newValue = val1 + val2;

        cachedValue = newValue;
        cacheValid = true;

        return newValue;
    }
private:
    int expensiveComputation1() const {
        return 42;
    }
    int expensiveComputation2() const {
        return 24;
    }
    mutable std::atomic<bool> cacheValid{false};
    mutable std::atomic<int> cachedValue{0};
}
```
Давайте снова представим что два потока пытаются получить getValue, и оба потока видят что cacheValid будет false и соответственно оба потока вызывают expensiveComputation1/2, можем попробовать изменить последовательность:
```cpp
class A {
public:
    int getValue() const {
        if (cacheValid)
            return cachedValue;
        cacheValid = true; // Переместили сюда
        
        int val1 = expensiveComputation1();
        int vla2 = expensiveComputation2();
        int newValue = val1 + val2;

        cachedValue = newValue;
        return newValue;
    }
private:
    int expensiveComputation1() const {
        return 42;
    }
    int expensiveComputation2() const {
        return 24;
    }
    mutable std::atomic<bool> cacheValid{false};
    mutable std::atomic<int> cachedValue{0};
}
```
В таком случае, если первый поток успел изменить cacheValid значение на true, он мог еще не пересчитать значение cachedValue, и тем самым второй поток получит неверное значение, вариант получился еще хуже чем предыдущий.

Таким образом, если у нас есть несколько переменных которые учавствуют в синхронизации - следует использовать мьютекс, если только одна - можно использовать std::atomic.