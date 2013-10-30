---
layout: post
title: "Replacing MSTest with Phil Nash's Catch framework for VS2012 native C++ tests"
date: 2013-10-29 19:31
comments: false
categories: [catch, c++, TDD, MSTest]
keywords: catch, c++, TDD, MSTest, ManagedC++
description: Replacing MSTest with Phil Nash's Catch framework for VS2012 native C++ tests
---
In [an earlier post I described how to share the same codebase for MSTest managed C++ code and Phil Nash's Catch framework]({% post_url 2013-10-28-replacing-mstest-with-phil-nashs-catch-framework-for-managed-tests %}).  In VS2012, there's an option for 'Native' C++ tests; this article describes how to use Catch to share the codebase for Native tests too.

# VS2012 and Native C++ Tests #

In VS2012, you can create a managed C++ test project like this:

<div style="float: center; margin: 10px;">
  <img src="http://www.graoil.co.uk/images/MSTest/ManagedProject1.png?raw=true" alt="Managed C++" Title="Managed C++"/>
</div>

You can also create a Native test, like this:

<div style="float: center; margin: 10px;">
  <img src="http://www.graoil.co.uk/images/MSTest/NativeProject1.png?raw=true" alt="Native C++" Title="Native C++"/>
</div>

In my case, a native test looks something like this:

```c++
namespace NativeTestProject
{		
	TEST_CLASS(UnitTest1)
	{
        NativeTestClass* m_data;
	public:
		UnitTest1()
        {}
        ~UnitTest1()
        {
        }
        TEST_METHOD_INITIALIZE(TestInit)
        {
            m_data = new NativeTestClass("Fred", 42);
        }
        TEST_METHOD_CLEANUP(TestCleanup)
        {
            delete m_data;
            m_data = NULL;
        }

		TEST_METHOD(TestMethod1)
		{
            ...
        }
```

<!-- more -->

I'm not using the constructor/destructor for initialisation and cleanup; instead I'm using the TEST_METHOD_INITIALIZE/CLEANUP macros that call setup and cleanup code before each method is called. This is very similar to my previous article...in fact it's so similar that all I need to do is redefine my macro for MS_TEST_CASE_METHOD and define REQUIRE/FAIL and I'm done!

We need a different test runner for native tests:

```
C:\Projects\NativeTestProject>"C:\Program Files (x86)\Microsoft Visual Studio 11.0\Common7\IDE\CommonExtensions\Microsof
t\TestWindow\vstest.console.exe" /Logger:Trx debug\NativeTestProject.dll
Microsoft (R) Test Execution Command Line Tool Version 11.0.60315.1
Copyright (c) Microsoft Corporation.  All rights reserved.

Starting test execution, please wait...
Passed   Method1
Failed   Method2
Error Message:
   Assert failed
Stack Trace:
        at ManagedTestProject::TestWithoutInitializeAndCleanup::Method2() in c:\projects\nativetestproject\unittestcatch
.cpp:line 23
Passed   Method1
Failed   Method2
Error Message:
   Assert failed
Stack Trace:
        at ManagedTestProject::TestWithInitializeAndCleanup::Method2() in c:\projects\nativetestproject\unittestcatch.cp
p:line 53
Results File: C:\Projects\NativeTestProject\TestResults\name_machine 2013-10-28 20_59_53.trx

Total tests: 4. Passed: 2. Failed: 2. Skipped: 0.
Test Run Failed.
Test execution time: 0.8892 Seconds
```

The code is identical to the managed/Catch example, so Catch works just the same:

```
C:\Projects\NativeTestProject\Debug>CatchTestProject.exe

CatchTestProject.exe is a Catch v1.0 b11 host application.
Run with -? for options

-------------------------------------------------------------------------------
TestWithoutInitializeAndCleanup::Method2
-------------------------------------------------------------------------------
c:\projects\nativetestproject\unittestcatch.cpp(18)
...............................................................................

c:\projects\nativetestproject\unittestcatch.cpp(23): FAILED:
  REQUIRE( result != "Fred" )
with expansion:
  "Fred" != "Fred"

-------------------------------------------------------------------------------
TestWithInitializeAndCleanup::Method2
-------------------------------------------------------------------------------
c:\projects\nativetestproject\unittestcatch.cpp(49)
...............................................................................

c:\projects\nativetestproject\unittestcatch.cpp(53): FAILED:
  REQUIRE( result != "Fred" )
with expansion:
  "Fred" != "Fred"

===============================================================================
4 test cases - 2 failed (8 assertions - 2 failed)
```

Project used in this article:

[NativeTestProject](http://www.graoil.co.uk/downloads/MSTest/NativeTestProject.zip "NativeTestProject")
[NativeTestLibrary](http://www.graoil.co.uk/downloads/MSTest/NativeTestLibrary.zip "NativeTestLibrary")
