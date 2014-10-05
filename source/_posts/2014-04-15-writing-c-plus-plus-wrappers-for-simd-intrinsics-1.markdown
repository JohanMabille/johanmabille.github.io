---
layout: post
title: "Writing C++ wrappers for SIMD intrinsics (1)"
date: 2014-04-15 08:06:58 +0200
comments: true
categories: [SIMD,test]
published: false
---

This is a test

{% coderay lang:cpp test %}
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


void testing_function()
{
	// This is an really really long comment for debugging horizontal scrollbars and to be sure they appear ; maybe we need the same for vertical ones.
}
int main(int argc, char* argv[])
{
	double val = rand();

	simd_vector<double> sd;
	if(val > 0.2)
	{
		// test comment
		val +=0.5;
	}
	else
	{
		val -= 0.2;
	}
	return 1;
}
{% endcoderay %}
end of the code test

void testing_function()
{
	// This is an really really long comment for debugging horizontal scrollbars and to be sure they appear ; maybe we need the same for vertical ones.
}

