## HW 4.1
#### Task: Characterize the following example from the perspective of C++23

```cpp
template <class T1, class T2> struct Pair {
    template<class U1 = T1, class U2 = T2> Pair(U1&&, U2&&) {}
};

struct S { S() = default; };
struct E { explicit E() = default; };

int f(Pair<E, E>) { return 1; }
int f(Pair<S, S>) { return 2; }

assert(f({{}, {}}) == 2, ""); // Error or Assert or OK?
```

#### Solution:

Для начала посмотрим на [поведение компиляторов](https://godbolt.org/z/heTv4sWMv):

_x86-64 clang 21.1.0_:
```text
Compiler returned: 0
```

Ошибки компиляции отсутствуют, вывод программы следующий:
```text
Program returned: 0
Program stdout
2
```

_clang_ предпочел вторую перегрузку - ``int f(Pair<S, S>)``.

_x86-64 gcc 15.2_:
```text
Compiler returned: 1
Compiler stderr
<source>: In function 'int main()':
<source>:14:19: error: call of overloaded 'f(<brace-enclosed initializer list>)' is ambiguous
   14 |     std::cout << f({{}, {}}) << std::endl;
      |                  ~^~~~~~~~~~
<source>:14:19: note: there are 2 candidates
<source>:10:5: note: candidate 1: 'int f(Pair<E, E>)'
   10 | int f(Pair<E, E>) { return 1; }
      |     ^
<source>:11:5: note: candidate 2: 'int f(Pair<S, S>)'
   11 | int f(Pair<S, S>) { return 2; }
```

При этом если избавиться от ``explicit`` в конструкторе ``struct E``, то _clang_ начнёт падать с аналогичной ошибкой. 

Ещё одно наблюдение: если пометить с помощью ``explicit`` конструкторы и ``struct E``, и ``struct S``, то _clang_ выдаёт следующую ошибку компиляции:
```text
<source>:14:18: error: no matching function for call to 'f'
   14 |     std::cout << f({{}, {}}) << std::endl;
      |                  ^
<source>:10:5: note: candidate function not viable: cannot convert initializer list argument to 'Pair<E, E>'
   10 | int f(Pair<E, E>) { return 1; }
      |     ^ ~~~~~~~~~~
<source>:11:5: note: candidate function not viable: cannot convert initializer list argument to 'Pair<S, S>'
   11 | int f(Pair<S, S>) { return 2; }
      |     ^ ~~~~~~~~~~
1 error generated.>
```

Получается, для изначального кода _clang_ смог разрешить перегрузку по той причине, что кандидат ``int f(Pair<E, E>)`` не является viable, если конструктор по умолчанию для ``struct E`` является ``explicit``.

Цитаты и примеры из стандарта:
- [0] "An explicit constructor constructs objects just like non-explicit constructors, but does so only where the direct- initialization syntax or where casts are explicitly used; A default constructor can be an explicit constructor; such a constructor will be used to perform default-initialization or value-initialization." [11.4.8.2] **[class.conv.ctor]**
- [1] "The initialization that occurs in the= form of a brace-or-equal-initializer or condition (8.5), as well as in argument passing, function return, throwing an exception (14.2), handling an exception (14.4), and aggregate member initialization other than by a designated-initializer-clause (9.4.2), is called copy-initialization." [9.4.1] **[dcl.init.general]**
- [2] "List-initialization can occur in direct-initialization or copy-initialization contexts; list-initialization in a direct-initialization context is called direct-list-initialization and list-initialization in a copy-initialization context is called copy-list-initialization." [9.4.5] **[dcl.init.list]**.
- [3] "In copy-list-initialization, if an explicit constructor is chosen, the initialization is ill-formed." [12.2.2.8] **[over.match.list]**
- [4] "When objects of non-aggregate class type T are list-initialized, ... overload resolution selects the constructor in two phases: ... 2. Otherwise, or if no viable initializer-list constructor is found, overload resolution is performed again, where the candidate functions are all the constructors of the class T and the argument list consists of the elements of the initializer list." [12.2.2.8] **[over.match.list]**
- [5] "When an argument is an initializer list (9.4.5), it is not an expression and special rules apply for converting it to a parameter type. ... if the parameter is a non-aggregate class X and overload resolution per 12.2.2.8 chooses a single best constructor C of X to perform the initialization of an object of type X from the argument initializer list: If C is not an initializer-list constructor and the initializer list has a single element of type cv U ... Otherwise, the implicit conversion sequence is a user-defined conversion sequence whose second standard conversion sequence is an identity conversion." [12.2.4.2.6] **[over.ics.list]**
- [6] "A user-defined conversion sequence consists of an initial standard conversion sequence followed by a user- defined conversion (11.4.8) followed by a second standard conversion sequence. If the user-defined conversion is specified by a constructor (11.4.8.2), the initial standard conversion sequence converts the source type to the type of the first parameter of that constructor." [12.2.4.2.3] **[over.ics.user]**

- Вызов ``f({{}, {}})`` - происходит инициализация аргумента функции, что по стандарту является _copy-initialization_ [1].
- Списковая инициализация в данном контексте является _copy-list-initialization_ [2].
- По правилам перегрузки [4], [5] для ``struct Pair`` выбирается конструктор ``Pair(U1&&, U2&&)``, и вызываются конструкторы для его аргументов [6].
- Аргументы конструкторов аналогично инициализируются списками - тоже _copy-list-initialization_.
- У ``struct E`` конструктор ``explicit``, что в силу [3] означает, что ``f(Pair<E, E>)`` не является viable кандидатом. ``explicit`` конструктор может быть использован только в контексте _direct-initialization_ или при приведении [0], но не вcopy-initialization контексте.
- Такой проблемы для ``struct S`` нет.

Таким образом, правда за _clang_.
