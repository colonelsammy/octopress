---
layout: post
title: "Installing Ruby 1.9.3 in Windows with MingW"
date: 2013-09-26 17:32
comments: false
categories: [ruby, mingw, windows]
keywords: ruby, mingw, windows
description: How to install ruby 1.9.3 with Mingw on Windows
---

This is the second of three posts describing how to install Ruby 1.9.3 native MingW environment.  If you 're wondering why I did this, I wanted to use redcarpet/github-linguist for markdown in Octopress; there's a dependency on the charlock_holmes Gem that requires native code libraries (file, ICU), some of which it needs to build.  I couldn't figure out how to make the Ruby Installer do this and I wanted to understand the process, so I did the whole thing from scratch.  The [first part is here]({% post_url 2013-09-24-setting-up-mingw %}) and [the final part is here]({% post_url 2013-09-27-installing-octopress-in-windows-with-redcarpet %}).

[Setup Msys base system.]({% post_url 2013-09-24-setting-up-mingw %} "Mingw base system")

Download and build libyaml. Yaml requires a built DLL and needs it to be available so we build that manually ([instructions for hacking a ruby gem very useful knowledge for this](http://jonforums.github.io/ruby/2012/02/24/hacking-a-gem.html "Hacking a Ruby Gem")). Not sure if the first part to build and install the static libraries is required.  We're hard coding the target location, but just need to get it to work...:
```
cd /build32
wget http://pyyaml.org/download/libyaml/yaml-0.1.4.tar.gz
tar zxvf yaml-0.1.4.tar.gz
cd yaml-0.1.4
./configure --prefix=/mingw CFLAGS="-DYAML_DECLARE_STATIC" 
make && make install
cd src
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT api.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT dumper.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT emitter.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT loader.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT parser.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT reader.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT scanner.c
gcc -c -I../include -I../ -DHAVE_CONFIG_H -DYAML_DECLARE_EXPORT writer.c
gcc -shared -o yaml.dll api.o dumper.o emitter.o loader.o parser.o reader.o scanner.o writer.o -Wl,--out-implib,libyaml.dll.a -Wl,--output-def,libyaml.def
cp yaml.dll /mingw/bin
cp libyaml.def libyaml.dll.a /mingw/lib
cd ..
```
Download and build libffi:
```
cd /build32
wget ftp://sourceware.org/pub/libffi/libffi-3.0.13.tar.gz
wget http://www.linuxfromscratch.org/patches/blfs/svn/libffi-3.0.13-includedir-1.patch
tar zxvf libffi-3.0.13.tar.gz
cd libffi-3.0.13
patch -Np1 < ../libffi-3.0.13-includedir-1.patch
./configure --prefix=/mingw
make && make install
```
Download and build gdbm
```
cd /build32
wget ftp://ftp.gnu.org/gnu/gdbm/gdbm-1.10.tar.gz
tar zxvf gdbm-1.10.tar.gz
wget https://raw.github.com/luislavena/knapsack-recipes/master/gdbm/1.10/0001-Mingw-port-of-gdbm-1.10.patch --no-check-certificate
cd gdbm-1.10
patch -p1 < ../0001-Mingw-port-of-gdbm-1.10.patch
./configure --prefix=/mingw --enable-shared
make && make install
```
Download and build termcap:
```
cd /build32
wget http://ftp.gnu.org/gnu/termcap/termcap-1.3.1.tar.gz
tar zxvf termcap-1.3.1.tar.gz
cd termcap-1.3.1
./configure --prefix=/mingw --enable-static
make && make install
```
Download and build readline:
```
cd /build32
wget ftp://ftp.cwru.edu/pub/bash/readline-6.2.tar.gz
tar zxvf readline-6.2.tar.gz
cd readline-6.2
```
Patch rldefs.h and util.c to make params const char*.  Patch _rl_strnicmp in util.c so that do{}while has trailing ';':
```
diff -rupN readline-6.2.orig//rldefs.h readline-6.2//rldefs.h
--- readline-6.2.orig//rldefs.h	2009-01-04 19:32:33 +0000
+++ readline-6.2//rldefs.h	2013-09-19 09:38:32 +0100
@@ -79,8 +79,8 @@ extern char *strchr (), *strrchr ();
 #define _rl_stricmp strcasecmp
 #define _rl_strnicmp strncasecmp
 #else
-extern int _rl_stricmp PARAMS((char *, char *));
-extern int _rl_strnicmp PARAMS((char *, char *, int));
+extern int _rl_stricmp PARAMS((const char *, const char *));
+extern int _rl_strnicmp PARAMS((const char *, const char *, int));
 #endif
 
 #if defined (HAVE_STRPBRK) && !defined (HAVE_MULTIBYTE)
diff -rupN readline-6.2.orig//util.c readline-6.2//util.c
--- readline-6.2.orig//util.c	2010-05-30 23:36:02 +0100
+++ readline-6.2//util.c	2013-09-19 09:38:57 +0100
@@ -369,10 +369,10 @@ _rl_strpbrk (string1, string2)
    doesn't matter (strncasecmp). */
 int
 _rl_strnicmp (string1, string2, count)
-     char *string1, *string2;
+     const char *string1, *string2;
      int count;
 {
-  register char *s1, *s2;
+  register const char *s1, *s2;
   int d;
 
   if (count <= 0 || (string1 == string2))
@@ -389,7 +389,7 @@ _rl_strnicmp (string1, string2, count)
         break;
       s2++;
     }
-  while (--count != 0)
+  while (--count != 0);
 
   return (0);
 }
@@ -397,9 +397,9 @@ _rl_strnicmp (string1, string2, count)
 /* strcmp (), but caseless (strcasecmp). */
 int
 _rl_stricmp (string1, string2)
-     char *string1, *string2;
+     const char *string1, *string2;
 {
-  register char *s1, *s2;
+  register const char *s1, *s2;
   int d;
 
   s1 = string1;
```
Build:
```
patch -p1 < ../readline-6.2.patch
./configure --prefix=/mingw --enable-static
make && make install
```
Download and install openssl:
```
cd /build32
wget http://www.openssl.org/source/openssl-1.0.1e.tar.gz
tar zxvf openssl-1.0.1e.tar.gz
cd openssl-1.0.1e
perl Configure mingw no-shared no-asm --prefix=/mingw
make depend
make
make install
```
Download and install ruby (note: dbm, pty and syslog will fail to configure, this is ok on Windows...no others should fail).  For some reason we need the drive/path to the install location or it doesn't install any files...:
```
cd /build32
wget http://cache.ruby-lang.org/pub/ruby/1.9/ruby-1.9.3-p448.tar.gz
tar zxvf ruby-1.9.3-p448.tar.gz
cd ruby-1.9.3-p448
./configure --prefix=C:/mingw/local32 --enable-shared --disable-install-doc
make
make install
```
Test...should output the version, e.g.:
```
$ ruby -v
ruby 1.9.3p448 (2013-06-27 revision 41675) [i386-mingw32]
```
Download and install rubygems:
```
cd /build32
wget http://production.cf.rubygems.org/rubygems/rubygems-2.0.7.tgz
tar zxvf rubygems-2.0.7.tgz
cd rubygems-2.0.7
ruby setup.rb 
```
Test ([final part of this useful post](http://phosphor-escence.blogspot.co.uk/2011/08/clean-installation-ruby-193-preview1.html "Clean installation Ruby 1.9.3")):
```
$ gem list omniauth$ -r --all

*** REMOTE GEMS ***

omniauth (1.1.4, 1.1.3, 1.1.2, 1.1.1, 1.1.0, 1.0.3, 1.0.2, 1.0.1, 1.0.0, 0.3.2, 0.3.0, 0.2.6, 0.2.5, 0.2.4, 0.2.3, 0.2.2, 0.2.1, 0.2.0, 0.1.6, 0.1.5, 0.1.4, 0.1.3, 0.1.2, 0.1.1, 0.1.0, 0.0.5, 0.0.4, 0.0.3, 0.0.1)
```
It works!