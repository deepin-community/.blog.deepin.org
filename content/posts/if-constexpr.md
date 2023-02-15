---
title: 使用 if constexpr 实现条件编译
date: 2022-03-10 13:03:27
draft: false
authors: ["justforlxz"]
tags: ["C++"]
---

在项目开发中，我们通常会使用条件编译对代码进行裁剪，选择性地排除不需要的代码，比如在某个平台下完全不支持某个功能，那么这个功能就不应该被编译。

一般我们使用宏来判断代码，选择性的挑选需要编译的部分，并在构建系统中开启这样的条件。

<!--more-->

```cpp
#ifdef XXXXXXXXXX
    std::cout << "hello world!" << std::endl;
#else
    std::cout << "good bye!" << std::endl;
#endif
```

在 C 语言的项目中，这样的行为是很正常的，甚至在 C++ 项目中，我们也会选择使用宏进行条件判断。

但是如果定义的宏多了，则很容易导致代码碎片化，并且无法直观地看到工作流程。

例如这样的代码:

```cpp
#define KWIN_VERSION_CHECK(major, minor, patch, build) ((major<<24)|(minor<<16)|(patch<<8)|build)
#ifdef KWIN_VERSION_STR
#define KWIN_VERSION KWIN_VERSION_CHECK(KWIN_VERSION_MAJ, KWIN_VERSION_MIN, KWIN_VERSION_PAT, KWIN_VERSION_BUI)
#endif

#if defined(KWIN_VERSION) && KWIN_VERSION < KWIN_VERSION_CHECK(5, 21, 3, 0)
typedef int TimeArgType;
#else
typedef std::chrono::milliseconds TimeArgType;
#endif

#if defined(Q_OS_LINUX) && !defined(QT_NO_DYNAMIC_LIBRARY) && !defined(QT_NO_LIBRARY)
QT_BEGIN_NAMESPACE
QFunctionPointer qt_linux_find_symbol_sys(const char *symbol);
QT_END_NAMESPACE
QFunctionPointer KWinUtils::resolve(const char *symbol)
{
    return QT_PREPEND_NAMESPACE(qt_linux_find_symbol_sys)(symbol);
#else
QFunctionPointer KWinUtils::resolve(const char *symbol)
{
    static QString lib_name = "kwin.so." + qApp->applicationVersion();

    return QLibrary::resolve(lib_name, symbol);
#endif
}
```

从上面的例子中可以看到，代码中一旦出现大量重复的判断条件，代码非常不直观，而且被宏分割成了很多部分。

在一次偶然的机会，我看到了一篇介绍 C++ 17 中的 if constexpr 的用法，可以在编译期进行一些计算，虽然我很早就知道了 constexpr 的用法，但是大家举的例子基本上都是数值计算，让编译器在编译期间将数值进行计算，从而减轻运行时的消耗，我也从来想到其他用法，所以一直没有在项目中使用到。

constexpr 的作用并不是编译期计算数值，而是编译期进行的代码分析，如果代码较小且非常直观，比如大家经常举的例子，在编译期间计算斐波那契数列，这种例子即使不使用 constexpr 显式要求，编译器也会帮助我们开启优化，直接给出结果。

但是如果代码非常复杂，编译器就不一定会为我们做这样的优化，就需要我们手动标记可以计算的位置，要求编译器在编译期间进行求值和优化。

我设想的是，使用 cmake 在构建时，先生成一份文件，将开关的值记录下来，在需要进行判断的地方，就可以直接使用 if constexpr 进行条件判断，在编译期间，编译器会发现有一个分支确定不会被执行（相当于  `if(false) {}`），那么这个分支就不会进行编译，直接剔除。

CMakeLists.txt 中需要做一些工作，将编译参数加入构建系统。

```cmake
option (ENABLE_MODULE "Enable Module" ON)

if(ENABLE_MODULE)
    set(ENABLE_MODULE "1")
else()
    set(ENABLE_MODULE "0")
endif(ENABLE_MODULE)

configure_file (
  "${CMAKE_CURRENT_SOURCE_DIR}/options/options.h.in"
  "${CMAKE_CURRENT_BINARY_DIR}/options/options.h"
)
```

在 options/options.h.in 文件里，按照 cmake 的要求将变量导入进文件中，进行内容替换。

```cpp
#pragma once

#cmakedefine01 ENABLE_MODULE
```

这里我仍然使用的是宏定义，也可以直接写成如下形式:

```cmake
option (ENABLE_MODULE "Enable Module" ON)

if(ENABLE_MODULE)
    set(ENABLE_MODULE "true")
else()
    set(ENABLE_MODULE "false")
endif(ENABLE_MODULE)

configure_file (
  "${CMAKE_CURRENT_SOURCE_DIR}/options/options.h.in"
  "${CMAKE_CURRENT_BINARY_DIR}/options/options.h"
)
```

```cpp
#pragma once

const bool ENABLE_MODULE{@ENABLE_MODULE@};
```

在 main.cpp 中写一段测试代码:

```cpp
#include "options/options.h"

#include <iostream>

int main() {
    if constexpr (ENABLE_MODULE) {
        std::cout << "Now Enable Module" << std::endl;
    }

    if constexpr (!ENABLE_MODULE) {
        std::cout << "Now Disable Module" << std::endl;
    }

    return 0;
}
```

执行结果是符合预期的。

```txt
# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:28:39]
$ cmake ../ -G Ninja -DENABLE_MODULE=ON
-- The CXX compiler identification is AppleClang 13.0.0.13000029
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/lxz/Develop/constexpr-demo/build

# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:28:45]
$ ninja
[2/2] Linking CXX executable src/constexpr

# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:28:48]
$ ./src/constexpr
Now Enable Module

# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:28:52]
$ cmake ../ -G Ninja -DENABLE_MODULE=OFF
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/lxz/Develop/constexpr-demo/build

# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:28:58]
$ ninja
[2/2] Linking CXX executable src/constexpr

# lxz @ lxz-MacBook-Pro in ~/Develop/constexpr-demo/build on git:master o [13:29:00]
$ ./src/constexpr
Now Disable Module
```

虽然结果是符合的，但是我们其实不确定是否真的在编译期间就完成了代码剔除，所以使用命令进行汇编，查看汇编中是否包含了判断指令和两段输出的字符串。

```shell
clang -S main.cpp -o main.s -I./
```

main.s

```txt
	.text
	.file	"main.cpp"
	.globl	main                            // -- Begin function main
	.p2align	2
	.type	main,@function
main:                                   // @main
	.cfi_startproc
// %bb.0:
	stp	x29, x30, [sp, #-32]!           // 16-byte Folded Spill
	str	x19, [sp, #16]                  // 8-byte Folded Spill
	mov	x29, sp
	.cfi_def_cfa w29, 32
	.cfi_offset w19, -16
	.cfi_offset w30, -24
	.cfi_offset w29, -32
	adrp	x19, :got:_ZSt4cout
	ldr	x19, [x19, :got_lo12:_ZSt4cout]
	adrp	x1, .L.str
	add	x1, x1, :lo12:.L.str
	mov	w2, #18
	mov	x0, x19
	bl	_ZSt16__ostream_insertIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_PKS3_l
	ldr	x8, [x19]
	ldur	x8, [x8, #-24]
	add	x8, x8, x19
	ldr	x19, [x8, #240]
	cbz	x19, .LBB0_5
// %bb.1:
	ldrb	w8, [x19, #56]
	cbz	w8, .LBB0_3
// %bb.2:
	ldrb	w1, [x19, #67]
	b	.LBB0_4
.LBB0_3:
	mov	x0, x19
	bl	_ZNKSt5ctypeIcE13_M_widen_initEv
	ldr	x8, [x19]
	mov	w1, #10
	mov	x0, x19
	ldr	x8, [x8, #48]
	blr	x8
	mov	w1, w0
.LBB0_4:
	adrp	x0, :got:_ZSt4cout
	ldr	x0, [x0, :got_lo12:_ZSt4cout]
	bl	_ZNSo3putEc
	bl	_ZNSo5flushEv
	ldr	x19, [sp, #16]                  // 8-byte Folded Reload
	mov	w0, wzr
	ldp	x29, x30, [sp], #32             // 16-byte Folded Reload
	ret
.LBB0_5:
	bl	_ZSt16__throw_bad_castv
.Lfunc_end0:
	.size	main, .Lfunc_end0-main
	.cfi_endproc
                                        // -- End function
	.section	.text.startup,"ax",@progbits
	.p2align	2                               // -- Begin function _GLOBAL__sub_I_main.cpp
	.type	_GLOBAL__sub_I_main.cpp,@function
_GLOBAL__sub_I_main.cpp:                // @_GLOBAL__sub_I_main.cpp
	.cfi_startproc
// %bb.0:
	stp	x29, x30, [sp, #-32]!           // 16-byte Folded Spill
	str	x19, [sp, #16]                  // 8-byte Folded Spill
	mov	x29, sp
	.cfi_def_cfa w29, 32
	.cfi_offset w19, -16
	.cfi_offset w30, -24
	.cfi_offset w29, -32
	adrp	x19, _ZStL8__ioinit
	add	x19, x19, :lo12:_ZStL8__ioinit
	mov	x0, x19
	bl	_ZNSt8ios_base4InitC1Ev
	adrp	x0, :got:_ZNSt8ios_base4InitD1Ev
	ldr	x0, [x0, :got_lo12:_ZNSt8ios_base4InitD1Ev]
	mov	x1, x19
	ldr	x19, [sp, #16]                  // 8-byte Folded Reload
	adrp	x2, __dso_handle
	add	x2, x2, :lo12:__dso_handle
	ldp	x29, x30, [sp], #32             // 16-byte Folded Reload
	b	__cxa_atexit
.Lfunc_end1:
	.size	_GLOBAL__sub_I_main.cpp, .Lfunc_end1-_GLOBAL__sub_I_main.cpp
	.cfi_endproc
                                        // -- End function
	.type	_ZStL8__ioinit,@object          // @_ZStL8__ioinit
	.local	_ZStL8__ioinit
	.comm	_ZStL8__ioinit,1,1
	.hidden	__dso_handle
	.type	.L.str,@object                  // @.str
	.section	.rodata.str1.1,"aMS",@progbits,1
.L.str:
	.asciz	"Now Disable Module" // 关键在这里
	.size	.L.str, 19

	.section	.init_array,"aw",@init_array
	.p2align	3
	.xword	_GLOBAL__sub_I_main.cpp
	.ident	"clang version 13.0.1"
	.section	".note.GNU-stack","",@progbits
	.addrsig
	.addrsig_sym _GLOBAL__sub_I_main.cpp
	.addrsig_sym _ZStL8__ioinit
	.addrsig_sym __dso_handle
	.addrsig_sym _ZSt4cout
```

查看整个 main.s 汇编，发现只在 .L.str 段中有预期的文本字符串，可以得出结论，代码是在编译期间完成了剔除，符合我们的要求。

原文链接：[https://blog.justforlxz.com/2022/03/10/if-constexpr/](https://blog.justforlxz.com/2022/03/10/if-constexpr/)
