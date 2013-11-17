---
layout: post
title: "Integrating Phil Nash's Catch C++ Unit Test Framework into Visual Studio"
date: 2013-11-17 19:44
comments: false
categories: [catch, c++, TDD, Visual Studio]
keywords: catch, c++, TDD, MSTest, ManagedC++, VS2010, VS2012
description: Integrating Phil Nash's Catch C++ Unit Test Framework into Visual Studio 
---

After I adapted some existing code that used MSTest so that I was able to use Catch, I started to look at whether it was possible to do better; specifically I wanted to share source code between VS and Catch written for a console app and I wanted to retain the ability to generate standard VS TestResults files (*.trx).

The results were better than I expected; I found it was possible, with a little adaptation, to change the Catch internals so that I could do all that; so much so that I was able to reimplement the Catch self tests as Managed C++ test projects for VS2010 and VS2012 as well as Native C++ test projects.

[You can find a fork of Catch here](https://github.com/colonelsammy/Catch "Catch fork") that implements this; hopefully Phil and I will be able to integrate this into the Catch mainline soon but until that is done I will do my best to keep the fork in sync with [the code here](https://github.com/philsquared/Catch "Catch main").

You can find [details on how to use it here](https://github.com/colonelsammy/Catch/blob/master/docs/vs/vs-index.md "VS integration details") and it looks like this:

VS2010: 
<div style="float: center; margin: 10px;">
  <img src="http://www.graoil.co.uk/images/MSTest/VS2010failingtest.png?raw=true" alt="VS2010" Title="VS2010"/>
</div>

VS2012:
<div style="float: center; margin: 10px;">
  <img src="http://www.graoil.co.uk/images/MSTest/VS2012failingtest.png?raw=true" alt="VS2012" Title="VS2012"/>
</div>
 