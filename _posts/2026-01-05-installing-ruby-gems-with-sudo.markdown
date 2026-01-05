---
layout: post
title: "Installing Ruby Gems with sudo"
date: 2026-01-05
---
# Introduction

I'm using mise (https://mise.jdx.dev/) to manage ruby and rails.

```
 % ruby --version
ruby 3.4.2 (2025-02-15 revision d2930f8e7a) +PRISM [arm64-darwin24]
 % gem --version
3.6.2
```

Today I installed rails and it said:
```
 % rails --version
Rails is not currently installed on this system. To get the latest version, simply type:

    $ sudo gem install rails

You can then rerun your "rails" command.
```
This is a script, a stub, in /usr/bin/rails probably installed with the system version of ruby.  It checks if rails is installed and invokes it or prints the above if not.

But I thought "sudo", really?  Googling, it seems that using sudo with gem install is considered a "bad idea"™ because the installed files are owned by root and any hooks invoked on install run as root, as does the gem later when it's required.  Who knows what it's doing with those privileges?  There are also references to the gem being installed somewhere odd and not being available later to an non-root user but I could not confirm this behaviour.

I don't fully understand how mise does its magic but `gem environment` shows this when ruby is managed by mise:
```
  - GEM PATHS:
     - /Users/chris/.local/share/mise/installs/ruby/3.4.2/lib/ruby/gems/3.4.0
     - /Users/chris/.local/share/gem/ruby/3.4.0
```

The first is the global (--no-user-install) location (which is the default) and the second is where gem installs for the current user only (--user-install must be specified). Some people add this to their ~/.gemrc file but my system is just for me so I'm happy to install gems globaly. (Though, obviously, it's not realy global since mise has put it under my $HOME).

So I installed a gem with sudo to see what happens.
```
% sudo gem install  awesome_print
Fetching awesome_print-1.9.2.gem
Successfully installed awesome_print-1.9.2
Parsing documentation for awesome_print-1.9.2
Installing ri documentation for awesome_print-1.9.2
Done installing documentation for awesome_print after 0 seconds
1 gem installed
```
It installed ok in the global directory with root ownership as expected:
```
chris@MacBookPro ~/.local/share/mise/installs/ruby/3.4.2/lib/ruby/gems/3.4.0/gems
 % ll -d awesome_print-1.9.2
drwxr-xr-x  15 root  staff  480  5 Jan 14:47 awesome_print-1.9.2/
```
Using sudo gem uninstall removed it ok and I was able to install it again without sudo.

Note that using  `muse deactivate`  and going back to the system ruby environment shows this for `gem env`:
```
  - GEM PATHS:
     - /Library/Ruby/Gems/2.6.0
     - /Users/chris/.gem/ruby/2.6.0
     - /System/Library/Frameworks/Ruby.framework/Versions/2.6/usr/lib/ruby/gems/2.6.0
```
The first is the global location and has some gems already, the 3rd seems to be an Apple location that they want to always have some important gems in.  I assume some macos scripts depend on them being there.  My user local gem directory is empty.  Path 1 is not writable by the user so that's why you must use sudo to install gems globally.

So, if installing gems as sudo is a bad idea, there are a few options:
1. always use --user-install (probably put it in your ~/.gemrc)
2. define $GEM_HOME in your .profile (and add it to your $PATH)
3. install ruby using an "environment switcher" such as RVM, rbenv, chruby, asdf, or Mise

I like the last one because it makes sure your ruby environment is yours and one day I will want to switch between ruby environments anyway.
