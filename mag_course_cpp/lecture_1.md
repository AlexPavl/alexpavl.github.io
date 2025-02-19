# Лекция 1 (Строки) 
### Источник:
https://www.youtube.com/watch?v=9N_wJ7oIHDk&list=PL3BR09unfgcgf7R88ZQRQqWOdLy4pRW2h&index=1

Начнем с самой простой программы:

```cpp
std::cout << "Hello, world!" << std::endl;

// Какие категории грамматических конструкций языка мы тут видим:
std::endl; // - функция
std::cout; // - объект
<< // - оператор
“Hello, world!” // - литерал
```

Hello, world! это литерал, массив символов, какие еще литералы мы знаем:

0x1000, 1.6e-2, ‘c’, true, nullptr…

nullptr - это литерал, но также это ключевое слово языка с типом nullptr_t (decltype выведет такой тип, его можно получить только от nullptr).

Примеры строковых литералов(другие примеры тут https://en.cppreference.com/w/cpp/language/string_literal):

```cpp
u8”Hello” - char8_t
u”Hello” - char16_t
U”Hello” - char32_t
L”Hello” - wchar_t
std::cout << R”(h
e
l
l
o)” << std::endl; - выведется в столбик
```

Что же такое литерал? Это константа времени компиляции, которая хранится в Raw Data.

В C++ есть категории значений: lvalue, rvalue, xvalue, glvalue и prvalue (про категории можно также почитать тут https://en.cppreference.com/w/cpp/language/value_category)

Дк что же такое “Hello, world!” ? Это const char[14].

Что происходит когда мы пишем? 

```cpp
const char *decayed = "Hello, world!"; // array-to-pointer
```

У нас происходит array-to-pointer конвертация, какие еще преобразования мы знаем?

```cpp
int -> double // integral to floating
rvalue -> lvalue // самое фундаментальное преобразование
int a = 5, b; b = a + 2; // lvalue to rvalue
itn a[10], int *b; b = a; // array to pointer
int foo(int);
int (*b)(int);
b = foo; // function to function pointer
// Как вы думаете, будет ли тут какой-нибудь side effect?
volatile std::nullptr_t a = nullptr; int *b; b = a;
```

Рассмотрим пример:

```cpp
int foo() {
	volatile int a = 10;
	int b = a;
	return b;
}
```

Сколько в данном случае будет побочных эффектов? Что такое побочный эффект - это то что, грубо говоря, не может оптимизировать компилятор, в стандарте есть правило as-if-rule (правило как-если-бы) - компилятор в праве переупорядочить вашу программу, но с одним условием, он должен сохранять порядок побочных эффектов. Сколько в нашем примере побочных эффектов? 

 - запись в volatile a

 - чтение из volatile a

Имеем два побочных действия, в коде ассемблера мы увидим:

```cpp
foo():
	mov dword ptr [rsp - 4], 10 // volatile int a = 10
	mov eax, dword ptr [rsp - 4] // return a
	ret
```

Как мы видим ассемблерный код представляет из себя соптимизированные две строчки, что будет если мы сделаем так:

```cpp
int *bar() {
	volatile std::nullptr_t a = nullptr;
	int *b;
	b = a;
	return b;
}
```

Сколько здесь будет побочных эффектов и посмотрим сгенерированный ассемблер? (Если честно, хуета какая-то)

```cpp
bar(): // собирали под clang
	mov qword ptr [rsp - 8], 0
	xor eax, eax
	ret
	
bar(): // gcc
	mov QWORD PTR [rsp-8], 0
	mov rax, QWORD PTR [rsp-8]
	xor eax, eax
	ret
```

C/C++ строки исторически так сложилось заканчиваются ‘\0’. Рассмотрим следующий код:

```cpp
const char *cinv = "Hello, world"; // cinv указывает на литерал
char cmut[] = "Hello, world"; // cmut также указывает на литерал (тут скрыто копирование в стек)
char *cheap = (char*)malloc(size); // cheap указывает на кучу
// Какие проблемы с следующими строками
strcpy(cheap, cinv); // Тут идет до конца cinv (но проблем вроде нет)
cheap = cinv; // не скомпилируется, так как char* = const char*
cinv = 0; // тут никаких проблем, и нет утечки
cmut = cheap; // массив cmut не может быть lvalue (сам массив, если играть с указателем - ок)
```

Где будут храниться два одинаковых литерала “Hello, world” ? В языке это unspecified behavior, так что повторяется ли литерал - на усмотрение компилятора.

implementation-defined - все идет по маслу, разработка не противоречит пожеланиям компилятора и все по документации, программа well-written 

unspecified behavior - то, для чего не совсем полная спецификация, но при этом это не влияет на работу программы, то есть программа все еще well-written. Как пример, что вызовется раньше `func(func1(), func2());`

undefined behavior - Все остальное, программа не well-written. Она может работать нормально, а может и не оч, как пример: переполнение int . или даже использование неинициализированных переменных.

Настоящая причина проблем с С-строками это то, что длина такой строки не является инвариантом. Чтобы сохранить инварианты таких объектов как строки, необходимо закрытое состояние, недоступное к модификации, т.е. необходима инкапсуляция. 

Тут много обсуждения строк и различной работы с ними. Про тип хранения с data, size, capacity. Различные их имплементацию и тд.

Прямое сложение строк, сколько операций выделения памяти может быть в следующем коде?

```cpp
std::string a = ssl ? "https" : "http";
a = a + "://" + path + "/" + query; // потенциально в одной этой строке 4 выделения памяти
// Как это исправить? Используя std::stringstream
std::stringstream ss;
ss << (ssl ? "https" : "http") << "://" << path << "/" << query;
// Также можно с помощью введенным в 20-м стандарте format
auto fmt = std::format("{}://{}/{}", (ssl ? "https" : "http"), path, query);
```

Проблема с строками номер 2:

```cpp
static const std::string kName = "FOO"; // Проблема, что до функции main мы уже выделяем память
// .....
int foo(const std::string &arg);
// .....
foo(kName);
```

Начиная с 17-го стандарта у нас появился std::string_view, который помогает справиться с этой проблемой, так как это невладеющий указатель на RawData, если сделать так:

```cpp
static const std::string_view kName = "FOO";
int foo(const std::string_view &arg); // тут можно не передавать с таким количеством спецификаторов
// так как string_view очень легкий (он хранит только указатель и size
```

Затем список того что можно делать над string_view, можно взять отсюда (https://en.cppreference.com/w/cpp/string/basic_string_view)

Дальше про Copy On Write строки, но их нет в STL, сюда добавлять не буду. Их нет в стандарте из-за проблем с параллельностью и инвалидацией указателей (можно найти конкретнее на 59:30 на видео).

SSO (small string optimizations)

Идея в том что если строка маленькая - мы можем хранить ее на стеке(сразу включу обобщенный вариант):

```cpp
template <typename CharT> class basic_string { // тут конечно будет больше параметров
	CharT *data;
	size_t size;
	union {
		size_t capacity;
		enum {SZ = (sizeof(data) + 2 * sizeof(size_t) + 31) / 32; };
		CharT small_str[SZ];
	} sso;
public:
	// тут все его 89 методов
};
```

Чтобы сделать строку нам нужны характеристики типа, мы можем тогда вместо char, utf8… и др вставить например float. Для этого нам надо определить char_traits для нашего класса.

```cpp
Основные методы char_traits:
assign, eq, lt, move, find, eof, ...
template <typename CharT,
					typename Traits = std::char_traits<CharT>>
class basic_string {}
```

Также для строки нам понадобится аллокатор, аллокатор не выделяет память сам, он лишь бегает в ее поиске, итоговый шаблонный класс для строки получаем примерно так:

```cpp
template <typename CharT,
					typename Traits = std::char_traits<CharT>,
					typename Allocator = std::allocator<CharT>>
class basic_string {}
```