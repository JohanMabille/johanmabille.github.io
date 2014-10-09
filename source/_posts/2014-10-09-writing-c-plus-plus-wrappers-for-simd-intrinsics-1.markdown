---
layout: post
title: "Writing C++ wrappers for SIMD intrinsics (1)"
date: 2014-10-09 09:10:38 +0200
comments: true
categories: [SIMD,vectorization]
published: false
---

## Introduction

SIMD stands for Single Instruction, Mutiple Data, a class of parallel computers which can perform
the same operation on multiple data points simultaneously. Let's consider a summation we want to perform
on two sets of four floating point numbers. The difference bewteen scalar and SIMD operations is illustrated
below :

![simd_scalar](/images/simd_scalar.png)

Using scalar operation, four add instructions must be executed one after other to obtain the sums, whereas
SIMD uses a single instruction to achieve the same result. Thus SIMD operations yield higher efficiency than
scalar operations.

SIMD instructions were first used in the early 1970s, but began to spread with desktop processors powerful
enough to support real-time gaming and video processing. Each processor manufacturer has implemented its
own SIMD instruction set :

+ MMX / SSE / AVX (Intel processors)
+ 3DNow! (AMD processors)
+ Altivec (Motorola processors)
+ MDMX (MIPS processors)

Many of these instruction set still coexist nowadays, and you have to deal with all of them if you want to write portable
software. This is a first argument for writing wrappers : capture the abstraction of SIMD operations in nice interfaces,
and forget about the implementation you rely on.

Although you can write assembly code to use the SIMD instructions, compilers usually provide built-in functions so you
can use SIMD instructions in C without writing a single line in assembler. These functions (and more generally functions
whose implementation is handled specially by the compiler) are called intrinsic functions, often shortened to intrinsics.
Of course the SIMD intrinsics depend on the underlying architecture, and may differ from one compiler to other even for a
same SIMD instruction set. Fortunately, compilers tend to standardize intrinsics prototype for a given SIMD instruction set,
and we only have to handle the differences between the various SIMD instruction set.

I'll focus on wrapping Intel SIMD instruction set through this series of articles, but the wrappers will
be generic enough so plugging other instruction set is easy.

Since SIMD instructions are longtstanding, you might wonder if writing your own wrapper is relevant; maybe someone
did it before and there no need to write any code, all you have to do is to reuse existing code. Well, yes and no.

[Agner Fog](http://www.agner.org/optimize/#vectorclass) has written some very usefull classes that handle Intel SIMD
instruction set (different versions of SSE and AVX), but he doesn't make heavy use of metaprogramming in its
implementation; thus adding a new wrapper (for a new instruction set or a new version of an existing one)
requires to type a lot of code that could have been factorized. Moreover, some essential tools are missing, such
as an aligned memory allocator (we'll see why you need such a tool later). However his library is
a good point to start.

Another library you might want to consider is the [Numerical Template Toolbox](https://github.com/MetaScale/nt2).
Although it has a very comprehensive set of mathematical functions, its major drawback is it really slows the
compilation. More over its development and documentation aren't finished yet, and it might be difficult to extend it.

And last but not least, writing your own wrapper will make you confront issues specific to SIMD instructions
and make you understand how it works; thus you will be able to use SIMD intrinsics in a really efficient way, regardless
of the implementation you choose (your own wrappers or an existing library).

## 1. SSE/AVX intrinsics

Before we start to write any code, we need to see the instrinsics the compiler provides, and how it's organized.
For the rest of this article and the following ones, I assume we use an Intel processor, recent enough to provide SSE 4
and AVX; the compiler can be gcc or MSVC, the instrinsics they provide are almost the same.

If you already know about SSE / AVX intrinsics you can skip this section.

### 1.1 Registers

SSE uses eight 128 bits registers, from xmm0 to xmm7; Intel and AMD 64 bits extensions adds eight more registers, from
xmm8 to xmm15; thus SSE intrinsics can perform on 4 packed float, 2 packed double, 4 32-bits integers, etc ...

With AVX, the width of the SIMD registers is increased from 128 to 256 bits; the register are renamed from xmm0-xmm7
to ymm0-ymm7 (and from xmm8-xmm15 to ymm8 to ymm15); however legacy sse instructions still can be used, and xmm
registers can still be addressed since they're the lower part of ymm registers.

AVX512 will increase the width of the SIMD registers from 256 to 512 bits.

### 1.2 Files to include

The intrinsic functions are splitted among different files, depending on the version of the SIMD instruction set they
belong to :

* \<xmmintrin.h\> : SSE, operations on 4 single precision floating point numbers (float).
* \<emmintrin.h\> : SSE 2, operations on integers and on 2 double precision floating point numbers (double).
* \<pmmintrin.h\> : SSE 3, horizontal operations on SIMD registers.
* \<tmmintrin.h\> : SSSE 3, additional instructions.
* \<smmintrin.h\> : SSE 4.1, dot product and many operations on integers
* \<nmmintrin.h\> : SSE 4.2, additional instructions.
* \<immintrin.h\> : AVX, operations on integers, 8 float or 4 double.

Each of these files includes the previous one, so you only have to include the one matching the highest version of the SIMD
instruction set available in your processor. Later we'll see how to detect at compile time which version on SIMD instruction
set is available and thus which file to include. For now, just assume we're able to include the right file each time we need it.

### 1.3 Naming rules

Now if you take a look at these files, you'll notice provided data and functions follow some naming rules :

* data vectors are named **\_\_mXXX(T)**, where :
	* XXX is the number of bits of the vector (128 for SSE, 256 for AVX)
	* is T a character for the type of the data; T is omitted for float, i fot integers and d for double; thus \_\_m128d is the
	data vector to use when performing SSE instructions on double.
* intrinsic functions operating on floating point numbers are usually named **\_mm(XXX)_NAME_PT**, where :
	* XXX is the number of bits of the SIMD registers; it is omitted for 128 bits regusters
	* NAME is the short name of the function (add, sub, cmp, ...)
	* P indicates whether the functions operates on a packed data vector (p) or on a scalar only (s)
	* T indicates the type of the floating point numbers : s for single precision, d for double precision
* intrinsic functions operating on integers are usually named **\_mm(XXX)_NAME_EPSYY**, where :
	* XXX is the number of bits of the SIMD registers; it is omitted for 128 bits registers
	* NAME is the short name of the function (add, sub, cmp)
	* S indicates whether the integers are signed (i) or unsigned (u)
	* YY is the number of bits of the integer

### 1.4 Intrinsics categories

Intrinsics encompass a wide set of features; we can distinguish the following categories (not exhausive) :

* Arithmetic : _mm_add_xx, _mm_sub_xx, _mm_mul_xx, _mm_div_xx, ...
* Logical : _mm_and_xx, _mm_or_xx, _mm_xor_xx, ...
* Comparison : _mm_cmpeq_xx, _mm_cmpneq_xx, _mm_cmplt_xx, ...
* Conversion : _mm_cvtepixx, ...
* Memory move : _mm_load_xx, _mm_store_xx, ...
* Setting : _mm_set_xx, _mm_setzero_xx, ...

Some intrinsics won't be wrapped, and some of them will bu used only to build higher level functions in the wrappers.

### 1.5 Sample code

Now you know a little more about SSE and AVX intrinsics, you may reconsider the need for wrapping them; indeed, if you don't need
to handle other instructions set, you could think of using SSE/AVX intrinsics directly. I hope this sample code will make you
change your mind :

{% coderay SSE_sample.cpp %}
// computes e = a*b + c*d using SSE where a, b, c, d and e are vector of floats
for(size_t i = 0; i < e.size(); i += 4)
{
	__m128 val = _mm_add_ps(_mm_mul_ps(_mm_load_ps(&a[i]),_mm_load_ps(&b[i])),
							_mm_mul_ps(_mm_load_ps(&c[i]),_mm_load_ps(&d[i])));
	_mm_store_ps(&e[i],val);
}
{% endcoderay %}

Quite hard to read, right ? And this is just for two multiplications and one addition; imagine using intrinsics in a huge amount of code,
and you'll get code really hard to understand and to maintain. What we need is a way to use \_\_m128 with traditional arithmetic
operators, as we do with float :

{% coderay wrapped_sample.cpp %}
// computes e = a*b + c*d using SSE where a, b, c, d and e are vector of floats
for(size_t i = 0; i < e.size(); i += 4)
{
	__m128 val = load(&a[i]) * load(&b[i]) + load(&c[i]) * load(&d[i]);
	store(&e[i],val);
}
{% endcoderay %}

That's the aim of the wrappers we start to write in the next section.

