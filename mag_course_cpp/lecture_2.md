# Лекция 2 Шаблоны функций
### Источник
https://www.youtube.com/watch?v=FshTrPe_Woc&list=PL3BR09unfgcgf7R88ZQRQqWOdLy4pRW2h&index=2

Задача: возвести число в степень, начнем с первого
```cpp
unsigned nth_power(unsigned x, unsigned n) {
    unsigned acc = 1;
    if ((x < 2) || (n == 1)) return x;
    while (n > 0) {
        if ((n & 0x1) == 0x1) { acc *= x; n -= 1; }
        else { x *= x; n /= 2; }
    }
    return acc;
}
// https://godbolt.org/z/TThqcove7
```
Что такое полиморфная функция? Полиморфная функция - та функция, которая ведет себя по разному с разным типом аргументов. Шаблон функции - статический полиморфизм, как и перегруженные функции с одним именем но разным типом аргументов.
Давайте сделаем ее статически полиморфной, какие проблемы с ним видим?
```cpp
template <typename T> T nth_power(T x, unsigned n) {
    T acc = 1; // Т может быть абсолютно разным и может не иметь неявного преобразования из 1
    if ((x < 2) || (n == 1)) return x; // сравнение с 2, та же проблема что с acc = 1
    while (n > 0) {
        if ((n & 0x1) == 0x1) { acc *= x; n -= 1; }
        else { x *= x; n /= 2; }
    }
    return acc;
}
// https://godbolt.org/z/MrTfTaM3T
```
Попробуем разобраться с этой проблемой:
```cpp
template <typename T> struct default_id_trait {
    static int id() { return 1; } // Параметр по умолчанию
};

template <> struct default_id_trait<Matrix2x2> { // Пример специализации дефолтного значения для матрицы
    static Matrix2x2 id() { return {1, 0, 0, 1}; }
}; // Явное инстанцирование структуры

template <typename T, typename Trait = default_id_trait<T>>
T nth_power(T x, unsigned n) {
    T acc = Trait::id();
    if ((x == acc) || (n == 1)) return x;
    while (n > 0) {
        if ((n & 0x1) == 0x1) { acc *= x; n -= 1; }
        else { x *= x; n /= 2; }
    }
    return acc;
}
// https://godbolt.org/z/s1Kjqj7of
// https://godbolt.org/z/nPdEGrWv3
```
Но так не делают в STL, там делают иначе:
```cpp
template <typename T> concept multiplicative = requires(T t) {
    { t *= t } -> std::convertible_to<T>; // concept for *=
};

// Чистая часть
template <typename T>
T do_nth_power(T x, T acc, unsigned n)
requires multiplicative<T> && std::copyable<T> // Проверяем что операции *= и копирования возможны с помощью концептов
{
    while (n > 0)
        if ((n & 0x1) == 0x1) { acc *= x; n -= 1; }
        else { x *= x; n /= 2; }
    return acc;
}
// Перегрузка для unsigned
unsigned nth_power(unsigned x, unsigned n) {
    if (x < 2u || n == 1u) return x;
    return do_nth_power<unsigned>(x, 1u, n); // тут НЕЯВНО инстанцируем функцию
// https://godbolt.org/z/e3fPToG88
}
```
### Class vs typename
Во многих местах шаблонный параметр написан как typename
```cpp
template <typename T> int foo(T x);
```
Во многиз других как class
```cpp
template <class T> int foo(T x);
```
Особой разницы нет. Раньше class использовался чтобы подчеркнуть, что там ожидается нетривиальный объект, но это уже никому не нужно.
Предпочтительно (если мы знаем что ожидать) писать концепт.
```cpp
template <std::integral T> int foo(T x);
// https://godbolt.org/z/s4vrW5583
```

Инстанцирование - это процесс порождения экземпляра специализации.
```cpp
template <typename T>
T max(T x, T y) { return x > y ? x : y; }
....
max<int>(2, 3) // неявное инстанцирование
```
Можем ли мы вмешаться в процесс инстанцирования?
Перед тем как мы об этом поговорим, у компилятора есть 3 процесса:
- Процесс инстанцирования
- Процесс вывода типов
- Процесс разрешения имен

Как вмешаться в инстанцирование?
```cpp
extern template int max<int>(int, int); // Для запрета инстанцирования в единице трансляции
template int max<int>(int, int); // Для насильного вызова инстанцирования, не важно где дальше будет вызван сам шаблон

// Также можно проводить явную специализацию
template <typename T> T max(T x, T y) { .... }
template <> int max<int>(int x, int y) { .... }
template <> float max(float x, float y) { .... }
// Специализация обязана физически следовать за основным шаблоном 
template <> int min<int>(int x, int y) { .... } // ошибка (обобщенный ниже)
template <typename T> T min(T x, T y) { .... } // обобщенный тут
```

### Правила игры
Общее правило для функций и не только (temp.spec.general)
- Явное инстанцирование единожды в программе
- Явная специализация единожды в программе
- Явное инстанцирование должно следовать за явной специализацией
```cpp
template <typename T> T max(T x, T y) { .... } // Обобщенный шаблон
template <> int max<int>(int x, int y) { .... } // Явная специализация
template int max<int>(int, int); // Явное инстанцирование
// https://godbolt.org/z/s43xoeYqd
```
Нарушение этих правил влечет за собой IFNDR (Ill formed no diagnostic requered).
IFNDR обозначает что никто не знает чем ваша программа является. Ill formed программа это любая ошибка и компилятор диагностирует и находит программу.
При IFNDR компилятор не запускает диагностику и программа будет запускаться но может с странной ошибкой.

Явная специализация может войти в конфликт с инстанцированием
```cpp
template <typename T> T max(T x, T y);

// Ок, указываем явную специализацию
template <> double max(double x, double y) { return 42.0; }
// никакой implicit instantiation не нужно
int foo() { return max<double>(2.0, 3.0); }

// процесс implicit instantiation нужен и он произошел
int bar() { return max<int>(2, 3); }

// ошибка: мы уже породили эту специализацию
template <> int max(int x, int y) { return 42; }
// https://godbolt.org/z/Y8zc7xdsd
```

Удаление специализаций
```cpp
// для всех указателей
template <typename T> void foo(T*);

// но не для char* и не для void*
template <> void foo<char>(char*) = delete;
template <> void foo<void>(void*) = delete;
// https://godbolt.org/z/o5WThaoxY
```
Обратите внимание - запрет специализации это специализация!
Как вы думаете, что произойдет если мы сначала сгенерируем специализацию, а потом запретим ее?

### Non-type шаблонные параметры параметры
Параметры не являющиеся типами могут быть структурными типами для шаблона
Структурные типы это: 
- Скалярные типы (до 20-го стандарта кроме плавающей точки)
- Левые ссылки
- Структуры у которых все поля и базовые классы public и не mutable. И при этом все поля и поля всех базовых классов тоже структурные типы или массивы.

```cpp
struct Pair { int x, y; }; // структурный тип
template <int N, int *PN, int &RN, Pair P> int foo();
// Базовая интуиция, все должно быть compile-time known.
// https://godbolt.org/z/399f8eYbK
```
Какими указателями и ссылками можно параметризовать шаблоны:
- nullptr
- На глобальную память
- На статические объекты

Более детальный пример:
```cpp
struct Pair {
    int x = 1, y = 1;
};

template <int N, int *PN, int &RN, Pair P> int foo() {
    return N + *PN + RN + P.x + P.y;
}

constexpr Pair p; // для шаблонов можем использовать только constexpr
int x = 1;
int y = 1;

TEST(nontypes, foo) {
    int res = foo<2, &x, y, p>(); // 2 - int, &x - *PN, y - &RN, p - constexpr Pair
    EXPECT_EQ(res, 2 + 1 + 1 + 1 + 1);
}

// Рассмотрим более жесткий тип
struct MoreComplex : public Pair {
    int arr[3] = {1, 2 ,3};
    int z = 1;
};

template <int N, int *PN, int &RN, MoreComplex M> int buz() {
    return N + *PN + RN + M.z + M.arr[0];
}
// Для каждого MoreComplex будет порождение отдельного инстанца buz
```

### Специализация по nontype параметрам
Нет никаких проблем в том, чтобы специализировать класс по нетиповым параметрам.
```cpp
// Почему тут используется ссылка на массив? Для того чтобы сохранить информацию о размере
template <typename T, int N> foo(T(&Arr)[N]);
template <> foo<int ,3>(int(&Arr)[3]) {
    // Тут более эффективная реализация для трех целых
// https://godbolt.org/z/8jxPjn39f
}
```
Обратите внимание, при явной специализации функций вы обязаны указать все параметры.
Как вы видите себе специализацию по указателям, ссылкам и структурным типам?
Массив в шаблонном параметре редуцируется до указателя (как в функции).

### Шаблонные шаблонные параметры
Параметрами могут быть шаблоны классов
```cpp
template <template<typename> typename Cont, typename Elt>
void print_size(const Cont<Elt> &a);
// Разумеется, специализация по ним тоже возможна
// Пока что кажется, что это переусложнение.
template <typename Container>
void print_size(const Container &a);
// Это работает не хуже (а собственно лучше, например для vector<int>).
// Мы вернемся к этому в Лекции 3 при разговоре о шаблонах классов.
// https://godbolt.org/z/f7MaYfvv1
```

### Вывод типов до подстановки
Для параметров являющихся типами, работает вывод типов
```cpp
int max = max(1, 2); // -> int max<int>(int, int);
// При выводе режутся ссылки и внешние cv-квалификаторы
const int& a = 1;
const int& b = 2;
int x = max(a, b); // -> int max<int>(int, int);
// Вывод не работает, если он не однозначен
unsigned x = 5; do_nth_power(x, 2, n); // FAIL
int a = 1; float b = 1.0; max(a, b); // FAIL
```

### Вывод типов после подстановки
Вывод типов внутри шаблонной функции дает точки вывода, где разрешить тип можно только подстановки
```cpp
template <typename T> T max(T x, T y) { ..... }
template <typename T> T min(T x, T y) { ..... }
template <typename T> bool
test_minmax(const T &x, const T &y) {
    if (x > y) return test_minmax(y, x);
    return min(x, y) == x && max(x, y) == y;
}
// Таким образом вывод и подстановка включаются попеременно.
```

### Вывод уточненных типов
Иногда шаблонный тип аргумента может быть уточнен ссылкой или указателем и cv-квалификатором.
```cpp
template <typename T> T max(const T& x, const T& y);
// В этом случае выведенный тип тоже будет уточнен
int a = max(1 ,3); // -> int max<int>(const int&, const int&);
// Уточненный вывод иначе работаент с типами: он сохраняет cv-квалификаторы
template <typename T> void foo(T& x);
const int &a = 3;
int b = foo(a); // -> void foo<const int>(const int& x);
// https://godbolt.org/z/rK9W4hbh8
```

### Вывод еще более уточненных типов
Вывод типов работает шире, чем люди обычно думают
```cpp
template <typename T> int foo(T(*p)(T));
int bar(int);
foo(bar); // -> int foo<int>(int(*)(int));
// Могут быть выведены даже параметры, являющиеся константами
template <typename T, int N> void buz(T const(&)[N]);
buz({1, 2, 3}); // -> void buz<int, 3>(int const(&)[3]);
// Общее правило: вывод типов матчит сложные композитные типы
// https://godbolt.org/z/6bdrr7d7z
```

### Частичный вывод типов
В некоторых случаях у нас просто нет контекста вывода
```cpp
template <typename DstT, typename SrcT>
DstT implicit_cast(SrcT const& x) {
    return x;
}
double value = implicit_cast(-1); // fail! Не знает что результат будет double
// Тогда мы можем указать необходимое и положиться на вывод остального
double value = implicit_cast<double, int>(-1); // ok
double value = implicit_cast<double>(-1); // ok
// https://godbolt.org/z/hY59f4hbe
```

### Параметры по умолчанию
Допустим у вас есть функция, берущая по умолчанию плавающее число.
```cpp
template <typename T> void foo(T x = 1.0);
// Увы, если его не указать, вывод типов работать не будет
foo(1); // ok, foo<int>(1);
foo<int>(); // ok, foo<int>(1.0 narrowed to int);
foo(); // fail
```
Тем не менее, ситуацию можно исправить.
```cpp
template <typename T = double> void foo(T x = 1.0);
// тогда
foo(); // ok
// https://godbolt.org/z/j4PW7j66s
```
Начиная с 20-го стандарта мы можем выводить типы для non-type параметров. 
Что делать компилятору если в шаблоне вывод типов требуется в списке параметров?
```cpp
template <auto n> int foo() { .... }
foo<1>();
foo<1.5>();
// В этом случае компилятор выводит тип построением invented expression.
auto n = 1;
auto n = 1.5;
// https://godbolt.org/z/bGWaPxb1P
```

### Вывод специализирующего типа
Очень интересной техникой являетя оставить специализирующий тип выводу типов.
```cpp
template <typename T> T foo(T x) { code for all }
// Ниже не требуется специализировать шаблон, так как по параметру определяется
template <> int foo(int x) { code for int } // -> foo<int>
// Это удобно и это часто применяетя. Но иногда сложно понять по чему специализируем
template <>
inline cl_int c1::Program::getInfo(c1_program_info name,
    vector<vector<unsigned char>>* param) const {}
// По чему специализироваться будет тут?
// https://godbolt.org/z/c9E968KP4
```

Что является единицей перегрузки в языке С++?
foo(s);
Как может быть истолкована строчка выше?
- Функция с аргументом s
- Конструктор типа foo
- Дефолт конструирование (конструирование без аргументов)
https://godbolt.org/z/WT51drTsq

### Общий обзор правил перегрузки
- Выбирается множество перегруженных имен
- Выбирается множество кандидатов
- Из множества кандидатов выбираются жизнеспособные (viable) кандидаты для данной перегрузки
- Лучший из жизнеспособных кандидатов выбирается на основании цепочек неявных преобразований для каждого параметра
- Если лучший кандидат существует и является единственным, то перегрузка разрешена успешно, иначе программа ill-formed.

### Спрятанные имена
Вопрос для новичка: что на экране?
```cpp
struct B {
    void f(int) { std::cout << "B" << std::endl; }
};
struct D : B {
    void f(const char*) { std::cout << "D" << std::endl; }
};
int main() {
    D d; d.f(0); // выведет D, так как именно он и виден с скоупа структуры D
}
// https://godbolt.org/z/jf66q65jM
```

Предыдущий пример может показаться странным, но давайте его упростим.

```cpp
void f(int) { std::cout << "B" << std::endl; }
void f(const char*) { std::cout << "D" << std::endl; }

int main() {
    extern void f(const char*);
    f(0); // Выведет D - так как если в текущем скоупе находим нужную функцию то ее и вызываем и не ищем дальше
}
```

### Проблема: операторы
Обычно оператор может находиться в любом пространстве имен.
```cpp
std::cout << "Hello!\n";
// Это вполне может быть эквивалентно следующему
operator<< (std::cout, "Hello!\n"); // Если в скоупе нет оператора подходящего то срабатывает ADL и мы смотрим в пространстве имен аргументов, в данном случае std
// Чтобы это работало, это должен быть оператор из пространства имен std
std::operator<< (std::cout, "Hello!\n");
// Но компилятор не может об этом догадаться из записи std::a << b
```
Решение: поиск Кенига
Эндрю Кениг предложил решение в начале 90-х
- Компилятор ищет имя функции из текущего и всех охватывающих пространств имен
- Если оно не найдено, компилятор ищет ия функции в пространствах имен ее аргументов

```cpp
namespace N { struct A; int f(A*); }
int g(N::A *a) { int i = f(a); return i; } // Тут найдет правильный метод
```

Поиск Кенига и шаблоны
Следующий пример не сработает:
```cpp
namespace N {
    struct A;
    template <typename T> int f(A*);
}
int g(N::A *a) {
    int i = f<int>(a); // FAIL думает что f< это оператор меньше
    return i;
}

// Чтобы пофиксить
namespace N {
    struct A;
    template <typename T> int f(A*);
}
template <typename T> void f(int); // неважно какой параметр, нам надо лишь ввести имя f в пространство имен шаблона
int g(N::A *a) {
    int i = f<int>(a); // FAIL думает что f< это оператор меньше
    return i;
}
```

### Идея построения цепочки неявных преобразований
С наивной точки зрения в цепочку преобразований входят:
- С высшим приоритетом: стандартные преобразования
- Немного ниже: пользовательские преобразования
- С низшим приоритетом: троеточия

Сложности начинаются когда комбинируются много разных
```cpp
struct S { S(int){} };
void foo(int); // 1
void foo(S); // 2
void foo(...); // 3

foo(1); // -> 1
// https://godbolt.org/z/1eafvcYW6
```

### Стандартные преобразования
```cpp
// Трансформации объектов (ранг точного совпадения)
int arr[10]; int *p = arr; // [conv.array]
// Коррекция квалификаторов (ранг точного совпадения)
int x; const int *px = &x; // [conv.qual]
// Продвижения (ранг продвижения)
int res = true; // [conv.prom]
// Конверсии (ранг конверсии)
float f = 1; // [conv.fpint]
```

TODO - добавить таблице Conversions из стандарта [tab.over.ics.scs]

### Пользовательские преобразования
Задаются implicit конструктором либо оператором преобразования.
```cpp
struct A {
    operator int(); // 1
    operator double(); // 2
};
int i = A{}; // calls (1)
// При этом (1) лучше чем (2) потому что для него нужно меньше стандартных преобразования
// Интуитивно: у цепочки короче хвост значит она лучше

// https://godbolt.org/z/hP1c7n
```

### Тонкости построения цепочек
В любой цепочке преобразований может быть сколько угодно стандартных преобразований и только одно пользовательское.

```cpp
struct S { S(long){} };
void foo(S) {}
int x = 42;
foo(x); // int -> long -> S

// another
struct T { T(int){} };
struct S { S(T){} };
void foo(S){}
int x = 42;
foo(x); // int -> T -> S не скомпилируется так как больше 1 пользовательского преобразования
```

### Перегрузка и шаблоны
Шаблон может выиграть перегрузку. При этом запускается вывод типов.
```cpp
void foo(double x); // 1
template <typename T> void foo(T x); // 2
foo(1); // Несомненно 2
// Для выигравшего перегрузку шаблона запускается инстанцирование или ищется специализация
template <> void foo<int>(int x); // 3
foo(1); // 
```