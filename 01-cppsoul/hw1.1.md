## HW 1.1
#### Task: justify the situation with volatile nullptr_t using the standard.
```cpp
volatile std::nullptr_t a = nullptr; int *b; b = a;
```

#### Solution:

Приведём для начала необходимые для ответа отрывки/цитаты из стандарта (для ответа на вопрос использовался рабочий черновик N4960 стандарта C++23).

\[1\] "... conforming implementations are required
to emulate (only) the observable behavior of the abstract machine ..." \[intro.abstract\]

\[2\] "The least requirements on a conforming implementation are: ... Accesses through volatile glvalues are evaluated strictly according to the rules of the abstract machine. ... These collectively are referred to as the observable behavior of the program." \[intro.abstract\]

\[3\] "A glvalue of a non-function, non-array type T can be converted to a prvalue. ... The result of the (lvalue-to-rvalue) conversion is determined according to the following rules: ... If T is cv std::nullptr_t, the result is a null pointer constant. ... Since the conversion does not access the object to which the glvalue refers, there is no side effect even if T is volatile-qualified. ..." \[conv.lval\]

\[4\] "Otherwise (T is not a class type and the object to which glvalue refers does not contain an invalid pointer value), the object indicated by the glvalue is read, and the value contained in the object is the prvalue result." \[conv.lval\]

\[5\] "Whenever a glvalue appears as an operand of an operator that expects a prvalue for that operand, the lvalue-to-rvalue, ... are applied to convert the expression to a prvalue." \[basic.lval\]

\[6\] "The lvalue-to-rvalue conversion is applied if and only if the expression is a glvalue of volatile-qualified type and it is one of the following: ... id-expression ..." \[expr.context\]

Теперь посмотрим на отрывок кода, который нас интересует.

```cpp
#include <cstddef>
#include <iostream>

int foo() {
  volatile int a = 10;
  int b = a;
  return b;
}

int *bar() {
  volatile std::nullptr_t a = nullptr;
  int *b;
  b = a;
  return b;
}
```

Для начала рассмотрим функцию ``foo()``. Результаты компиляции ``clang`` и ``gcc`` одинаковы:

```code
foo():
        mov     dword ptr [rsp - 4], 10
        mov     eax, dword ptr [rsp - 4]
        ret
```

Переменная ``a`` объявлена как ``volatile``, при присвоении переменной ``b`` значения ``a`` это значение читается (\[4\], \[5\], \[6\]), и в соответствии с требованиями \[1\] и \[2\] генерируются инструкции, отвечающие чтению из переменной ``a``.

Далее рассмотрим функцию ``bar()``. 

gcc:
```code
bar():
        mov     QWORD PTR [rsp-8], 0
        mov     rax, QWORD PTR [rsp-8]
        xor     eax, eax
        ret
```

clang:
```code
bar():
        mov     qword ptr [rsp - 8], 0
        xor     eax, eax
        ret
```

В данном случае переменная ``a`` объявлена как ``volatile nullptr_t``, и в силу замечания \[3\] при конверсии (\[5\], \[6\]) чтения из ``a`` не происходит, а потому и никаких side-effects не возникает. Это позволяет исключить операцию чтения из storage'а переменной ``a``.

Таким образом, результат компиляции ``clang`` корректен, а результат ``gcc`` наоборот является более сомнительным - если в соответствии со стандартом в данном контексте чтения значения ``a`` не происходит, то и в операции чтения нет смысла.
