## HW 3.1
#### Task: Design realistic overload set for generic function nth_power using constraints. Account for integers, floating point and matrices.

#### Solution:

```cpp

#include <concepts>
#include <type_traits>

#include <cmath>
#include <cstddef>

#include <iostream>
#include <vector>

/* Concept to eliminate bool. */
template <typename T>
concept NonBoolean = !std::same_as<std::remove_cvref_t<T>, bool>;

/*
 * Concepts for integral types besides boolean, floating point 
 * types and matrix-like types.
 */
template <typename T>
concept ArithIntegral = std::integral<T> && NonBoolean<T>;

template <typename T>
concept Floating = std::floating_point<T>;

template <typename T>
concept MatrixLike =
  requires(T a, T b, size_t n) {
    /* operator*() */
    { a * b } -> std::same_as<T>;                                     
    
    /* identity matrix of given dimension */
    { T::identity(n) } -> std::same_as<T>;
    
    /* dimension */
    { std::declval<T>().size() } -> std::convertible_to<std::size_t>;
  };

/* United concept for types that are ArithIntegral or Floating. */
template <typename T>
concept ScalarArith = ArithIntegral<T> || Floating<T>;

/* Logarithmic complexity algorithm. */
template <ScalarArith T>
constexpr T nth_power(T x, unsigned n) noexcept {
  
  T result = T{1};
  
  while (n > 0U) {
    if (n & 1U) {
      result *= x;
    }
    
    x *= x;
    n >>= 1U;
  }
  return result;
}

/* Same algorithm with few minor implementation differences. */
template <MatrixLike T>
T nth_power(const T& x, unsigned n) {
    
  if (n == 0U) {
    return T::identity(x.size());
  }
  
  if (n == 1U) {
    return x;
  }

  T result = T::identity(x.size());
  T base = x;

  while (n > 0U) {
    if (n & 1U) {
      result = result * base;
    }
    base = base * base;
    n >>= 1U;
  }
  return result;
}

/* Matrix example. */
struct Matrix {
  std::vector<std::vector<double>> data;

  Matrix(size_t n)
    : data(n, std::vector<double>(n, 0.0)) {}

  size_t size() const { return data.size(); }

  static Matrix identity(size_t n) {
    Matrix I(n);
    for (size_t i = 0; i < n; ++i) { 
      I.data[i][i] = 1.0;
    }
    return I;
  }

  Matrix operator*(const Matrix& other) const {
    size_t n = size();
    Matrix result(n);
    
    for (size_t i = 0; i < n; ++i) {
      for (size_t j = 0; j < n; ++j) {
        for (size_t k = 0; k < n; ++k) {
          result.data[i][j] += data[i][k] * other.data[k][j];
        }
      }
    }

    return result;
  }

  void print(const char* label = "") const {
    if (label && *label) std::cout << label << std::endl;
    for (auto& row : data) {
      for (auto v : row) {
        std::cout << v << '\t';
      }
      std::cout << std::endl;
    }
  }
};

int main() {
  /* ArithIntegral */
  std::cout << "int: -2 ^ 6u = " << nth_power(-2, 6U) << std::endl;
  std::cout << "int: -2 ^ 7u = " << nth_power(-2, 7U) << std::endl;

  std::cout << "unsigned: 2u ^ 6u = " << nth_power(2u, 6U) << std::endl;
  std::cout << "unsigned: 2u ^ 7u = " << nth_power(2u, 7U) << std::endl;

  /* Floating */
  std::cout << "float:  2.5f ^ 6u = " << nth_power(2.5f, 6U) << std::endl;
  std::cout << "double: 2.5  ^ 6u = " << nth_power(2.5 , 6U) << std::endl;

  /* Matrix */
  Matrix m(2);
  m.data = {{1., 2.}, {3., 4.}};
  
  std::cout << "matrix: \n";
  m.print();
  std::cout << std::endl;

  std::cout << "6th power: \n";
  auto res = nth_power(m, 6U);
  res.print();

  // nth_power(true, 2U);  // CE
  // nth_power(false, 2U); // CE
}
```

Example output:
```text
int: -2 ^ 6u = 64
int: -2 ^ 7u = -128
unsigned: 2u ^ 6u = 64
unsigned: 2u ^ 7u = 128
float:  2.5f ^ 6u = 244.141
double: 2.5  ^ 6u = 244.141
matrix:
1	2
3	4

6th power:
5743	8370
12555	18298
```
