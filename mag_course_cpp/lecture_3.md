# Лекция 3 (Шаблоны классов и частичная специализация):
## Первоисточник
https://www.youtube.com/watch?v=e4NpXk9l51w&list=PL3BR09unfgcgf7R88ZQRQqWOdLy4pRW2h&index=3

### Стандартные преобразования
Процесс разрешения имен работает на цепочках неявных преобразований. Тут у нас есть стандартные преобразования:
```cpp
// Трансформации объектов (ранг точного совпадения)
int arr[10]; int *p = arr; // [conv.array]

// Коррекции квалификаторов (ранг точного совпадения)
int x; const int *px = &x; // [conv.qual]

// Продвижения (ранг продвижения)
int res = true; // [conv.prom]

// Конверсии (ранг конверсии)
float f = 1; // [conv.fpint]
```
### Пользовательские преобразования
Пользовательские преобразования ограничиваются в общем и целом конструктором с каким-то типом и оператором из этого класса:
```cpp
struct A {
    A(int); // Пользовательское преобразование int -> A
    operator int(); // (1) Пользовательское преобразование A -> int
    operator double(); // (2)
};

int i = A{}; // calls operator int
// При этом (1) лучше чем (2) потому что для него нужно меньше стандартных преобразований.
// Интуитивно: у цепочки короче хвост значит она лучше
// https://godbolt.org/z/hP1c7n
```

### Тонкости построения цепочек
Вариант 1:
```cpp
struct S { S(long) {} };
void foo(S) {}
int x = 42;
foo(x); // int -> long -> S
```
Вариант 2:
```cpp
struct T { T(int) {} };
struct S { S(T) {} };
void foo(S) {}
int x = 42;
foo(x); // int -> T -> S
```
Второй вариант работать не будет так как в цепочке неявных преобразований может быть только одно пользовательское преобразование.

### Перегрузка и шаблоны
```cpp
// Шаблон может тоже выиграть перегрузку. При этом запускается вывод типов.
void foo(double x); // 1
template <typename T> void foo(T x); // 2

foo(1); // -> несомненно 2
// Для выигравшего перегрузку шаблона запускается инстанцирование или ищется специализация.
```
Другой случай:
```cpp
void foo(double x); // 1
template <typename T> void foo(T x); // 2
template <> void foo<int>(int x); // 3
foo(1); // -> специализация не учавствует в перегрузке, выиграет 2 но потом найдет специализацию и будет вызван 3
```

### Что если вывод удался дважды?
```cpp
// Рассмотрим более сложный пример
template <typename T> void f(T);  // 1
template <typename T> void f(T*); // 2
// В точке вызова у нас нечто вроде
int ***a;
foo(a); // -> 2
// Здесь вывод работает, но шаблоны сами по себе преегружены
// Тогда внезапно вывод будет повторен дважды
```
Здесь шаблонное инстанцирование, разрешение имен и вывод типов работают вместе. У нас есть два подходящих шаблона и компилятор ничего не знает про них, он их еще не начал инстанцировать. Как решает эту проблему компилятор:

### Частичный порядок шаблонов функций
- В случае шаблонов функций, они частично упорядочены на более и менее специализированные.
- Для установления этого порядка мы должны трансформировать параметры.
```cpp
template <typename T> void f(T); // -> f(T1)
template <typename T> void f(T*); // -> f(T2*)
```
- И далее запустить вывод типов

| #template | Parameter type | Argument type |
|-----------|----------------|---------------|
| template1 | template <typename> void f(T) | f(T1) |
| template2 | template <typename> void f(T*)| f(T2*)|

T1 и Т2* это пустые структуры с уникальным именем. И Т2* является более специальным шаблоном и соответственно, когда два шаблона выигрывают перегрузку -> выбирается тот, который является более специализированным (частным).

### Результирующее отношение
```cpp
// Мы видим, что нам нужно рассмотреть два случая
// 1. compare (1) not less specialized then (2)
template <typename T> void f(T);
T2 *a; f(a); // OK, (1) >= (2)
// 2. compare (2) not less specialized then (1)
template <typename T> void f(T*);
T2 b; f(b); // FAIL, (2) < (1)
// Откуда очевидно (1) > (2)
```

### Некая двусмысленность в выводе
```cpp
template <typename T> void foo(T); // 1
template <typename T> void foo(T*); // 2
template <> void foo(int*); // 3 Чья это специализация?

int x;
foo(&x); // Вызове [3], и это в целом ок, но
```
- Вопрос является (3) специализацией для (2) или для (1) не имеет смысла. Она одинаково хорошо подходит для обоих, поэтому специализирует выигравший перегрузку шаблон.
- В связи с этим могут возникать неприятные сюрпризы.

### Контрпример Димова-Абрамса
```cpp
template <typename T> void foo(T); // 1
template <> void foo(int*); // 2
tmeplate <typename T> void foo(T*); // 3

int x;
foo(&x); // вызовет (3), хотя (2) подходит лучше
```
- Важно помнить: специализации не участвуют в перегрузке. Сначала разрешается перегрузка, потом ищется наименее общая специализация.
- В данном случае (2) не специализирует (3), так как встречается раньше.
- В целом это аргумент против специализации.

Контрольный вопрос на размышление:
```cpp
template <typename T, typename U> void foo(T, U); // 1
template <typename T, typename U> void foo(T*, U*); // 2
template <> void foo<int*, int*>(int*, int*); // 3 чья это специализация?

int x;
foo(&x, &x); // Что вызовется тут?
```

Вызовется (2), а (3) не является специализацией 2.

## Шаблоны классов
### Точки объявления (PoD)
Точка объявления - это место в котором компилятору становится известно имя.
Ниже приведены два фрагмента кода (не пишите так!):
```cpp
int y = 2; { int y /* PoD */ = y; }
int x = 2; { int x[x] /* PoD */; }
// https://godbolt.org/z/5Tx4vExs1
```
Оба варианта будут работать, в первом варианте мы получим мусор в y, во втором варианте мы получим массив из двух элементов.
- Оба сомнительны, но в одном случае будет использовано значение переменной из внешней области видимости, а в другом случае нет.
- Точка объявления [basic.scope.pdecl] это точка в которой объявление завершено (после полного объявления но перед инициализацией)
- До точки объявления имя не считается введенным в область видимости (соответственно используется имя из прошлой области видимости). - поэтому мы и получаем массив из 2 элементов а 'y' берется уже из текущей области видимости и соответстенно получаем мусор.

TODO: Изучить VLA - что это такое?

### Шаблоны классов
- Обобщенный тип, задаваемый шаблоном считается объявленным в PoD.
```cpp
template <typename T> struct fwnode /* PoD */ {
    T data_;
    fwnode<T> *next_;
};
```
- В теле таким образом может быть использован указатель или ссылка на себя (как на неполный тип).
- Для удобства, шаблонные параметры рядом с именем может не указывать (только внутри тела класса).

### Зависимые имена внутри шаблонов
- Нет никаких проблем (как с функциями) в использовании вторичной типизации.
```cpp
template <typename T> class Stack {
    fwnode<T> *top_;
    // Все остальное в стеке для объектов типа Т общего вида
};
```
- Такой стек (построенный на односвязном списке, что вполне реалистично) использует fwnode. Параметр обязателен.

### Порождение классов из шаблонов
```cpp
template <typename T> class Stack {
    fwnode<T> *top_;
public:
    push(T x);
};

// class Stack<int> {
//   fwnode<int> *top_;
// public:
//   push(int x);
//};
// /\\ вверху идет инстанцирование шаблона
Stack<int> s;
```
Проблема: слишком общие шаблоны, для целых чисел было бы проще хранить не связный список для стека а в массиве:
```cpp
template <> class Stack<int> {
    int *content_;
    // Остальной функционал
}
```

- Как и для функций ручная специализация уже инстанцированного типа не имеет смысла.
```cpp
template <typename T> class Invalid{};
// тут уже произошло инстанцирование
Invalid<int> i;
// Эта специализация не имеет смысла и запрещена
template <> class Invalid<int>{};
```
- Опасайтесь специализаций, происходящих внутри единицы трансляции. Единственный разумный способ создавать шаблоны и специализации к ним это в хэдере. 

### Частичная специализация
- Кажется этот стек не слишком эффективен для хранения всех указателей
```cpp
template <typename T> class Stack {
    fwnode<T> *top_;
    // Остальное
};
```
- Для указателей было бы проще хранить массив указателей:
```cpp
template <typename T> class Stack<T*> {
    T** content_;
    // Остальное
};
```

### Упрощение имен в специализации
- Рассмотренное ранее упрощение имен отлично работает в (частичных) специализациях.
```cpp
template <typename T> class A {
    A* a1; // A здесь означает A<T>
};

template <typename T> class A<T*> {
    A* a2; // A здесь означает A<T*>
};
```
- Разумеется это опционально. Указывать полные имена - вполне легально (и часто отличная идея).

### Примеры частичной специализации
```cpp
template <typename T, typename U> class Foo{}; // 1
template <typename T> class Foo<T, T>{};  // 2
template <typename T> class Foo<T, int>{}; // 3
template <typename T, typename U> class Foo<T*, U*>{}; // 4

Foo<int, float> mif; // 1
Foo<float, float> mff; // 2
Foo<float, int> mfi; // 3
Foo<int*, float*> mp; // 4
// https://godbolt.org/z/rxedevMv6
```

### Немного расширения сознания
```cpp
template <typename T> struct vector {};

template <typename T> struct X {
    void print() { std::cout << "forall" << std::endl; }
};
// Это тоже частичная специализация, но тут vector это не std::vector, так как у вектора два параметра и при частичной специализации параметр по умолчанию работать не будет
template <typename T> struct X<vector<T>> {
    void print() { std::cout << "forvec" << std::endl; }
};
X<int> a;
X<vector<int>> b;

a.print();
b.print();
// https://godbolt.org/z/fGsceo69d
```

У частичной специализации как и у специализации есть ограничения:
- Специализации следуют за primary template
- Частично специализированный шаблон должен быть действительно менее общим, чем тот, версией которого он является

```cpp
template <typename T> class X { /* */ };
template <typename U> class X<U> { /**/ }; // Ошибка
```

### Менее специальна, менее обобщена?
- Частичная специализация съедает какое-то количество параметров primary шаблона.
- До тех пор пока она менее специальна в некоем смысле, у нее может быть даже больше шаблонных параметров.
```cpp
template <typename T> class X;
template <typename R, typename Arg> class X<R(Arg)>{} // std::function реализован именно так
```

### Ограничения специализации
- Специализация всегда должна в коде следовать за объявлением шаблона общего вида.
- Специализированный шаблон должен быть действительно менее общим, чем тот, версией которого он является.
- Полная специализация возможна и для классов и для функций, наряду с перегрузкой.
- Частичная специализация для функций невозможна.
```cpp
template <typename T> void foo(T x);
template <> void foo<int> (int x); // ok
template <typename T> void foo<T*> (T* x); // fail!!!
```
- Но мы можем сэмитировать частичную специализацию функцию через статический метод частично специализированного класса. 

### Трюк Саттера
- Частичная специализация функций запрещена.
- Можно ли сымитировать частичную специализацию функций через частичную специализацию классов?
```cpp
template <typename T> struct FImpl;
template <typename T> void f(T t) { FImpl<T>::f(t); }
template <typename T> struct FImpl {
    static void f(T t);
};
// https://godbolt.org/z/q89PTjn1c
```
- Здесь используется то, что статический метод stateless класса мало отличается от свободной функции.

### Специализация и LSP
LSP - Liskov Substitution Principle
- Можно ли шаблонную специализацию назвать разновидностью наследования?
- Увы нет. С точки зрения наследования это нарушение LSP.
```cpp
template <typename T> struct S { void foo(); };
template <> struct S<int> { void bar(); };
S<double> sd; sd.foo();
S<int> si; si.bar();
```
- Специализация может не иметь ничего общего с полной версией

## Специализация методов
### Шаблоны членов: простая задача
```cpp
struct Data {
    template <typename T> T read() const;
};
template <typename T> class DataReader {
    const T& source_;
public:
    DataReader(const T& s) : source_(s) {}
    template <typename R> R read(); // вызывает source_.read();
};
// Попробуйте написать определение DataReader::read
```

Ниже спойлер:
- Первый вариант решения
```cpp
template <typename T>
template <typename R>
R DataReader<T>::read() {
    // R res = source_.read<R>(); -> работать не будет, синтаксическая неоднозначность
    R res = source_.template read<R>();
    return res;
}
```

Как написать специализацию для DataReader<T>::read<string>?

Следующий вариант работать не будет, так как мы получаем специализацию функции а не метода.
```cpp
tmeplate <typename T> template<>
string DataReader<T>::read<string>() const
{
    string foo = source_.template read<string>();
    return foo;
}
```
Чтобы сделать специализацию под конкретный тип надо делать полную специализацию:
```cpp
struct Data { template <typename T> T read() const; };
template <typename T> class DataReader {
    const T& source_;
public:
    template <typename R> R read();
};

template <>
template <> // это не опечатка!
string DataReader<Data>::read<string>() {
    return source_.template read<string>();
}
// https://godbolt.org/z/W1rzo5M53
```

Задача: параметризация методов
```cpp
template <typename T1, typename T2> struct A {
    void func();
};
// Необходимо добиться следующего эффекта:
A <int, double> a;
A <float, double> b;
a.func(); // for int
b.func(); // for all
```
- То есть параметризовать метод первым аргументом шаблона
- Задачу усложняет то, что частичная специализация для методов невозможна.

Параметризация: первая попытка
```cpp
template <typename T1, typename T2> struct A  {
    void func() {
        T1 dummy; internal_func(dummy); // Проблема что мы создаем непредсказуемый объект
    }
private:
    template <typename V> void internal_func(V) { cout << "all\n"; }
    void internal_func(int) { cout << "int\n"; }
};
A<int, double> a;
A<float, double> b;

a.func(); // for int, благодаря разрешению перегрузки
b.func(); // for all
```

Проблема с инициализацией dummy внутри, мы не знаем можем ли мы создать этот объект и дополнительное выделение памяти, как можно решить эту проблему? 

Переходники типов:
```cpp
template <typename T> struct Type2Type {
    typedef T OriginalType;
};
```
- Минимальный размер.
- Прозрачность.
- Номинативная типизация.

Параметризация: переходники типов
```cpp
template <typename T1, typename T2> struct A {
    void func() {
        internal_func(Type2Type<T1>());
    }
private:
    template <typename V> void internal_func(V) { cout << "all\n"; }
    void internal_func(Type2Type<int>) { cout << "int\n"; }
};

A <int, double> a;
A <float, double> b;

a.func(); // for int
b.func();  // for all
```

- Переходники Type2Type изначально были придуманы Александреску для еще одной имитации частичной специализации функций.
- Он рассматривал шаблонную функцию-"конструктор"
```cpp
template <typename T, typename U> T* Create(const U& arg);
```
- И решал задачу специализации его по первому аргументу. Как-то вот так:
```cpp
template <typename U>
Widget* Create<Widget, U>(const U& arg); // not C++
```
- Разумеется такое тупое решение не сработает
- Как здесь помогут переходники типов?

- Решение просто и впечатляюще:
```cpp
template <typename T, typename U>
T* Create(const U& arg, Type2Type<T>);
template <typename U>
Widget* Create(const U& arg, Type2Type<Widget>);
```
- Красиво, хотя и накладывает обязательства.

## Вывод типов
### Вывод конструкторами классов
- Начиная с С++17 конструкторы классов могут использоваться для вывода типов:
```cpp
template <typename T> struct container {
    container(T t);
    // и так далее
};

container c(7); // -> container<int> c(7);
auto c = container(7); // -> аналогично
auto c = new container(7); // -> аналогично
```

Проблема:
- Конструтор класса сам может быть шаблонным
```cpp
template <class T> struct container {
    template <class Iter> container(Iter beg, Iter end);
    // и так далее
};
vector<double> v;
auto d = container(v.begin(), v.end()); // container<double>???
```
- Сейчас это приведет к ошибке
note: template argument deduction/substitution failed:
note: couldn't deduce template parameter 'T'
    auto d = container(v.begin(), v.end());

Как мы можем помочь компилятору? 
### Хинты для вывода (С++17)
- Также пользователь может помочь выводу в сложных случаях
```cpp
template <class T> struct container {
    template <class Iter> container(Iter beg, Iter end);
    // и так далее
};
// Пользовательский хинт для вывода
template <class Iter> container(Iter b, Iter e) -> container<typename iterator_traits<Iter>::value_type>;

vector<double> v;
auto d = container(v.begin(), v.end()); // -> container<double>
```

### Вывод без конструктора
- Агрегатное значение может и не иметь конструктора.
```cpp
template <typename T>
struct NamedValue {
    T value;
    string name;
};
```
- Здесь вывод не сработает без специальных мер.
```cpp
NamedValue n{"hello", "world"}; // FAIL
```
- Но мы можем помочь компилятору.
```cpp
NamedValue(const char*, const char*) -> NamedValue<string>;
NamedValue n{"hello", "world"}; // OK
```

### Неявные хинты
- В случае если конструктора нет, мы должны писать явный хинт.
```cpp
template <typename T> struct Value { T value; };
template <typename T> Value(T) -> Value<T>; // explicit
// Но если конструтор есть, то хинт будет неявным(т.е. сгенерированным).
template <typename T> struct Value {
    T value_;
    Value(T val) : value_(val) {}
};

// template <typename T> Value(T) -> Value<T>; implicit
// template <typename T> Value(Value<T>) -> Value<T>; // implicit (copy constructor)
```