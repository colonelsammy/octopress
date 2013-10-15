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

Throughout this document I use a test project that I've setup in C:\Projects\catch - I'll use the Mingw directory /c/Projects/catch to access it.

For testing, I was using the latest CATCH with Mingw32:

```
MINGW32_NT-6.1 MACHINE 1.0.17(0.48/3/2) 2011-04-24 23:39 i686 Msys
```
and Git:
```
git version 1.7.9.msysgit.0
```

# First, CATCH your framework #

There are two easy ways to get CATCH, and one that is slightly harder, so guess which one I chose?  The easy ways are either to download the 'single header', catch.hpp or download the source zip file, both available from https://github.com/philsquared/Catch.

The alternative that I chose was to first download and install git for windows (http://msysgit.github.com/).  

Then I cloned Phil's repository with:
```
cd /c/Projects/catch
git clone https://github.com/philsquared/Catch.git
```

...this clones the CATCH framework and puts it into the 'Catch' sub--directory.

# Build a sanity check #

Next create a cpp file (e.g. main.cpp) with the following contents:

```
#define CATCH_CONFIG_MAIN
#include "catch.hpp"

TEST_CASE( "main/sanity check", "The test case must *always* be true" )
{
    REQUIRE(true);
}
```

Now a simple makefile:

```
CC=g++
CFLAGS=-c -DDEBUG -I./Catch/include -g -Wall -pedantic -Wextra
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
User@machine /c/Projects/catch/catch
$ make
g++ -c -DDEBUG -I./Catch/include -g -Wall -pedantic -Wextra main.cpp -o main.o
g++  main.o -o test.exe

User@machine /c/Projects/catch/catch
$ ./test -t "main/sanity check"

[Testing completed. All tests passed (1 assertion in 1 test case)]
```

# Now for the debugger #

Add the following failing test:

```
TEST_CASE("test/fails","")
{
    int a = 1;
    REQUIRE(a == 0);
}
```
, recompile and start gdb:

```
User@machine /c/Projects/catch/catch
$ make
g++ -c -I./Catch/include -g -Wall -pedantic -Wextra main.cpp -o main.o
g++  main.o -o test.exe

User@machine /c/Projects/catch/catch
$ gdb ./test
GNU gdb (GDB) 7.4
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-pc-mingw32".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from c:\Projects\catch\catch\test.exe...done.
(gdb)
```
<!-- more -->
Now you can run the test program and set it to break on the failure of test conditions:

```
$ gdb ./test
GNU gdb (GDB) 7.4
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-pc-mingw32".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from c:\Projects\catch\catch\test.exe...done.
(gdb)
```

Now you can run the (failing) test case:

```
(gdb) r -b
Starting program: c:\Projects\catch\catch\test.exe -b
[New Thread 6688.0x1ba4]
[Started testing]
[Started group: 'test case run']

[Running: test/fails]
main.cpp:47: a == 0 failed for: 1 == 0

Program received signal SIGTRAP, Trace/breakpoint trap.
0x756e3e2f in KERNELBASE!DeleteAce () from C:\Windows\system32\KernelBase.dll
(gdb)
```

If you look at the stack you'll see that we aren't actually in the test function:

```
(gdb) bt
#0  0x756e3e2f in KERNELBASE!DeleteAce () from C:\Windows\system32\KernelBase.dll
#1  0x004040b6 in TestCaseFunction_catch_internal_44 () at main.cpp:47
#2  0x00413576 in Catch::FreeFunctionTestCase::invoke (this=0x382fe8)
    at ./Catch/include/internal/catch_test_case_registry_impl.hpp:115
#3  0x004132e8 in Catch::TestCaseInfo::invoke (this=0x381894) at ./Catch/include/internal/catch_test_case_info.hpp:97
#4  0x0040e657 in _fu2___ZSt4cerr () at ./Catch/include/internal/catch_runner_impl.hpp:358
#5  0x0040eb6c in Catch::Runner::runTest (this=0x22fcac, testInfo=...)
    at ./Catch/include/internal/catch_runner_impl.hpp:151
#6  0x0040ea14 in Catch::Runner::runAll (this=0x22fcac, runHiddenTests=false)
    at ./Catch/include/internal/catch_runner_impl.hpp:105
#7  0x0040d7e3 in _fu6___ZSt4cerr () at ./Catch/include/catch_runner.hpp:59
#8  0x0040dc0a in _fu7___ZSt4cerr () at ./Catch/include/catch_runner.hpp:130
#9  0x0040daec in Catch::Main (argc=2, argv=0x381000) at ./Catch/include/catch_runner.hpp:143
#10 0x00401ba1 in main (argc=2, argv=0x381000) at ./Catch/include/internal/catch_default_main.hpp:34
```

So we need to go up 1 frame and then we can examine varables in the test:

```
(gdb) up
#1  0x004040b6 in TestCaseFunction_catch_internal_44 () at main.cpp:47
47          REQUIRE(a == 0);
(gdb) p a
$1 = 1
(gdb)
```

and that's all there is to it!  Continuing the run gives the expected CATCH output:

```
(gdb) c
Continuing.
[End of group: 'test case run'. 0 test cases failed]

[Finished: 'test/fails' 1 test case failed (1 assertion failed)]

[Testing completed. 1 of 4 test cases failed (1 of 8 assertions failed)]

[Inferior 1 (process 6688) exited with code 01]
(gdb)
```
