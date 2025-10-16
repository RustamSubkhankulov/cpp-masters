## HW 3.2
#### Task: Try to explain what is going wrong here
```cpp
template <typename... Ts>
concept Addable =
  requires(Ts... p) {
    (... + p);
    requires sizeof...(Ts) > 1;
  };

template <Addable... Ts>
auto sum_all(Ts... p) {
  return (... + p);
}

sum_all(1, 2); // FAIL
```

#### Solution:

Полный исходный код:
```cpp
template <typename... Ts> concept VariadicAddable =
  requires(Ts... vs) {
    (... + vs);
    requires sizeof...(Ts) > 1;
  };

template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
  return (... + vs);
}

int main() {
  return add_variadic(1, 2); // FAIL
}
```

Вывод сообщения об ошибке компиляторами:

clang 21.1.0:
```text
<source>:14:10: error: no matching function for call to 'add_variadic'
   14 |   return add_variadic(1, 2); // FAIL
      |          ^~~~~~~~~~~~
<source>:9:39: note: candidate template ignored: constraints not satisfied [with Ts = <int, int>]
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |                                       ^
<source>:9:11: note: because 'int' does not satisfy 'VariadicAddable'
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |           ^
<source>:6:14: note: because 'sizeof...(Ts) > 1' (1 > 1) evaluated to false
    6 |     requires sizeof...(Ts) > 1;
      |              ^
<source>:9:11: note: and 'int' does not satisfy 'VariadicAddable'
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |           ^
<source>:6:14: note: because 'sizeof...(Ts) > 1' (1 > 1) evaluated to false
    6 |     requires sizeof...(Ts) > 1;
      |              ^
1 error generated.
Compiler returned: 1
```

gcc 15.2:
```text
<source>:14:10: error: no matching function for call to 'add_variadic'
   14 |   return add_variadic(1, 2); // FAIL
      |          ^~~~~~~~~~~~
<source>:9:39: note: candidate template ignored: constraints not satisfied [with Ts = <int, int>]
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |                                       ^
<source>:9:11: note: because 'int' does not satisfy 'VariadicAddable'
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |           ^
<source>:6:14: note: because 'sizeof...(Ts) > 1' (1 > 1) evaluated to false
    6 |     requires sizeof...(Ts) > 1;
      |              ^
<source>:9:11: note: and 'int' does not satisfy 'VariadicAddable'
    9 | template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
      |           ^
<source>:6:14: note: because 'sizeof...(Ts) > 1' (1 > 1) evaluated to false
    6 |     requires sizeof...(Ts) > 1;
      |              ^
1 error generated.
Compiler returned: 1
```

Отметим следующий момент из вывода - ``'sizeof...(Ts) > 1'`` - проверка на размер пачки параметров возвращает ``false``, при этом оба компилятора сообщают о том, что ``'int' does not satisfy 'VariadicAddable'`` - дважды, для каждого шаблонного параметра в пачке по отдельности. Это означает, что на соответствие концепту ``VariadicAddable`` проверяется ``int``.

Поэкспериментируем:
```cpp
template <typename... Ts> concept VariadicAddable =
  requires(Ts... vs) {
    (... + vs);
    requires sizeof...(Ts) > 1;
  };

template <VariadicAddable... Ts> auto add_variadic(Ts... vs) {
  return (... + vs);
}

int main() {
  return add_variadic(1., 2); // FAIL
}
```

Обратимся к стандарту:
\[1\]"A declaration’s associated constraints are defined as follows:
  (3.1) — If there are no introduced constraint-expressions, the declaration has no associated constraints.
  (3.2) — Otherwise, if there is a single introduced constraint-expression, the associated constraints are the normal form (13.5.4) of that expression.
  (3.3) — Otherwise, the associated constraints are the normal form of a logical and expression (7.6.14) whose operands are in the following order:
    - the constraint-expression introduced by each type-constraint (13.2) in the declaration’s template parameter-list, in order of appearance, and 
    - the constraint-expression introduced by a requires-clause following a template-parameter-list (13.1), and ..."\[temp.constr.decl]\]

Пример из стандарта:
```cpp
template<C1... T> struct s2; // associates (C1<T> && ...)
template<C2... T> struct s3; // associates (C2<T> && ...)
```

Это означает, что в данной форме концепт ``VariadicAddable`` проверяется для каждого шаблонного параметра в отдельности. 

Исправление - вынести проверку концепта в отдельный requires-clause:
```cpp
template <typename... Ts>
  requires VariadicAddable<Ts...>
  auto add_variadic(Ts... vs) {
  return (... + vs);
}
```

Это позволяет применить концепт ко всей пачке параметров.

\[2\] "An atomic constraint is formed from an expression E and a mapping from the template parameters that appear within E to template arguments that are formed via substitution during constraint normalization in the declaration of a constrained entity (and, therefore, can involve the unsubstituted template parameters of the constrained entity), called the parameter mapping."" \[temp.const.atomic\]
\[3\] "To determine if an atomic constraint is satisfied, the parameter mapping and template arguments are first substituted into its expression." \[temp.const.atomic\]
\[4\] "... Otherwise, if there is a single introduced constraint-expression, the associated constraints are the normal form of that expression." \[temp.const.atomic\]
\[5\] "The normal form of a concept-id C\<A1, A2, ..., An\> is the normal form of the constraint-expression of C, after substituting A1, A2, ..., An for C’s respective template parameters in the parameter mappings in each atomic constraint." \[temp.constr.normal\]

В данном случае пачка шаблонных параметров мапится на пачку шаблонных параметров в определении концепта, что соответствует ожидаемому нами результату.
