
## 内存对齐

- 在未与类内存对齐的内存块上，使用placement new构造对象，可能会出错
- placement new构造对象，需使用内存对齐的内存块
```cpp
struct A { int var; };

char xx[32]; // 1字节对齐
new (xx) A; // xx的起始地址，可能不再A对象指定的对齐位置上


template < std::size_t len, std::size_t Align = /*default-alignment*/ > // len参数为类型的size，align参数为该类型的内存对齐大小
struct aligned_storage;

#include <type_traits>

using Aligned_A = std::aligned_storage< sizeof(A), std::alignment_of<A>::value >::type;

int main() {
    Aligned_A a, b;
    new (&a) A(1);
    b = a;
    std::cout << reinterpret_cast<A&>(b).var;
}
```