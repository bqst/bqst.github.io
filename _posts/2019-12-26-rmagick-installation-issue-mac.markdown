---
layout: post
title:  "RMagick installation issue on macOS: Can't find MagickWand.h"
date:   2019-12-26 11:44:13 +0100
categories: [macos, bash]
tags: [rmagick, imagemagick]
---

I use a Mac, and homebrew for many packages including ImageMagick. I noticed a very annoying issue that I had to work around.

When I tried to install rmagick gem with :
`gem install rmagick`

```bash
Building native extensions. This could take a while...
ERROR:  Error installing rmagick:
        ERROR: Failed to build gem native extension.

    current directory: /Users/bqst/.rbenv/versions/2.5.3/lib/ruby/gems/2.5.0/gems/rmagick-2.16.0/ext/RMagick
/Users/bqst/.rbenv/versions/2.5.3/bin/ruby -I /Users/bqst/.rbenv/versions/2.5.3/lib/ruby/site_ruby/2.5.0 -r ./siteconf20191226-75151-8hx0yf.rb extconf.rb
checking for clang... yes
checking for Magick-config... no
checking for pkg-config... yes
checking for outdated ImageMagick version (<= 6.4.9)... no
checking for presence of MagickWand API (ImageMagick version >= 6.9.0)... no
checking for Ruby version >= 1.8.5... yes
checking for stdint.h... yes
checking for sys/types.h... yes
checking for wand/MagickWand.h... no

Can't install RMagick 2.16.0. Can't find MagickWand.h.
*** extconf.rb failed ***
Could not create Makefile due to some reason, probably lack of necessary
libraries and/or headers.  Check the mkmf.log file for more details.  You may
need configuration options.

Provided configuration options:
        --with-opt-dir
        --without-opt-dir
        --with-opt-include
        --without-opt-include=${opt-dir}/include
        --with-opt-lib
        --without-opt-lib=${opt-dir}/lib
        --with-make-prog
        --without-make-prog
        --srcdir=.
        --curdir
        --ruby=/Users/bqst/.rbenv/versions/2.5.3/bin/$(RUBY_BASE_NAME)

To see why this extension failed to compile, please check the mkmf.log which can be found here:

  /Users/bqst/.rbenv/versions/2.5.3/lib/ruby/gems/2.5.0/extensions/x86_64-darwin-18/2.5.0-static/rmagick-2.16.0/mkmf.log

extconf failed, exit code 1
```

The rmagic gem compile does not seem capible of following symlinks and therefore cannot find /usr/local/include/wand/MagickWand.h which is there.

rmagick has a problem working with imagemagick (>= 6.8.0-10) from Homebrew.
To solve the issue in Mac El Capitan, OSX Sierra, High Sierra, Mojave and Catalina,
you can either do the following:

update `rmagick` gem by

```bash
bundle update rmagick
```

or

```bash
brew unlink imagemagick
brew install imagemagick@6 && brew link imagemagick@6 --force
```

`imagemagick@6` is `keg-only`, so you'll need to force linking.
