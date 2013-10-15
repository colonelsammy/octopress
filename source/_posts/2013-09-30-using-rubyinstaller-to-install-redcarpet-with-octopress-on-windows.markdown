---
layout: post
title: "Using Ruby Installer to install Redcarpet and Github-linguist with Octopress on Windows"
date: 2013-09-30 17:32
comments: false
categories: [octopress, ruby, mingw, markdown, windows]
keywords: ruby, octopress, redcarpet, github, markdown, mingw, windows
description: Using Ruby Installer to install Redcarpet and Github-linguist syntax highlighting with Octopress on Windows
---

This article follows three posts describing how to install Ruby 1.9.3 to use Octopress with redcarpet and github-linguist in a native MingW environment; using what I learnt in that process I've figured out how to use the Ruby Installer for Windows to do the same thing - this post describes how to do that.  I plan to leave the original posts ([starting here]({% post_url 2013-09-24-setting-up-mingw %})) in case they may provide inspiration for someone else with similar ambitions.

I wanted to use redcarpet/github-linguist for markdown in Octopress, however there's a dependency on the charlock_holmes Gem that requires native code libraries (file, ICU), some of which it needs to build.  It turns out that we *can* do this with Ruby Installer

# First, install Ruby... #

Install and install ruby 1.9.3 ([from rubyinstaller.org](http://rubyinstaller.org/downloads/ "Ruby installer downloads"))
Download the devkit ([current version here](https://github.com/downloads/oneclick/rubyinstaller/DevKit-tdm-32-4.5.2-20111229-1559-sfx.exe ""))
Unpack dev kit, e.g. into C:\Ruby193\devkit
Setup the Ruby installation to work with the devkit:
```
ruby dk.rb init
ruby dk.rb install
```
All of this follows the regular instructions for installing Ruby and the devkit.
<!-- more -->
# Install Python 2.7 #
Next, install Python 2.7, if you havn't already - this is required for pygments.rb.  Make sure Python is in your path.  Check in a Mingw window:

```
python --version
```

# Additional tools for devkit #
For some reason, the devkit doesn't come with tar, zip & patch so install these from MingW downloads. I used 7-zip to unpack the wget and xz and then used the binaries of those tools to get the rest; if you want you can just download the missing tools and unpack them with 7-zip, then copy the binaries manually.

If you want to use MingW, first you need to download and unpack xz, wget and dlls for openssl and liblzma; [download wget from here](http://sourceforge.net/projects/mingw/files/MSYS/Extension/wget/wget-1.12-1/wget-1.12-1-msys-1.0.13-bin.tar.lzma/download) , [xz from here](http://sourceforge.net/projects/mingw/files/MSYS/Base/xz/xz-5.0.3-1/xz-5.0.3-1-msys-1.0.17-bin.tar.lzma/download), [openssl dlls from here](http://sourceforge.net/projects/mingw/files/MSYS/Extension/openssl/openssl-1.0.0-1/libopenssl-1.0.0-1-msys-1.0.13-dll-100.tar.lzma/download) and [liblzma dlls from here](http://sourceforge.net/projects/mingw/files/MSYS/Base/xz/xz-5.0.3-1/liblzma-5.0.3-1-msys-1.0.17-dll-5.tar.lzma/download). Unpack them all with 7-zip (unpack the tar files as well) and copy the binaries to the bin, etc and lib directories of your devkit (copying with Windows Explorer is fine).

Start the devkit using the msys.bat file located in the devkit root and check that they installed:
```
$ wget --version
GNU Wget 1.12 built on msys.
...
$ xz --version
xz (XZ Utils) 5.0.3
liblzma 5.0.3
``` 
Now you should be able to use MingW/MSYS to download and install the missing tools; create a temporary directory to unpack the files into and copy the files:
```
cd ~
mkdir tmp
cd tmp
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
cp -r -v bin /
cp -r -v share /
cd .. 
```
Now we're ready to install the dependencies for the charlock_holmes Gem...

# Build native libraries #

We need some additional libraries for the Ruby Gem charlock_holmes.  I've built them both with static linking here so that we don't need to worry about where the DLLs might have gone.
 
Download and build ICU ([this post on compiling for qt](http://qt-project.org/wiki/Compiling-ICU-with-MinGW "Compiling ICU with Mingw") and [this post for compiling for OpenTTD](http://wiki.openttd.org/Compiling_on_Windows_using_MinGW "Compiling for OpenTTD") both help):
```
cd ~
mkdir build32
cd ~/build32
wget http://download.icu-project.org/files/icu4c/4.6/icu4c-4_6-src.zip
unzip icu4c-4_6-src.zip
wget http://devs.openttd.org/~terkhen/libicu/libicu_4_6_mingw32.diff
wget http://devs.openttd.org/~terkhen/libicu/libicu_reduce_icudata_size.diff
cd icu
patch -p1 -i ../libicu_4_6_mingw32.diff
patch -p1 -i ../libicu_reduce_icudata_size.diff
cd source
./configure --prefix=/mingw --enable-static --disable-shared --disable-strict --disable-threads
#./configure --prefix=$PWD/../dist --enable-static --disable-shared --disable-strict --disable-threads
make && make install
cd ../..
```

Download and install libgnurx (required for file-5.08, also a dependency of charlock_holmes):
```
cd ~/build32
wget http://sourceforge.net/projects/mingw/files/Other/UserContributed/regex/mingw-regex-2.5.1/mingw-libgnurx-2.5.1-src.tar.gz
tar zxvf mingw-libgnurx-2.5.1-src.tar.gz
cd mingw-libgnurx-2.5.1
```
Apply the following patch for static build:
(diff -up Makefile.in Makefile.in.new > ../mingw-libgnurx-static.patch)
```
--- Makefile.in 2007-05-07 20:28:28 +0100
+++ Makefile.in.new     2013-09-23 12:44:15 +0100
@@ -56,7 +56,7 @@ SRCDIST_FILES = ${srcdir}/configure ${sr
 ZIPCMD = @ZIPCMD@
 ZIPEXT = @ZIPEXT@

-all: libgnurx-$(DLLVERSION).dll libgnurx.dll.a libregex.a @GNURX_LIB@
+all: libgnurx.a libgnurx-$(DLLVERSION).dll libgnurx.dll.a libregex.a @GNURX_LIB@

 Makefile: config.status Makefile.in
        ./config.status
@@ -69,6 +69,10 @@ $(OBJECTS): Makefile
 libgnurx-$(DLLVERSION).dll libgnurx.dll.a: $(OBJECTS)
        $(CC) $(CFLAGS) -shared -o libgnurx-$(DLLVERSION).dll $(LDFLAGS) $(OBJECTS)

+libgnurx.a: $(OBJECTS)
+       ar rcu $@ $(OBJECTS)
+       ranlib $@
+
 libregex.a: libgnurx.dll.a
        cp -p libgnurx.dll.a $@

@@ -81,6 +85,11 @@ install-dll:
        mkdir -p ${bindir}
        cp -p $(BINDIST_FILES) ${bindir}

+install-static: libgnurx.a
+       mkdir -p ${includedir} ${libdir}
+       cp -p ${srcdir}/regex.h ${includedir}
+       cp -p ${srcdir}/libgnurx.a ${libdir}
+
 install-dev:
        mkdir -p ${includedir} ${libdir}
        cp -p ${srcdir}/regex.h ${includedir}
```
Build and install:
```
patch -i ../mingw-libgnurx-static.patch
./configure -prefix=/mingw
make && make install-static
```
Optional: check that file-5.08 builds (note: **no make install**) without errors:
```
cd ~/build32
wget ftp://ftp.astron.com/pub/file/file-5.08.tar.gz
tar zxvf file-5.08.tar.gz
cd file-5.08
./configure --prefix=/mingw  --disable-shared --enable-static --with-pic
make
```

# Install Octopress #

Download Octopress ([see also Octopress setup instructions](http://octopress.org/docs/setup/ "Octopress setup")).  The easiest way to do this is to first install Git for Windows.  Start Git Bash, and then (note: this example uses ~/build32 but you can put octopress where you like...):
```
cd /c/Ruby193/devkit/home/[user]/build32/
git clone git://github.com/imathis/octopress.git octopress
```

At this stage (back in MingW build environment):
```
cd ~/build32/octopress
$ gem list

*** LOCAL GEMS ***

bigdecimal (1.1.0)
io-console (0.3)
json (1.5.5)
minitest (2.5.1)
rake (0.9.2.2)
rdoc (3.9.5)
```

Install bundler and the default bundle:
```
gem install bundler
```
```
$ gem list

*** LOCAL GEMS ***

bigdecimal (1.1.0)
bundler (1.3.5)
io-console (0.3)
json (1.5.5)
minitest (2.5.1)
rake (0.9.2.2)
rdoc (3.9.5)
```
Then install the default bundle:
```
bundle install
```
Install the default Octopress theme, create a new test page and generate the site:
```
rake install
rake new_post["first post"]
rake generate
```
You should get the site built in octopress/public.  Check that index.html is generated properly (size > 0).

At this point we have the default Octopress installed using rdiscount for markdown generation and we've setup the required libraries for redcarpet.  Now we need to install charlock_holmes manually:

## Install some updates required for github-linguist ##

Update the Gemfile to install pygments 0.3.7 - this is required so that we have the updated lexer Augeas (part contents of Gemfile):
```
gem 'pygments.rb', '~> 0.3.7'
```
and update the bundle:
```
bundle update pygments.rb
gem install rake-compiler
```
Now it should look something like this:
```
$ bundle list
Gems included by the bundle:
  * RedCloth (4.2.9)
  * bundler (1.3.5)
  * chunky_png (1.2.5)
  * classifier (1.3.3)
  * compass (0.12.2)
  * directory_watcher (1.4.1)
  * fast-stemmer (1.0.1)
  * fssm (0.2.9)
  * haml (3.1.7)
  * jekyll (0.12.0)
  * kramdown (0.13.8)
  * liquid (2.3.0)
  * maruku (0.6.1)
  * posix-spawn (0.3.6)
  * pygments.rb (0.3.7)
  * rack (1.5.2)
  * rack-protection (1.5.0)
  * rake (0.9.2.2)
  * rb-fsevent (0.9.1)
  * rdiscount (2.0.7.3)
  * rubypants (0.2.0)
  * sass (3.2.9)
  * sass-globbing (1.0.0)
  * sinatra (1.4.2)
  * stringex (1.4.0)
  * syntax (1.0.0)
  * tilt (1.3.7)
  * yajl-ruby (1.1.0)
```

Create a patch file for pygments.rb:
```
--- popen.rb	2013-09-24 14:41:22 +0100
+++ popen.rb.new	2013-09-24 15:12:42 +0100
@@ -110,7 +110,7 @@
     def lexers
       begin
         lexer_file = File.expand_path('../../../lexers', __FILE__)
-        raw = File.open(lexer_file, "r").read
+        raw = File.open(lexer_file, "rb").read
         Marshal.load(raw)
       rescue Errno::ENOENT
         raise MentosError, "Error loading lexer file. Was it created and vendored?"
```

Now apply the patch to popen.rb in pygments.rb:
```
cd ~/local32/lib/ruby/gems/1.9.1/gems/pygments.rb-0.3.7/lib/pygments
patch -u -i ~/build32/pygments.rb.popen.rb.patch
cd ~/build32/octopress
```

## Get and patch charlock_holmes ##
```
mkdir ~/build32/tmpgems
cd ~/build32/tmpgems
gem fetch charlock_holmes
gem spec charlock_holmes-0.6.9.4.gem --ruby > charlock_holmes.gemspec
gem unpack charlock_holmes-0.6.9.4.gem
mv -v charlock_holmes.gemspec charlock_holmes-0.6.9.4
cd charlock_holmes-0.6.9.4
```
Now modify ext/charlock_holmes/extconf.rb:
```
--- extconf.rb	2013-09-24 14:49:14 +0100
+++ extconf.rb.new	2013-09-24 14:51:54 +0100
@@ -9,7 +9,7 @@
   ret
 end
 
-if `which make`.strip.empty?
+if `sh which make`.strip.empty?
   STDERR.puts "\n\n"
   STDERR.puts "***************************************************************************************"
   STDERR.puts "*************** make required (apt-get install make build-essential) =( ***************"
@@ -37,7 +37,8 @@
   end
 end
 
-unless have_library 'icui18n' and have_header 'unicode/ucnv.h'
+#unless have_library 'icui18n' and have_library 'icuuc' and have_library 'regex' and have_header 'unicode/ucnv.h'
+unless have_library 'icudata' and have_library 'icuuc' and have_library 'icui18n' and have_library 'gnurx' and have_header 'unicode/ucnv.h'
   STDERR.puts "\n\n"
   STDERR.puts "***************************************************************************************"
   STDERR.puts "*********** icu required (brew install icu4c or apt-get install libicu-dev) ***********"
@@ -57,7 +58,7 @@
 
   sys("tar zxvf #{src}")
   Dir.chdir(dir) do
-    sys("./configure --prefix=#{CWD}/dst/ --disable-shared --enable-static --with-pic")
+    sys("sh ./configure --prefix=#{CWD}/dst/ --disable-shared --enable-static --with-pic")
     sys("patch -p0 < ../file-soft-check.patch")
     sys("make -C src install")
     sys("make -C magic install")
```

```
cd ext/charlock_holmes
patch -u -i /build32/charlock_holmes_extconf.rb.patch
cd ../..
gem build charlock_holmes.gemspec
gem install charlock_holmes-0.6.9.4.gem --platform=ruby
```

We should now have charlock_holmes installed:
```
$ gem list

*** LOCAL GEMS ***

bigdecimal (1.1.0)
bundler (1.3.5)
charlock_holmes (0.6.9.4)
chunky_png (1.2.5)
classifier (1.3.3)
compass (0.12.2)
directory_watcher (1.4.1)
fast-stemmer (1.0.1)
fssm (0.2.9)
haml (3.1.7)
io-console (0.3)
jekyll (0.12.0)
json (1.5.5)
kramdown (0.13.8)
liquid (2.3.0)
maruku (0.6.1)
minitest (2.5.1)
posix-spawn (0.3.6)
pygments.rb (0.3.7, 0.3.4)
rack (1.5.2)
rack-protection (1.5.0)
rake (0.9.2.2)
rake-compiler (0.9.1)
rb-fsevent (0.9.1)
rdiscount (2.0.7.3)
rdoc (3.9.5)
RedCloth (4.2.9 x86-mingw32)
rubypants (0.2.0)
sass (3.2.9)
sass-globbing (1.0.0)
sinatra (1.4.2)
stringex (1.4.0)
syntax (1.0.0)
tilt (1.3.7)
yajl-ruby (1.1.0 x86-mingw32)
```

## Add redcarpet to Octopress ##
```
cd ~/build32/octopress
```

Now we can modify the Gemfile to include redcarpet (part contents of Gemfile):
```
  gem 'rdiscount', '~> 2.0.7'
  gem "charlock_holmes", "~> 0.6.9.4"
  gem "github-linguist", "~> 2.4"
  gem 'redcarpet', '~> 2.1.1'
  gem 'pygments.rb', '~> 0.3.7'
```
And update the Gems:
```
cd ~/build32/octopress
bundle install
```

Test. Save this code into a file in (for example) /tmp/test.rb:
```
require 'rdiscount'
require 'charlock_holmes'
require 'escape_utils'
require 'pygments'
require 'yaml'
require 'linguist/classifier'
require 'linguist/samples'
require 'linguist/file_blob'
require 'redcarpet'

puts "Ruby test"

#puts Pygments.highlight(File.read(__FILE__), :lexer => 'ruby')

@rdiscount_extensions = "smart"

content = "Some *text*"
rd = RDiscount.new(content, *@rdiscount_extensions)
html = rd.to_html

puts html

detection = CharlockHolmes::EncodingDetector.detect(html)

puts detection

fb = Linguist::FileBlob.new("c:\\Ruby193\\devkit\\dk.rb")
puts fb.language.name

markdown = Redcarpet::Markdown.new(Redcarpet::Render::HTML, :autolink => true, :space_after_headers => true)
puts markdown.render("This is *bongos* ```c++ int main() { return 0 } ```, indeed.")

```

And run the test:
```
$ ruby test.rb
...
Ruby test
<p>Some <em>text</em></p>
{:type=>:text, :encoding=>"ISO-8859-2", :confidence=>40, :language=>"cs"}
Ruby
<p>This is <em>bongos</em> <code>c++ int main() { return 0 }</code>, indeed.</p>
``` 

Now modify _config.yml for octopress to generate redcarpet markdown:
```
markdown: redcarpet
redcarpet:
  extensions: ["no_intra_emphasis", "fenced_code_blocks", "autolink", "tables", "with_toc_data"]
```

Modify the 'frist post' created above to include some github markdown, e.g. (note: replace last [backtick]...):
```
Some text
```c++
int main()
{
  return 0;
}
``[backtick]

Some more text.
``` 

And regenerate:
```
rake generate
```
You should now have syntax highlighted code blocks, like this:

```c++
int main()
{
  return 0;
}
```
