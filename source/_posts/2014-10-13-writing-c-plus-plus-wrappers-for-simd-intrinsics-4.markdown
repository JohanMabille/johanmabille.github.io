---
layout: post
title: "Writing C++ wrappers for SIMD intrinsics (4)"
date: 2014-10-13 22:46:01 +0200
comments: true
categories: [SIMD,Vectorization]
published: false
---

## 3. Plugging the wrappers into existing code

### 3.1 Storing vector4f instead of float

Now we have nice wrappers, let's see how we can use them in real code. Suppose you have the following
computation loop:

{% coderay sample.cpp %}
std::vector<float> a, b, c, d, e;
// somewhere in the code, a, b, c, d and e are
// resized so they hold n elements
// ...
for(size_t i = 0; i < n; ++i)
{
	e[i] = a[i]*b[i] + c[i]*d[i];
}
{% endcoderay %}

<!-- more -->

A first solution could be to store vector of vector4f instead of vector of float:

{% coderay sample.cpp %}
std::vector<vector4f> a, b ,c, d, e;
// somewhere in the code, a, b, c, d and e are
// resized so they hold n/4 vector4f
// ...
for(size_t i = 0; i < n/4; ++i)
{
	e[i] = a[i]*b[i] + c[i]*d[i];
}
{% endcoderay %}

Not so bad, thanks to the operators overloads, the code is exactly the same as the one for float, but
the operations are performed on four floats at once. If n is not a multiple of four, we allocate an
additional vector4f in each vector and we initialize the useless elements with 0.

The problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for a specific element in the vector or if you fill your vector pushing back elements one by one. In this
case, you would have to recode any piece of algorithm that works on single elements (and that includes a
lot of STL algorithms) and then add special code for working on scalars inside a vector4f. Working on
scalars inside vector4f is possible (we'll see later how to modify our wrappers so we can do ir), but is
slower than working directly on scalars, thus you could lose the benefits of using vectorization.

### 3.2 Initializing vector4f from container of float

Another solution could be to initialize the wrapper from values stored in a vector:

{% coderay sample.cpp %}
std::vector<float>a, b, c, d, e;
// somewhere in the code, a, b, c, d and e are
// resized so they hold n elements
/ ...
for(size_t i = 0; i < n/4; i += 4)
{
	vector4f av(a[i],a[i+1],a[i+2],a[i+3]);
	vector4f bv(b[i],b[i+1],b[i+2],b[i+3]);
	vector4f cv(c[i],c[i+1],c[i+2],c[i+3]);
	vector4f dv(d[i],d[i+1],d[i+2],d[i+3]);

	vector4f ev = av*bv + cv*dv;
	// how do we store ev in e[i],e[i+1],e[i+2],e[i+3] ?
}
for(size_t i = n/4; i < n; ++i)
{
	e[i] = a[i]*b[i] + c[i]*d[i];
}
{% endcoderay %}

The first problem is we need a way to store a vector4f in 4 floats; as said in the previous paragraph, we
can add a method to our wrappers that return a scalar inside the vector4f and invoke it that way:

{%coderay sample.cpp %}
e[i]   = ev[0];
e[i+1] = ev[1];
e[i+2] = ev[2];
e[i+3] = ev[3];
{% endcoderay %}

The second problem is this code is not generic; if you migrate from SSE to AVX, you'll have to update the
initialization of your wrapper so it takes 8 floats; the same for storing your vector4f in scalar results.

What we need here is a way to load float into vector4f and to store vector4f into floats that doesn't
depend on the size of vector4f (that is, 4).

### 3.3 Load from and store to memory

If you take a look at the emmintrin.h file, you'll notice load and store functions: _mm_


