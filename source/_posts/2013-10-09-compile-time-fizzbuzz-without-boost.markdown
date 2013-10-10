---
layout: post
title: "Compile time FizzBuzz without Boost"
date: 2013-10-09 05:53
comments: false
categories: [c++, templates, fizzbuzz]
keywords: c++, templates, fizzbuzz, boost
description: How to deconstruct compile time FizzBuzz without using Boost 
---
Several years ago [Adam Peterson published an article on how to implement FizzBuzz at cmpile time in C++](http://www.adampetersen.se/articles/fizzbuzz.htm "Adam Petersen's FizzBuzz"). The code was clever, but had a dependency on Boost and didn't go into great detail on how it worked, so I thought I'd write it again, from scratch and try to explain my workings.  I'm only going to show examples that would work for very small FizzBuzz sequences because, well, this is a learning exercise not a typing exercise! 

# First, get an error message #

To make this work, we first have persuade the compiler to print an error message to the console when we build the program, something like this:

```c++
struct void_;

template <class T0 = void_, class T1 = void_, class T2 = void_, class T3 = void_> 
struct vector
{
    typedef vector type;
    typedef T0 T0;
    typedef T1 T1;
    typedef T2 T2;
    typedef T3 T3;
};
```
When we compile this with VS2010, we get this:

```
1>ClCompile:
1>  main.cpp
1>c:\projects\boost_any_example\main.cpp(243): error C2039: 'compilation_error_here' : is not a member of 'fizzbuzz::vector<T0,T1,T2>'
1>          with
1>          [
1>              T0=int,
1>              T1=long,
1>              T2=double
1>          ]
1>c:\projects\boost_any_example\main.cpp(243): error C2146: syntax error : missing ';' before identifier 'res'
1>c:\projects\boost_any_example\main.cpp(243): error C4430: missing type specifier - int assumed. Note: C++ does not support default-int
1>c:\projects\boost_any_example\main.cpp(243): error C2065: 'res' : undeclared identifier
```
This is promising; it displays the types that we puit in, and in the right order.  That's fine, but the compiler works with types and we need to display numeric values, so we need a helper to convert intergers to types. [Loki](http://loki-lib.sourceforge.net/ "Loki library") calls this Int2Type and [Boost.MPL](http://www.boost.org/doc/libs/1_54_0/libs/mpl/doc/index.html "Boost.MPL") has mpl::int_ that does the same thing, but it's really easy to implement:

```c++
template <int N>
struct int_
{
    static const int value = N;
};
```
For each distinct value 'N', this template creates a distinct type, so int_<0> is a different type from int_<1> etc.

Now we can put this into our 'error' template:

```c++
typedef vector<int_<0>,int_<1>,int_<2> >::compilation_error_here res;
```
and we get error output with increasing numeric values, which is what we want:

```
1>c:\projects\boost_any_example\main.cpp(250): error C2039: 'compilation_error_here' : is not a member of 'fizzbuzz::vector<T0,T1,T2>'
1>          with
1>          [
1>              T0=fizzbuzz::int_<0>,
1>              T1=fizzbuzz::int_<1>,
1>              T2=fizzbuzz::int_<2>
1>          ]
```

# Now generate the sequence... #

That's fine, but now we need to generate the sequence (this largely follows the implementation of only those required bits of the 'tiny' sequence in chapter 5 of [C++ Template Metaprogramming](http://www.amazon.co.uk/Template-Metaprogramming-Concepts-Techniques-Beyond/dp/0321227255/ref=sr_1_1?ie=UTF8&qid=1381403602&sr=8-1&keywords=c%2B%2B+template+metaprogramming "C++ Template Metaprogramming")).

To get a new sequence, we have to append a new type (int_<1>) to an existing sequence (int_<0>). We can only do that if we know how big our current sequence is, so we first need to specialize our template for every possible size of sequence (this is why we're only working with small sequences!). Note that the size is derived from the helper template that we defined earlier so that we can get the 'value':

```c++
template <class T0, class T1, class T2, class T3>
struct size_impl : int_<4> {};

template <class T0, class T1, class T2>
struct size_impl<T0,T1,T2,void_> : int_<3> {};

template <class T0, class T1>
struct size_impl<T0,T1,void_,void_> : int_<2> {};

template <class T0>
struct size_impl<T0,void_,void_,void_> : int_<1> {};

template <>
struct size_impl<void_,void_,void_,void_> : int_<0> {};

template <class Sequence>
struct size : size_impl<typename Sequence::T0, typename Sequence::T1
    , typename  Sequence::T2, typename  Sequence::T3>
{};

```
 and now we can append a new type to another:

```c++
template <class Sequence, class T, int N>
struct push_back_impl;

template <class Sequence, class T>
struct push_back_impl<Sequence, T, 0>
    : vector<T, void_, void_, void_>
{};

template <class Sequence, class T>
struct push_back_impl<Sequence, T, 1>
    : vector<typename Sequence::T0, T, void_, void_>
{};

template <class Sequence, class T>
struct push_back_impl<Sequence, T, 2>
    : vector<typename Sequence::T0, typename Sequence::T1, T, void_>
{};

template <class Sequence, class T>
struct push_back_impl<Sequence, T, 3>
    : vector<typename Sequence::T0, typename Sequence::T1, typename Sequence::T2, T>
{};

template <class Sequence, class T>
struct push_back
    : push_back_impl<Sequence, T, size<Sequence>::value >
{};

```

Now we can use the recursive parts of Adam's RunFizzBuzz to generate our sequence:

```c++
template<int i>
struct RunFizzBuzz
{
    typedef int_<i> Number;
    typedef typename push_back<typename RunFizzBuzz<i - 1>::type, Number>::type type;
};

template<>
struct RunFizzBuzz<0>
{
    typedef vector<int_<0> > type;
};

int main()
{
    typedef RunFizzBuzz<3>::type::compilation_error_here res;
}
```

which gives us:

```
1>c:\projects\boost_any_example\main.cpp(306): error C2039: 'compilation_error_here' : is not a member of 'vector<T0,T1,T2,T3>'
1>          with
1>          [
1>              T0=int_<0>,
1>              T1=int_<1>,
1>              T2=int_<2>,
1>              T3=int_<3>
1>          ]
```

That looks a lot like what we want, but so far we havn't output Fizz or Buzz; we'll fix that now...

# Selecting Fizz #

if_c is a fairly simple type selection template that works by specializing for one value of the condition:

```c++
template <class T1, class T2, bool c>
struct if_c_impl
{
    typedef T1 type;
};

template <class T1, class T2>
struct if_c_impl<T1, T2, false>
{
    typedef T2 type;
};

template<bool c, class T1, class T2>
struct if_c : if_c_impl<T1,T2,c>
{};
```

Now we can add the condition for Fizz:
```c++
struct Fizz{};

template<int i>
struct RunFizzBuzz
{
    typedef int_<i> Number;
    typedef if_c<i % 3 == 0, Fizz, Number> condition3;
    typedef typename push_back<typename RunFizzBuzz<i - 1>::type, typename condition3::type>::type type;
};

template<>
struct RunFizzBuzz<0>
{
    typedef vector<int_<0> > type;
};
```

, which produces:

```
1>c:\projects\boost_any_example\main.cpp(327): error C2039: 'compilation_error_here' : is not a member of 'vector<T0,T1,T2,T3>'
1>          with
1>          [
1>              T0=int_<0>,
1>              T1=int_<1>,
1>              T2=int_<2>,
1>              T3=Fizz
1>          ]
```

which is exactly what we want!

Adding the additonal conditions for Buzz and FizzBuzz is trivial (as long as we remember to test for FizzBuzz first!):

```
struct Fizz{};
struct Buzz{};
struct FizzBuzz{};

template<int i>
struct RunFizzBuzz
{
    typedef int_<i> Number;
    typedef typename if_c<i % 3 == 0, Fizz, Number>::type condition1;
    typedef typename if_c<(i % 5 == 0), Buzz, condition1>::type condition2;
    typedef typename if_c<(i % 3 == 0) && (i % 5 == 0), FizzBuzz, condition2>::type condition3;
    typedef typename push_back<typename RunFizzBuzz<i - 1>::type, condition3>::type type;
};

template<>
struct RunFizzBuzz<0>
{
    typedef vector<int_<0> > type;
};
```

```
1>c:\projects\boost_any_example\main.cpp(305): error C2039: 'compilation_error_here' : is not a member of 'vector<T0,T1,T2,T3,T4,T5,T6,T7,T8,T9,T10,T11,T12,T13,T14,T15>'
1>          with
1>          [
1>              T0=int_<0>,
1>              T1=int_<1>,
1>              T2=int_<2>,
1>              T3=Fizz,
1>              T4=int_<4>,
1>              T5=Buzz,
1>              T6=Fizz,
1>              T7=int_<7>,
1>              T8=int_<8>,
1>              T9=Fizz,
1>              T10=Buzz,
1>              T11=int_<11>,
1>              T12=Fizz,
1>              T13=int_<13>,
1>              T14=int_<14>,
1>              T15=FizzBuzz
1>          ]
```

Just for completeness, it works on gcc (MingW) too:

```
$ gcc --version
gcc.exe (GCC) 4.8.1
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ make
g++ -c -Wall main.cpp -o main.o
main.cpp: In function 'int main()':
main.cpp:306:13: error: 'compilation_error_here' in 'RunFizzBuzz<15>::type {aka struct vector<int_<0>, int_<1>, int_<2>,
 Fizz, int_<4>, Buzz, Fizz, int_<7>, int_<8>, Fizz, Buzz, int_<11>, Fizz, int_<13>, int_<14>, FizzBuzz>}' does not name
a type
     typedef RunFizzBuzz<15>::type::compilation_error_here res;
             ^
make: *** [main.o] Error 1
```

You can find the code used in this article [here](http://www.graoil.co.uk/downloads/fizzbuzz_example.zip "FizzBuzz example").
