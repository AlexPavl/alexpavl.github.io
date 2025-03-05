## Выводы типов шаблонов
С C++11 появились ключевые слова auto и decltype. Но, к сожалению, выводимые компилятором типы не всегда так очевидны.

По каким правилам и в каких случаях компилятор выводит тип для наших шаблонных функций?

```cpp
template <typename T>
void f(T param);

int main() {
    int x = 10;
    f(x); // T = int, paramType = int

    const int cx = x;
    f(cx); // T = int, paramType = int - то есть модификатор игнорируется

    const int& rx = x;
    f(rx); // T = int, paramType = int - ссылочность игнорируется
}
```
#### Два правила преобразования типов для объектов передаваемых по значению:
- Игнорируется "ссылочность" объекта
- Модификаторы const или volatile также игнорируются

Важно понимать какая именно константность игнорируется:
```cpp
template <typename T>
void f(T param) {
    *param = "a"; // Fail константность данных остается, убирается лишь константность самого объекта
    param = nullptr;
}

int main() {
    const char *const ptr = "Hello";
    f(ptr);
}
```

#### Посмотрим с ссылками, какие правила действуют тут:
- Игнорируется "ссылочность" объекта
- Модификаторы const или volatile сохраняются
```cpp
template <typename T>
void f(T& param);

int main() {
    int x = 10;
    f(x); // T = int, paramType = int&

    const int cx = x;
    f(cx); // T = const int, paramType = const int&

    const int& rx = x;
    f(rx); // T = const int, paramType = const int&
}
```

#### Посмотрим с константными ссылками, какие правила действуют тут:
- Игнорируется "ссылочность" объекта
- Модификаторы const или volatile сохраняются
```cpp
template <typename T>
void f(const T& param);

int main() {
    int x = 10;
    f(x); // T = int, paramType = const int&

    const int cx = x;
    f(cx); // T = int, paramType = const int&

    const int& rx = x;
    f(rx); // T = int, paramType = const int&
}
```


#### Теперь с указателями:
- Игнорируется "ссылочность" объекта
- Модификаторы const или volatile сохраняются
```cpp
template <typename T>
void f(T* param);

int main() {
    int x = 10;
    f(&x); // T = int, paramType = int*

    const int *cx = &x;
    f(cx); // T = const int, paramType = const int*
}
```

#### Перейдем к универсальным ссылкам
Рассмотрим такой пример:
```cpp
void g(int&& x) {

}

int main() {
    int x = 10;
    g(x); // fail
    g(std::move(x)); // ok
}
```
Чтобы принимал обычный g(x) - нам придется создать перегрузку для g(int& x);

Но есть и другой метод, было придумано понятие универсальной ссылки, которое хорошо работает с шаблонами:

```cpp
template <typename T>
void f(T&& param);

int main() {
    f(27); // T - int, paramType - int&&
    int x = 10; // lvalue
    f(x); // && & -> &    T - int&, paramType - int&

    const int cx = x;
    f(cx); // T - const int&, paramType - const int&

    const int &rx = x;
    f(rx); // T - const int&, paramType - const int&
}
```
Refference collapsing:
- T&& & -> T&
- T& && -> T& 
- T&& && -> T&
- T& & -> T&

Но refference collapsing работает только с шаблонами.

#### Аргументы-массивы
Передаем по значению
```cpp
template <typename T>
void f(T param);

int main() {
    const char name[] = "Hello";
    f(name); // T - const int*
}
```

Передаем по ссылке
```cpp
template <typename T>
void f(T& param);

int main() {
    const char name[] = "Hello";
    f(name); // T - const char[6], paramType - const char(&)[6]
}
```

#### Аргумент - указатель на функцию
```cpp
template <typename T>
void f1(T& param);

template <typename T>
void f2(T param);

void func(double, int);

int main() {
    f1(func); // void (&)(double, int)
    f2(func); // void (*)(double, int)
    // То есть логика такая же как для массивов
}
```

Какие рекомендации дает тут Скот Майерс:
- В процессе вывода типа шаблона аргументы, являющиеся ссылками, рассматриваются как ссылками не являющиеся, т.е. их "ссылочность" игнорируется
- При выводе типов для параметров, являющихся универсальными ссылками, lvalue аргументы рассматриваются специальным образом.
- При выводе типов для параметров, передаваемых по значению, аргументы, объявленные как const и/или volatile, рассматриваются как не являющиеся ни const, ни volatile.
- В процессе вывода типа шаблона аргументы, являющиеся именами массивов или функций, преобразуются в указатели, если только они не использованы для инициализации ссылок.