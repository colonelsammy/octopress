---
layout: post
title: "Debugging Phil Nash's CATCH framework with gdb on Mingw32"
date: 2013-10-15 19:31
comments: false
categories: [catch, c++, TDD, gdb]
keywords: catch, c++, TDD, gdb
description: Debugging Phil Nash's CATCH framework with gdb on Mingw32
---
So you have Mingw (or gdb on Linux) and you'd like to use Phil Nash's CATCH testing framework.  This article will go through some simple tests that we can construct and how to use it in that environment.

Throughout this document I use a test project that I've setup in C:\Projects\catch_gdb_example - I'll use the Mingw directory /c/Projects/catch_gdb_example to access it.

For testing, I was using the latest CATCH with Mingw32:

```
$ uname -a
MINGW32_NT-6.1 machine 1.0.18(0.48/3/2) 2012-11-21 22:34 i686 Msys
```
and Git:
```
$ git --version
git version 1.8.3.msysgit.0
```

# First, CATCH your framework #

There are two easy ways to get CATCH, and one that is slightly harder, so guess which one I chose?  The easy way is download the source zip file, available from https://github.com/philsquared/Catch and unpack it - you can either use the full include headers (from `catch/include`) or the single header (from `catch/single_include`).

The alternative that I chose was to first download and install git for windows (http://msysgit.github.com/).  

Then I cloned Phil's repository with:
```
cd /c/Projects/catch_gdb_example
git clone https://github.com/philsquared/Catch.git
```

...this clones the CATCH framework and puts it into the 'Catch' sub--directory.

# Build a sanity check #

Next create a cpp file (e.g. main.cpp) with the following contents:

```
#define CATCH_CONFIG_MAIN
#include "catch.hpp"

TEST_CASE( "main/sanity check", "[sanity]" )
{
    REQUIRE(true);
}
```

Now a simple makefile:

```
CC=g++
CFLAGS=-c -DDEBUG -I./Catch/include -g -Wall -pedantic -Wextra -std=c++0x
LDFLAGS=
SOURCES= \
    main.cpp
    
OBJECTS=$(SOURCES:.cpp=.o)
EXECUTABLE=test.exe

all: $(SOURCES) $(EXECUTABLE)
	
$(EXECUTABLE): $(OBJECTS) 
	$(CC) $(LDFLAGS) $(OBJECTS) -o $@

.cpp.o:
	$(CC) $(CFLAGS) $< -o $@

clean:
	rm -rf *.o $(EXECUTABLE)
```

...and we can build the test with:
```
make
```

and run with:

```
User@machine /c/Projects/catch_gdb_example
$ make
g++ -c -DDEBUG -I./Catch/include -g -Wall -pedantic -Wextra -std=c++0x main.cpp -o main.o
g++  main.o -o test.exe

User@machine /c/Projects/catch_gdb_example
$ ./test.exe main/*
All tests passed (1 assertion in 1 test case)
```

# Now for the debugger #

Add the following failing test:

```
TEST_CASE("test/fails","[gdb]")
{
    int a = 1;
    REQUIRE(a == 0);
}
```
and recompile:

```
User@machine /c/Projects/catch_gdb_example
$ make
g++ -c -DDEBUG -I./Catch/include -g -Wall -pedantic -Wextra -std=c++0x main.cpp -o main.o
g++  main.o -o test.exe
```

Now you can run the test program and set it to break on the failure of test conditions:
<!-- more -->

```
$ gdb ./test
GNU gdb (GDB) 7.6.1
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "mingw32".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from c:\Projects\catch_gdb_example\test.exe...done.
(gdb)
```

Now you can run the (failing) test case:

```
(gdb) r -b
Starting program: c:\Projects\catch_gdb_example/./test.exe -b
[New Thread 8364.0x2398]
 
test.exe is a Catch v1.0 b10 host application.
Run with -? for options

-------------------------------------------------------------------------------
test/fails
-------------------------------------------------------------------------------
main.cpp:9
...............................................................................

main.cpp:12: FAILED:
  REQUIRE( a == 0 )
with expansion:
  1 == 0

 
Program received signal SIGTRAP, Trace/breakpoint trap.
0x7685321a in KERNELBASE!DebugBreak () from C:\windows\syswow64\KernelBase.dll
(gdb)
```

If you look at the stack you'll see that we aren't actually in the test function:

```
0x7685321a in KERNELBASE!DebugBreak () from C:\windows\syswow64\KernelBase.dll
(gdb) bt
#0  0x7685321a in KERNELBASE!DebugBreak () from C:\windows\syswow64\KernelBase.dll
#1  0x0040602d in ____C_A_T_C_H____T_E_S_T____9 () at main.cpp:12
#2  0x0041fcf9 in Catch::FreeFunctionTestCase::invoke (this=0xb72850)
    at ./Catch/include/internal/catch_test_case_registry_impl.hpp:103
#3  0x00403837 in Catch::TestCase::invoke (this=0xb72b10)
    at ./Catch/include/internal/catch_test_case_info.hpp:78
#4  0x00409621 in _fu32___ZSt4cerr () at ./Catch/include/internal/catch_runner_impl.hpp:277
#5  0x0040a066 in Catch::RunContext::runTest (this=0x28fbc8, testCase=...)
    at ./Catch/include/internal/catch_runner_impl.hpp:123
#6  0x00417473 in Catch::Runner::runTestsForGroup (this=0x28fcdc, context=..., filterGroup=...)
    at ./Catch/include/catch_runner.hpp:67
#7  0x00417744 in Catch::Runner::runTests (this=0x28fcdc) at ./Catch/include/catch_runner.hpp:48
#8  0x004182c2 in Catch::Session::run (this=0x28fe6c) at ./Catch/include/catch_runner.hpp:203
#9  0x004181fa in Catch::Session::run (this=0x28fe6c, argc=2, argv=0xb72dc8)
    at ./Catch/include/catch_runner.hpp:186
#10 0x00405959 in main (argc=2, argv=0xb72dc8) at ./Catch/include/internal/catch_default_main.hpp:15
(gdb)
```

So we need to go up 1 frame and then we can examine variables in the test:

```
(gdb) up
#1  0x0040602d in ____C_A_T_C_H____T_E_S_T____9 () at main.cpp:12
12          REQUIRE(a == 0);
(gdb) p a
$1 = 1
(gdb)
```

Oh look, 'a == 1', why did I think it should be 0?

That's all there is to it!  Continuing the run gives the expected CATCH output:

```
(gdb) c
Continuing.
===============================================================================
2 test cases - 1 failed (2 assertions - 1 failed)

[Inferior 1 (process 8364) exited with code 01]
(gdb)
```
