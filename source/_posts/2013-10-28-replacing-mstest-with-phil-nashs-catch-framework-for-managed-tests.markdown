---
layout: post
title: "Replacing MSTest with Phil Nash's Catch framework for managed C++ tests"
date: 2013-10-28 19:47
comments: false
categories: [catch, c++, TDD, MSTest]
keywords: catch, c++, TDD, MSTest, ManagedC++
description: Replacing MSTest with Phil Nash's Catch framework for managed C++ tests
---

Visual Studio 2010 has an option for creating C++ unit tests, but these tests are 'managed' C++.  There are a number of problems with such tests:

1. The tests are slow to compile and slow to load/run.
2. Tests tend to be more complicated than needed - the Managed C++ compiler does not allow C++ value types as member variables for example, so these have to be kept as pointers and new/delete used.
3. The debugger doesn't understand C++ types very well, so when in managed code it is often not possible to view object values. Whilst it could be argued that using TDD should avoid most uses of the debugger, sometimes it's inevitable.
4. Sometimes the Managed C++ debugger gets a 'mind of its own' and decides to run a test to completion part way through debugging. This can be frustrating!
5. Sometimes, the process vstest.executionengine.x86.exe gets left running after the tests have completed whcih prevents the any new builds from compiling as the DLL is held open by the executionengine.  There is [a bug report for this](https://connect.microsoft.com/VisualStudio/feedback/details/771994/vstest-executionengine-x86-exe-32-bit-not-closing-vs2012-11-0-50727-1-rtmrel "Bug report") but apparently no solution other than using taskkill as a pre-build event [http://stackoverflow.com/questions/13497168/vstest-executionengine-x86-exe-not-closing](http://stackoverflow.com/questions/13497168/vstest-executionengine-x86-exe-not-closing "Stack overflow answer")

In the absence of Native C++ unit tests (note: VS2012 has these - see later) I wanted to be able to share the code base between MSTest managed tests and native tests so that the build system could happily run MSTest on its managed build and I could run native tests using Catch.  This article describes my experiment to do this...

# Differences in test philosophy #

To persuade MSTest to look like Catch, we have to think about how the test needs to run.  MSTest **requires** a class, but Catch does not. Catch allows multiple runs through using SECTIONs, but this doesn't figure in MSTest. So my first attempt used a TEST_CASE as a 'class' wrapper and a SECTION for each test method.  This worked pretty well, except that it wasn't possible to specify individual tests to run (SECTIONs don't support tags).

Looking at the tests that I wanted convert, they tended to follow a pattern:

```c++
namespace ManagedTestProject
{
    [TestClass]
    public ref class ClassNameForTest
    {
        // some data that needs to be initialized...
        Stuff* m_data;

	    //Use TestInitialize to run code before running each test
	    [TestInitialize()]
	    void MyTestInitialize()
        {
            m_data = new Stuff(...);
        }

	    //Use TestCleanup to run code after each test has run
	    [TestCleanup()]
	    void MyTestCleanup()
        {
            delete m_data;
            m_data = NULL;
        }

    public:
	    [TestMethod]
        void Method1() {}

	    [TestMethod]
        void Method2() {}
    };
}
```

<!-- more -->

# A specific solution #

Fortunately, they always did this; declare pointer member variables for objects the test needed, create in TestInitialize and destroy in TestCleanup, in that order, followed by test methods. I realised that I could declare the data at namespace scope and wrap the initialisation/destruction in a helper class, something like this:

```c++
namespace x
{
    Stuff* m_data;
    void setup()
    {
        m_data = new Stuff();
    }
    void cleanup()
    {
        delete m_data;
        m_data = NULL;
    }

    struct setup_wrapper
    {
        setup_wrapper()
        {
            setup();
        }
        virtual ~setup_wrapper()
        {
            cleanup();
        }
    };
    
    TEST_CASE_METHOD(setup_wrapper, "Method1"); 
}
```
TEST_CASE_METHOD creates a class instance derived from setup_wrapper for each method, then runs the test in its own method. Unfortunately, not all my tests had TestInitiaize/TestCleanup methods, so I wanted to be able to specify those in a similar way to Native C++ tests. I tried several ways to do this, but ended up with a couple of macros, used like this:

```c++
    TEST_METHOD_INITIALIZE(setupMethodName)
    {
        m_data = new Stuff(...);
    }

    TEST_METHOD_CLEANUP(cleanupMethodName)
    {
        delete m_data; m_data = NULL;
    }
```
and declared like this:

```
#define TEST_METHOD_INITIALIZE(methodName) \
    void setup(int)

#define TEST_METHOD_CLEANUP(methodName) \
    void cleanup(int)
```

The wrapper class then calls each:

```c++
        setup_wrapper()
        {
            setup(0);
        }
        virtual ~setup_wrapper()
        {
            cleanup(0);
        }
```

If there isn't a setup method, there won't be a `setup(int)` or a `cleanup(int)` but we can fix that by defining a templated setup/cleanup function at namespace scope:

```c++
namespace X
{
    template <typename T>
    void setup(T t) { }
    template <typename T>
    void cleanup(T t) { }
    ...
    // no TEST_METHOD_INITIALIZE/CLEANUP
    struct setup_wrapper
    {
        setup_wrapper()
        {
            // calls template function
            setup(0);
        }
        virtual ~setup_wrapper()
        {
            // calls template function
            cleanup(0);
        }
    };
    
    TEST_CASE_METHOD(setup_wrapper, "Method1"); 
}
```
C++ overload resolution will always prefer the non-templated `setup(int)` if it exists, so will call the non-templated method.

# A final solution #

Now I can embed all the nasty bits in macros and mangle some names to avoid duplicate definitions so that my final shared codebase looks like this (code for this project can be found here):    

```c++
#include "stdafx.h"
#include "MSManagedTestMacros.hpp"

#include "lib.h"    // code under test, declares NativeTestClass

namespace ManagedTestProject {

    TEST_CLASS(TestWithoutInitializeAndCleanup)
    {
        TEST_CLASS_CONTEXT()

        MS_TEST_CASE_METHOD(Method1)
        {
            NativeTestClass* m_data = new NativeTestClass("Fred", 42);
            REQUIRE(m_data != NULL);
            std::string result = m_data->getName();
            REQUIRE(result == "Fred");
        }

        MS_TEST_CASE_METHOD(Method2)
        {
            NativeTestClass* m_data = new NativeTestClass("Fred", 42);
            REQUIRE(m_data != NULL);
            std::string result = m_data->getName();
            REQUIRE(result != "Fred");
        }
    };

    TEST_CLASS(TestWithInitializeAndCleanup)
    {
        TEST_CLASS_CONTEXT()

        NativeTestClass* m_data;

        TEST_METHOD_INITIALIZE(MyInitialize)
        {
            m_data = new NativeTestClass("Fred", 42);
        }
		
        TEST_METHOD_CLEANUP(MyCleanup)
        {
            delete m_data;
            m_data = NULL;
        }

        MS_TEST_CASE_METHOD(Method1)
        {
            REQUIRE(m_data != NULL);
            std::string result = m_data->getName();
            REQUIRE(result == "Fred");
        }

        MS_TEST_CASE_METHOD(Method2)
        {
            REQUIRE(m_data != NULL);
            std::string result = m_data->getName();
            REQUIRE(result != "Fred");
        }
    };

} // namespace 
```

Running this with MSTest gives this:

```
C:\Projects\ManagedTestProject>"c:\Program Files (x86)\Microsoft Visual Studio 11.0\Common7\IDE\MSTest.exe" /testcontain
er:Debug\DefaultTest.dll
Microsoft (R) Test Execution Command Line Tool Version 11.0.50727.1
Copyright (c) Microsoft Corporation. All rights reserved.

Loading Debug\DefaultTest.dll...
Starting execution...

Results               Top Level Tests
-------               ---------------
Passed                ManagedTestProject.TestWithInitializeAndCleanup.Method1
Failed                ManagedTestProject.TestWithInitializeAndCleanup.Method2
Passed                ManagedTestProject.TestWithoutInitializeAndCleanup.Method1
Failed                ManagedTestProject.TestWithoutInitializeAndCleanup.Method2
2/4 test(s) Passed, 2 Failed

Summary
-------
Test Run Failed.
  Passed  2
  Failed  2
  ---------
  Total   4
Results file:  C:\Projects\ManagedTestProject\TestResults\name_machine 2013-10-28 20_24_15.trx
Test Settings: Default Test Settings
```

Recompiling and running the same code with Catch gives:

```
C:\Projects\ManagedTestProject\Debug>CatchTestProject.exe

CatchTestProject.exe is a Catch v1.0 b11 host application.
Run with -? for options

-------------------------------------------------------------------------------
TestWithoutInitializeAndCleanup::Method2
-------------------------------------------------------------------------------
c:\projects\managedtestproject\unittestcatch.cpp(20)
...............................................................................

c:\projects\managedtestproject\unittestcatch.cpp(25): FAILED:
  REQUIRE( result != "Fred" )
with expansion:
  "Fred" != "Fred"

-------------------------------------------------------------------------------
TestWithInitializeAndCleanup::Method2
-------------------------------------------------------------------------------
c:\projects\managedtestproject\unittestcatch.cpp(53)
...............................................................................

c:\projects\managedtestproject\unittestcatch.cpp(57): FAILED:
  REQUIRE( result != "Fred" )
with expansion:
  "Fred" != "Fred"

===============================================================================
4 test cases - 2 failed (8 assertions - 2 failed)
```

Much more satisfactory...

# VS2012 and Native C++ Tests #

In VS2010, you can create a C++ test project like this:

<div style="float: center; margin: 10px;">
  <img src="http://www.graoil.co.uk/images/MSTest/VS2010TestProject.png?raw=true" alt="VS2010" Title="VS2010"/>
</div>

This is always a managed C++ project and the macros above work fine.  In VS2012, there are two options....but I think I'll leave that for another post ;-) 

Project used in this article:

[ManagedTestProject](http://www.graoil.co.uk/downloads/ManagedTestProject.zip "ManagedTestProject")
[NativeTestLibrary](http://www.graoil.co.uk/downloads/NativeTestLibrary.zip "NativeTestLibrary")
