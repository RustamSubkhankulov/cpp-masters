## HW 1.3
#### Task: find how to write proper protection code for
```cpp
int foo(int *a, int base, int off) { return a[base + off]; }
```

#### Solution:

Плохой код:
```cpp
int bar(int*a, int base, int off) {
    if (off > 0 && base > base + off) {
        return 42;
    }
    return a[base + off];
}
```

Результат компиляции x86-64 clang 21.1.0 с флагами ``-O2 --std=c++23``:
```code
bar(int*, int, int):
        add     esi, edx
        movsxd  rax, esi
        mov     eax, dword ptr [rdi + 4*rax]
        ret
```

Код проверки перепишем исходя из предположения, что (это не описано в задании, но является естественным предположением):
  - ``base`` - положительное значение, базовый индекс в массиве
  - ``off`` - смещение относительного базового индекса, которое может принимать отрицательные значения

В данных предположениях переполнение, которое необходимо проверить - это переполнение суммы сверху.

Требуется проверить две потенциальные ошибочные ситуации:
  - ``off`` отрицателен и по модулю больше ``base``, что приводит к обращению за границу массива
  - Сумма ``base`` и ``off`` переполняется.

```cpp
int foo(int*a, int base, int off) {
  if (base < 0) {
    std::print(std::cerr, "Negative base\n");
    return 42;
  }

  if (off < 0 && std::abs(off) > base) {
    std::print(std::cerr, "Out-of-range access by negative index\n");
    return 42;
  }

  if (off > 0 && base > std::numeric_limits<int>::max() - off) {
    std::print(std::cerr, "Signed integer overflow\n");
    return 42;
  }
  return a[base + off];
}
```

Результат компиляции x86-64 clang 21.1.0 с флагами ``-O2 --std=c++23``: https://godbolt.org/z/rK8hjYP6o

Компиляция с флагом ``-fsanitize=undefined`` и запуск с аргументами, потенциально вызывающими переполнение:

```cpp
#include <cmath>
#include <limits>
#include <iostream>
#include <print>

int foo(int*a, int base, int off) {
  /* ... */
}

int main() {
  constexpr auto max = std::numeric_limits<int>::max();
  auto a = new int[max];
  std::fill(a, a + max, 0);

  std::cout << foo(a, -10, 0) << std::endl;     // '42'
  std::cout << foo(a, max, max) << std::endl;   // '42'
  std::cout << foo(a, 50, max-60) << std::endl; // '0'
  std::cout << foo(a, 50, max-51) << std::endl; // '0'
  std::cout << foo(a, 50, max-49) << std::endl; // '42

  delete[] a;
  return 0;
}
```

Out:
```text
Negative base
42
Out-of-range access by negative index
42
Signed integer overflow
42
0
0
Signed integer overflow
42
```

Избавимся от выводов сообщений об ошибках, чтобы лучше рассмотреть ассемблер:
```cpp
int foo(int*a, int base, int off) {
  if (base < 0) {
    return 42;
  }

  if (off < 0 && std::abs(off) > base) {
    return 42;
  }

  if (off > 0 && base > std::numeric_limits<int>::max() - off) {
    return 42;
  }
  return a[base + off];
}
```

Результат компиляции x86-64 clang 21.1.0 с флагами ``-O2 --std=c++23``: https://godbolt.org/z/f8fP7eMqc
```asm
foo(int*, int, int):
        mov     eax, 42
        test    esi, esi
        js      .LBB0_4
        test    edx, edx
        sets    cl
        mov     r8d, edx
        neg     r8d
        cmp     esi, r8d
        setl    r8b
        test    cl, r8b
        jne     .LBB0_4
        test    edx, edx
        setg    cl
        mov     r8d, edx
        xor     r8d, 2147483647
        cmp     esi, r8d
        seta    r8b
        test    cl, r8b
        jne     .LBB0_4
        add     edx, esi
        movsxd  rax, edx
        mov     eax, dword ptr [rdi + 4*rax]
.LBB0_4:
        ret
```
