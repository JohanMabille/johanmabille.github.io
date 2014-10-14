---
layout: post
title: "Writing C++ wrappers for SIMD intrinsics (4)"
date: 2014-10-13 22:46:01 +0200
comments: true
categories: [SIMD,Vectorization]
published: false
---

## 3. Plugging the wrappers into existing code

### 3.1 Storing vector4f in vectors

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

A first solution would be to store vector of vector4f instead of vector of float:

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
for the smallest element of the vectoThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vectorThe problem is you could need to work with the scalar instead of the vector4f, for instance if you search
for the smallest element of the vecto
must reset the element toorr
