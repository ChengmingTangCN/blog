+++
title = '2025 12 23 Odr Violations Dynamic Static Libraries'
date = 2025-12-23T22:38:28+08:00
+++

C++中多个动态库链接同一个静态库: 违反ODR导致double free

## 引言

最近在研究C++中单例模式, 发现网上有人提到这个问题. 记录一下探究的过程.

google找到了这个能够复现该问题的帖子: [Global variable in static library - double free or corruption error](https://gcc.gnu.org/legacy-ml/gcc-help/2010-10/msg00248.html)

复现流程如下:

- 静态库里定义一个全局变量, 该对象构造时分配内存, 析构时释放内存

- 有2个动态库都使用到了该变量, 动态库都链接了该静态库

- 主程序里调用2个动态库的函数, 运行时libc提示检测到了double free

```bash
$ ./reproduce.sh
Foo::Foo(int) begin: 0x7d15500f9048 0
Foo::Foo(int) end: 0x7d15500f9048 0x61720d3316c0
Foo::Foo(int) begin: 0x7d15500f9048 0x61720d3316c0
Foo::Foo(int) end: 0x7d15500f9048 0x61720d3316e0
void a(): 0x7d15500f9048
void b(): 0x7d15500f9048
Foo::~Foo(): 0x7d15500f9048 0x61720d3316e0
Foo::~Foo(): 0x7d15500f9048 0x61720d3316e0
free(): double free detected in tcache 2
./reproduce.sh: line 8:  5622 Aborted                 (core dumped) LD_LIBRARY_PATH=. ./main.elf
```

## 测试环境

```bash
$ cat ./reproduce.sh
#!/bin/env bash

g++ ./foo.cpp -std=c++17 -O2 -g -fPIC -c -o foo.o
ar rcs libfoo.a foo.o
g++ ./a.cpp -std=c++17 -O2 -g -fPIC -shared -L. -lfoo -o liba.so
g++ ./b.cpp -std=c++17 -O2 -g -fPIC -shared -L. -lfoo -o libb.so
g++ ./main.cpp -std=c++17 -O2 -g -L. -la -lb -o main.elf
LD_LIBRARY_PATH=. ./main.elf
```

```c++
// foo.h
#pragma once

#include <iostream>

class Foo {
public:
  Foo(int val) {
    std::cout << __PRETTY_FUNCTION__ << " begin: " << this << " " << p_ << "\n";
    p_ = new int(val);
    std::cout << __PRETTY_FUNCTION__ << " end: " << this << " " << p_ << "\n";
  }
  Foo(Foo const &) = delete;
  Foo &operator=(Foo const &) = delete;
  ~Foo() noexcept {
    std::cout << __PRETTY_FUNCTION__ << ": " << this << " " << p_ << "\n";
    delete p_;
  }

private:
  int *p_;
};

extern Foo foo;
```

```c++
// foo.cpp
#include "foo.h"

Foo foo {0x666};
```

```c++
// a.cpp
#include "foo.h"

void a() { std::cout << __PRETTY_FUNCTION__ << ": " << &foo << "\n"; }
```

```c++
// b.cpp
#include "foo.h"

void b() { std::cout << __PRETTY_FUNCTION__ << ": " << &foo << "\n"; }
```

```c++
// main.cpp
extern void a();

extern void b();

int main() {
  a();
  b();

  return 0;
}
```

## 原因分析

分析输出的字符串, 可以看到里同一块内存调用了2次构造函数, 2次析构函数. 第2次构造函数产生了内存泄露, 第1次申请的内存不会被释放, 而第2次申请的内存会被反复释放, 导致了double free.

其中`liba.so`以及`libb.so`里都链接了`libfoo.a`, 所以都定义了一份`foo`, 程序加载时`ld.so`会对符号进行合并, `liba.so`以及`libb.so`里重定位后都会使用一份内存.

而`liba.so`以及`libb.so`都会在`.init_array`以及`.fini_array`里执行`Foo::Foo(int)`以及`Foo::~Foo()`, 导致了最终的crash.

这个应该属于UB, ODR要求foo在整个程序里只能存在一份定义, 而现在有2份.

> One and only one definition of every non-inline function or variable that is odr-used (see below) is required to appear in the entire program (including any standard and user-defined libraries).
> The compiler is not required to diagnose this violation, but the behavior of the program that violates it is undefined.

通过`readelf`以及`objdump`简单验证我们的猜想.

可以看到`liba.so`以及`libb.so`里都定义了`foo`符号.
```
File: ./liba.so

Symbol table '.symtab' contains 41 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    26: 0000000000004048     8 OBJECT  GLOBAL DEFAULT   26 foo

File: ./libb.so

Symbol table '.symtab' contains 41 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
    27: 0000000000004048     8 OBJECT  GLOBAL DEFAULT   26 foo
```

查看so的`.init_array` section

```
$ objdump -s -j .init_array ./liba.so

./liba.so:     file format elf64-x86-64

Contents of section .init_array:
 3dc8 e0110000 00000000 00110000 00000000  ................
$ objdump -s -j .init_array ./libb.so

./libb.so:     file format elf64-x86-64

Contents of section .init_array:
 3dc8 e0110000 00000000 00110000 00000000  ................
```

反汇编查看是否调用了`Foo:Foo(int)`, 可以发现`.init_array`里的第3项指向的函数调用了构造函数.

```asm
$ objdump -d -Mintel ./liba.so | c++filt
0000000000001100 <_GLOBAL__sub_I_foo.cpp>:
    1100:       f3 0f 1e fa             endbr64
    1104:       53                      push   rbx
    1105:       48 8b 1d ac 2e 00 00    mov    rbx,QWORD PTR [rip+0x2eac]        # 3fb8 <foo@@Base-0x90>
    110c:       be 66 06 00 00          mov    esi,0x666
    1111:       48 89 df                mov    rdi,rbx
    1114:       e8 d7 ff ff ff          call   10f0 <Foo::Foo(int)@plt>
    1119:       48 8b 3d a8 2e 00 00    mov    rdi,QWORD PTR [rip+0x2ea8]        # 3fc8 <Foo::~Foo()@@Base+0x2d78>
    1120:       48 89 de                mov    rsi,rbx
    1123:       5b                      pop    rbx
    1124:       48 8d 15 05 2f 00 00    lea    rdx,[rip+0x2f05]        # 4030 <__dso_handle>
    112b:       e9 80 ff ff ff          jmp    10b0 <__cxa_atexit@plt>
```

进一步确认`0x3fb8`通过重定位项指向了`foo`.

```
$ readelf -rW ./liba.so | c++filt

Relocation section '.rela.dyn' at offset 0x758 contains 12 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000003fb8  0000000d00000006 R_X86_64_GLOB_DAT      0000000000004048 foo + 0
```

怎么解决这个问题: C++17开始支持inline variable, 它允许在程序中多次定义.

## inline variable

> There can be more than one definition in a program of each of the following: class type, enumeration type, inline function, inline variable(since C++17),
> templated entity (template or member of template, but not full template specialization), as long as all following conditions are satisfied:

> For an inline function or inline variable(since C++17), a definition is required in every translation unit where it is odr-used.

修改`foo.h`以及`foo.cpp`, 将`foo`定义放到头文件, 注释掉`foo.cpp`里原本的定义.

```
// foo.h
inline Foo foo {0x666};
```

```
// foo.cpp
// Foo foo {0x666};
```

可以看到现在`foo`只会构造1次, 析构1次.

```bash
$ ./reproduce.sh
Foo::Foo(int) begin: 0x747cb4e42050 0
Foo::Foo(int) end: 0x747cb4e42050 0x5d921df826c0
void a(): 0x747cb4e42050
void b(): 0x747cb4e42050
Foo::~Foo(): 0x747cb4e42050 0x5d921df826c0
```

继续通过`readelf`以及`objdump`简单分析下底层机制. 发现多了一个guard variable, 第1次构造后将它置1,
第2次执行到这发现已经置1后, 就不会执行`Foo:FOO(int)`.

```asm
0000000000001100 <_GLOBAL__sub_I_a.cpp>:
    1100:       f3 0f 1e fa             endbr64
    1104:       48 8b 05 9d 2e 00 00    mov    rax,QWORD PTR [rip+0x2e9d]        # 3fa8 <guard variable for foo@@Base-0xa0>
    110b:       80 38 00                cmp    BYTE PTR [rax],0x0
    110e:       74 01                   je     1111 <_GLOBAL__sub_I_a.cpp+0x11>
    1110:       c3                      ret
    1111:       53                      push   rbx
    1112:       48 8b 1d 9f 2e 00 00    mov    rbx,QWORD PTR [rip+0x2e9f]        # 3fb8 <foo@@Base-0x98>
    1119:       be 66 06 00 00          mov    esi,0x666
    111e:       c6 00 01                mov    BYTE PTR [rax],0x1
    1121:       48 89 df                mov    rdi,rbx
    1124:       e8 c7 ff ff ff          call   10f0 <Foo::Foo(int)@plt>
    1129:       48 8b 3d 98 2e 00 00    mov    rdi,QWORD PTR [rip+0x2e98]        # 3fc8 <Foo::~Foo()@@Base+0x2d68>
    1130:       48 89 de                mov    rsi,rbx
    1133:       5b                      pop    rbx
    1134:       48 8d 15 f5 2e 00 00    lea    rdx,[rip+0x2ef5]        # 4030 <__dso_handle>
    113b:       e9 70 ff ff ff          jmp    10b0 <__cxa_atexit@plt>
```

```
$ readelf -rW ./liba.so  | c++filt

Relocation section '.rela.dyn' at offset 0x778 contains 13 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000003fa8  0000001300000006 R_X86_64_GLOB_DAT      0000000000004048 guard variable for foo + 0
0000000000003fb8  0000000d00000006 R_X86_64_GLOB_DAT      0000000000004050 foo + 0
```



