---
layout: post
title: "Installing Ruby in Windows"
date: 2013-09-27 17:32
comments: false
unpublished: true
categories: [octopress, ruby, mingw, windows]
keywords: ruby, octopress, redcarpet, github, mingw, windows
description: How to install Octopress with Redcarpet and Gitgub syntax highlighting with Mingw on Windows
---

Install ruby 1.9.3 (from ...)
Download devkit from ....
Unpack dev kit, e.g. into C:\Ruby193\devkit
Install devkit:
ruby dk.rb init
ruby dk.rb install

install tar, zip, unzip & patch:
```
wget http://sourceforge.net/projects/mingw/files/MSYS/Base/tar/tar-1.22-1/tar-1.22-1-msys-1.0.11-bin.tar.lzma
xz -d tar-1.22-1-msys-1.0.11-bin.tar.lzma
tar xvf tar-1.22-1-msys-1.0.11-bin.tar
wget http://sourceforge.net/projects/mingw/files/MSYS/Extension/zip/zip-3.0-1/zip-3.0-1-msys-1.0.14-bin.tar.lzma
xz -d zip-3.0-1-msys-1.0.14-bin.tar.lzma
tar xvf zip-3.0-1-msys-1.0.14-bin.tar
wget http://sourceforge.net/projects/mingw/files/MSYS/Extension/unzip/unzip-6.0-1/unzip-6.0-1-msys-1.0.13-bin.tar.lzma
xz -d unzip-6.0-1-msys-1.0.13-bin.tar.lzma
tar xvf unzip-6.0-1-msys-1.0.13-bin.tar
wget http://sourceforge.net/projects/mingw/files/MSYS/Extension/patch/patch-2.5.9-1/patch-2.5.9-1-msys-1.0.11-bin.tar.lzma
xz -d patch-2.5.9-1-msys-1.0.11-bin.tar.lzma
tar xvf patch-2.5.9-1-msys-1.0.11-bin.tar
wget http://sourceforge.net/projects/mingw/files/MSYS/Base/gzip/gzip-1.3.12-1/gzip-1.3.12-1-msys-1.0.11-bin.tar.lzma
xz -d gzip-1.3.12-1-msys-1.0.11-bin.tar.lzma
tar xvf gzip-1.3.12-1-msys-1.0.11-bin.tar
```
On the devkit msys:
```
cp -r -v bin /
cp -r -v share / 
```
Using another msys/MingW, download ICU ([this post helps a lot](http://qt-project.org/wiki/Compiling-ICU-with-MinGW "Compiling ICU with Mingw")):
```
wget http://download.icu-project.org/files/icu4c/51.2/icu4c-51_2-src.tgz
tar zxvf icu4c-51_2-src.tgz
```

In the devkit:
```
cd icu/source
./runConfigureICU MinGW --prefix=/mingw
make && make install
```

ICU DLLs have been renamed, but our ruby installation needs the old names, so rename them (see [this Boost post anout ICU renaming](http://boost.2283326.n4.nabble.com/icu-libraries-renamed-in-icu-50-td4643722.html "Boost post anout ICU renaming") and [this LibreOffice post on ICU renaming](http://nabble.documentfoundation.org/Naming-clash-on-icuin-lib-on-Windows-build-td3270700.html "LibreOffice post on ICU renaming")):

* icudata library is now named icudt 
* icui18n library is now named icuin 

```
mv -v /mingw/lib/icuin.dll /mingw/lib/icui18n.dll
mv -v /mingw/lib/icuin.lib /mingw/lib/icui18n.lib
mv -v /mingw/lib/icudt.dll /mingw/lib/icudata.dll
mv -v /mingw/lib/icudt.lib /mingw/lib/icudata.lib
```
Using another msys/MingW, download libgnurx (required for file-5.08, a dependency of charlock_holmes):
```
wget http://sourceforge.net/projects/mingw/files/Other/UserContributed/regex/mingw-regex-2.5.1/mingw-libgnurx-2.5.1-src.tar.gz
tar zxvf mingw-libgnurx-2.5.1-src.tar.gz
```
In the devkit:
```
cd mingw-libgnurx-2.5.1
./configure -prefix=/mingw
make && make install
```
Optional: check that file-5.08 builds (note: **no make install**):
```
wget ftp://ftp.astron.com/pub/file/file-5.08.tar.gz
tar zxvf file-5.08.tar.gz
cd file-5.08
./configure --prefix=/mingw  --disable-shared --enable-static --with-pic
make
```
Download Octopress ([see also Octopress setup instructions](http://octopress.org/docs/setup/ "Octopress setup")).  The easiest way to do this is to first install Git for Windows.  Start Git Bash, and then (note: this example uses /build32 but you can put octopress where you like...):
```
cd /c/mingw/build32
git clone git://github.com/imathis/octopress.git octopress
```

Install bundler and the default bundle:
```
gem install bundler
bundle install
```
Install the default Octopress theme:
```
rake install
```
Now install charlock_holmes manually:
```
gem install charlock_holmes -- --with-icu-lib=/mingw/lib --with-icu-include=/mingw/include
```
gem fetch charlock_holmes
gem spec charlock_holmes-0.6.9.4.gem --ruby > charlock_holmes.gemspec
gem unpack charlock_holmes-0.6.9.4.gem
move charlock_holmes.gemspec charlock_holmes-0.6.9.4
cd charlock_holmes-0.6.9.4
gem build charlock_holmes.gemspec
gem install charlock_holmes-0.6.9.4.gem -- --with-icu-lib=c:/Ruby193/devkit/mingw/lib --with-icu-include=c:/Ruby193/devkit/mingw/include

start msys.bat to start an msys command prompt
download icu (from http://site.icu-project.org/download need the 'C' package), e.g.icu4c-51_2-src.zip
unzip and copy to the devkit tree, e.g to:C:\Ruby193\devkit\home\NoyesMa
cd to icu/source
./configure
make
make install

gem install charlock_holmes -- --with-icu-lib=C:/Ruby193/devkit/local/lib --with-icu-include=C:/Ruby193/devkit/local/include

Download octopress
git clone .../octopress.git
cd octopress
gem install bundler

modify Gemfile:
add ??? (do we need all these??):
  gem 'redcarpet', '~> 2.1.1'
  gem "charlock_holmes", "~> 0.6.9.4"
  gem "github-linguist", "~> 2.4"

Install Python 2.7!!

This:
  gem 'rdiscount', '~> 2.0.7'
  gem "charlock_holmes", "~> 0.6.9.4"
  gem "github-linguist", "~> 2.4"
  gem 'pygments.rb', '~> 0.3.7'
  gem 'redcarpet', '~> 2.1.1'
  gem 'RedCloth', '~> 4.2.9'

Update pygments poen.rb line 113 to:
raw = File.open(lexer_file, "rb").read
