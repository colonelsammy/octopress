---
layout: post
title: "Using Catch for Jeff Langr's TDD book chapter 2"
date: 2013-12-30 16:12
comments: false
categories: [catch, TDD]
keywords: catch, tdd
description: Using Catch for Jeff Langr's "Modern C++ Programming with Test Driven Development" book, chapter 2
---

As I'm a bit of a fan of Phil Nash's Catch test framework I decided to convert the code in Jeff Langr's book to use Catch.  This post describes the changes that were required for the first tests, i.e. chapter 2.

[Code for this project is here...](https://github.com/colonelsammy/c2.git "Chapter 2")

## Basic changes ##

Catch is header only so is much simpler to incorporate than Google Mock/Test so the CMakeLists.txt can be much simpler.  For the start of the chapter, this is sufficient:

```
project(chapterFirstExample)
cmake_minimum_required(VERSION 2.6)

include_directories(../../Catch/include)
add_definitions(-std=c++0x)
set(CMAKE_CXX_FLAGS "${CMAXE_CXX_FLAGS} -Wall")

set(sources
   main.cpp
   SoundexTest.cpp)
add_executable(test ${sources})
target_link_libraries(test pthread)
```

All we need to do is change the path to the Catch include directory, remove the link path and remove the link libraries.

Similarly, main.cpp is also simpler:

```c++
#define CATCH_CONFIG_MAIN
#include "catch.hpp"
```

Strictly speaking we don't need main.cpp at all, but it simplifies the test code to keep it.

<!-- more -->

## First 4 examples ##

The first examples are not very challenging and all that needs to be changed is to `#include "catch.hpp"` and change the TEST macro for a TEST_CASE with quotes around the identifiers, e.g. example 4:

```c++
#include <string>

class Soundex
{
public:
   std::string encode(const std::string& word) const {
      return "";
   }
};

#include "catch.hpp"

TEST_CASE("SoundexEncoding", "RetainsSoleLetterOfOneLetterWord") {
   Soundex soundex;

   auto encoded = soundex.encode("A");
}
```

## Examples 5-7 ##

Catch provides a much richer assertion mechanism so we can replace the ASSERT_THAT macro with a REQUIRE macro:

```c++
//...
TEST_CASE("SoundexEncoding", "RetainsSoleLetterOfOneLetterWord") {
   Soundex soundex;

   auto encoded = soundex.encode("A");

   REQUIRE(encoded == "A");
}
```
Running this for example 5 gives the following output in Catch:
```
test.exe is a Catch v1.0 b23 host application.
Run with -? for options

-------------------------------------------------------------------------------
SoundexEncoding
-------------------------------------------------------------------------------
c:/Projects/git/jlangr/c2/SoundexTest.cpp:18
...............................................................................

c:/Projects/git/jlangr/c2/SoundexTest.cpp:24: FAILED:
  REQUIRE( encoded == "A" )
with expansion:
  "" == "A"

===============================================================================
1 test case - failed (1 assertion - failed)
```
As you can see, the output provides both the original expression and the reconstructed expression with the values the caused the assertion failure.  Much better - Jeff's explanation would have been much shorter with this!

The fix in example 6 gives this output:

```
All tests passed (1 assertion in 1 test case)
```

In Catch, example 7 id identical to example 6 so has been removed.

## Examples 9 ##

This requires the same type of changes as above, except that the test case name needs to be unique...we'll fix that in example 10.

## Example 10,11 ##

We can use Catch SECTIONs to simplify this example...so we don't need the SoundexEncoding class at all, and the test looks like this:

```c++
TEST_CASE("SoundexEncoding", "RetainsSoleLetterOfOneLetterWord") {
   Soundex soundex;

   SECTION("RetainsSoleLetterOfOneLetterWord") {
      auto encoded = soundex.encode("A");

      REQUIRE(encoded == "A000");
   }

   SECTION("PadsWithZerosToEnsureThreeDigits") {
      auto encoded = soundex.encode("I");

      REQUIRE(encoded == "I000");
   }
}
```

Catch will run the TEST_CASE for each SECTION, automatically initialising the soundex object on each run.

Example 11 is similar.

## Examples 12-17 ##

Example 12 requires similar changes, but fails (as expected):

```
-------------------------------------------------------------------------------
SoundexEncoding
  ReplacesConsonantsWithAppropriateDigits
-------------------------------------------------------------------------------
c:/Projects/git/jlangr/c2/SoundexTest.cpp:7
...............................................................................

c:/Projects/git/jlangr/c2/SoundexTest.cpp:19: FAILED:
  REQUIRE( soundex.encode("Ab") == "A100" )
with expansion:
  "Ab000" == "A100"
```

Again, the output from Catch is much more helpful for diagnosing the error, providing the TEST_CASE name, the SECTION and the expansions.

Examples 13-17 follow the same pattern.

## Examples 18-22 ##

Examples 18-22 are similar but we can use the Catch CHECK() macro instead of EXPECT_THAT(), e.g.:

```c++
   //...
   SECTION("ReplacesConsonantsWithAppropriateDigits") {
      CHECK(soundex.encode("Ab") == "A100");
      CHECK(soundex.encode("Ac") == "A200");
   }
```

## Examples 23-25 ##

In example 23, Jeff disables a test.  We can't do the same thing with a Catch SECTION (yet...) but we can disable an entire test by using the '[hide]' tag, so we temporarily put the test in a TEST_CASE on its own:

```c++
TEST_CASE("ReplacesMultipleConsonantsWithDigits", "[hide]") {
   Soundex soundex;
   REQUIRE(soundex.encode("Acdl") == "A234");
}
```

## Example 26-31 ##

In passing, note that Catch also detects the exception:

```
SoundexEncoding
  LimitsLengthToFourCharacters
-------------------------------------------------------------------------------
c:/Projects/git/jlangr/c2/SoundexTest.cpp:5
...............................................................................

c:/Projects/git/jlangr/c2/SoundexTest.cpp:25: FAILED:
  REQUIRE( soundex.encode("Dcdlb").length() == 4u )
due to unexpected exception with message:
  basic_string::_S_create
```

## Example 32 ##

To use StartsWith in Catch, we must use the REQUIRE_THAT() macro, e.g.:

```c++
   SECTION("UppercasesFirstLetter") {
      REQUIRE_THAT(soundex.encode("abcd"), StartsWith("A"));
   }
```

## Examples 33-39 ##

These all follow the same pattern as above.
