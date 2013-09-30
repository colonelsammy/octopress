---
layout: post
title: "Setting up Mingw ready for Ruby 1.9.3 install"
date: 2013-09-24 21:31
comments: false
categories: [mingw, windows]
keywords: mingw, windows
description: How to install base Mingw on Windows ready to install Ruby
---
This is the first of three posts describing how to install Ruby 1.9.3 native MingW environment.  If you 're wondering why I did this, I wanted to use redcarpet/github-linguist for markdown in Octopress; there's a dependency on the charlock_holmes Gem that requires native code libraries (file, ICU), some of which it needs to build.  I couldn't figure out how to make the Ruby Installer do this and I wanted to understand the process, so I did the whole thing from scratch.  The [second part is here]({% post_url 2013-09-26-installing-ruby-in-windows %}) and [the final part is here]({% post_url 2013-09-27-installing-octopress-in-windows-with-redcarpet %}).
 
This is a prerequisite for installing Ruby 1.9.3 in a native Mingw environment on Windows.

Download and run mingw installer: mingw-get-setup.exe
Select 'mingw-developer-toolkit' and apply changes. This installs a default msys and default packages.
Start msys.bat (usually in C:\MinGW\msys\1.0).

Edit /etc/fstab to add:
```
c:/mingw        /mingw
c:/mingw/local32        /local32
c:/mingw/build32        /build32
```
Create a directory for app install and a build directory (similar to [instructions for installing a base system](http://ingar.satgnu.net/devenv/mingw32/base.html "instructions for installing a base system"))
```
mkdir /usr/local
mkdir /usr/local/bin
mkdir /c/mingw/opt
mkdir /c/mingw/build32 /c/mingw/local32
mkdir /c/mingw/opt/bin /local32/{bin,etc,include,lib,share}
mkdir /local32/lib/pkgconfig
```
Create a profile:
```
cat > /local32/etc/profile.local << "EOF"
#
# /local32/etc/profile.local
#

alias dir='ls -la --color=auto'
alias ls='ls --color=auto'

PKG_CONFIG_PATH="/local32/lib/pkgconfig"
CPPFLAGS="-I/local32/include"
CFLAGS="-I/local32/include"
CXXFLAGS="-I/local32/include"
LDFLAGS="-L/local32/lib"
export PKG_CONFIG_PATH CPPFLAGS CFLAGS CXXFLAGS LDFLAGS

PATH="/local32/bin:$PATH"

# package build directory
LOCALBUILDDIR=/build32
# package installation prefix
LOCALDESTDIR=/local32
export LOCALBUILDDIR LOCALDESTDIR

EOF
```
and a user profile to load it:
```
cat >> /etc/profile << "EOF"
if [ -f /local32/etc/profile.local ]; then
        source /local32/etc/profile.local
fi

EOF
```
Remove unwanted automake's ([following these Mingw install instructions](http://puredata.info/docs/developer/WindowsMinGW "Mingw install instructions")):
```
mingw-get remove automake1.4 automake1.5 automake1.6 automake1.7 automake1.8
```
'exit' from all msys shells and restart.  You should see the following mounts:
```
$ mount
...
c:\mingw\build32 on /build32 type user (binmode)
c:\mingw\local32 on /local32 type user (binmode)
c:\mingw on /mingw type user (binmode)
...
```
Add some additional packages:
```
mingw-get install base bzip2 expat gcc-g++ gettext gmp libarchive libpdcurses libpopt libunistring lua mingw-utils mpc mpfr pdcurses pkginfo pthreads-w32 zlib

mingw-get install msys-bash msys-bison msys-core-dev msys-crypt msys-cygutils msys-inetutils msys-unzip msys-wget msys-zip  
```
(optional) Run the mingw installer and mark mingw-tcl and mingw-tk for installation and apply changes.
Check they are there:
```
cd /mingw
find . -name "tk.h"
```
