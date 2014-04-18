---
layout: post
title: "Writing C++ wrappers for SIMD intrinsics (1)"
date: 2014-04-15 08:06:58 +0200
comments: true
categories: [SIMD,test]
published: false
---

This is a test

``` cpp
#include <iostream>
template <class T>
	class simd_vector
	{
	public:

		inline void do_the_stuff()
		{
			std::cout << "coincoin" << std::endl;
		}

		simd_vector& operator=(const simd_vector&)
		{
			return *this;
		}
	};

int main(int argc, char* argv[])
{
	return 1;
}
```

