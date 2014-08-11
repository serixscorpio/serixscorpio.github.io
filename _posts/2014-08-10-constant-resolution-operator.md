---
layout: post
title: Ruby Constant Resolution Operator (::)
---
I came across `::ENV` and `ENV` when I was reading the Figaro gem.  I was not able to articulate the difference between the two, so I set out to find more about the constant resolution operator (::).  Hopefully this will be useful for others as well.

Let's play around with it in irb (or alternatively, pry), to get a feel of what we are dealing with:
```
[1] pry(main)> ENV
=> {"CLUTTER_IM_MODULE"=>"xim",
 "COLUMNS"=>"119",
 "DBUS_SESSION_BUS_ADDRESS"=>"unix:abstract=/tmp/dbus-o7ga8nDMeL",
 "DEFAULTS_PATH"=>"/usr/share/gconf/Lubuntu.default.path",
 ...
 "XMODIFIERS"=>"@im=ibus",
 "ZSH"=>"/home/eric/.oh-my-zsh",
 "_LXSESSION_PID"=>"1689"}
[2] pry(main)> 
```
It looks like `ENV` has all of my environment variables already.  So I suspect this is built into Ruby already.  Indeed, this is the [ENV class documentation](http://www.ruby-doc.org/core-2.1.2/ENV.html).  Then I tried:
```
[2] pry(main)> ::ENV
=> {"CLUTTER_IM_MODULE"=>"xim",
 "COLUMNS"=>"119",
 "DBUS_SESSION_BUS_ADDRESS"=>"unix:abstract=/tmp/dbus-o7ga8nDMeL",
 "DEFAULTS_PATH"=>"/usr/share/gconf/Lubuntu.default.path",
...
 "XMODIFIERS"=>"@im=ibus",
 "ZSH"=>"/home/eric/.oh-my-zsh",
 "_LXSESSION_PID"=>"1689"}
[3] pry(main)>
```
It seems like we are getting the same output for both `ENV` and `::ENV`.  What does the `::` do?  The clearest explanation I can find is an excerpt from Dave Thomas and Andy Hunt's [Programming Ruby - The Pragmatic Programmer's Guide](http://ruby-doc.com/docs/ProgrammingRuby/) -> The Ruby Language (section) -> Scope of Constants and Variables (sub-section):

**"Constants defined outside any class or module may be accessed unadorned or by using the scope operator `::` with no prefix."**

But wait, we just confirmed that `ENV` is a Ruby core class.  Is it also a constant? Yes, in Ruby, [class names and module names are constants](http://rubylearning.com/satishtalim/ruby_names.html).   Let's confirm that:
```
[3] pry(main)> Object.constants
=> [:Object,
 :Module,
 :Class,
 ...
 :ENV,
 ...
 :RUBYGEMS_ACTIVATION_MONITOR]
[4] pry(main)> 
```
`ENV` is indeed one of `Object`'s constants.

So far, we still have not figured out why the Figaro gem uses [`::ENV`](https://github.com/laserlemon/figaro/blob/master/lib/figaro/env.rb#L33) instead of `ENV` to access the Ruby built-in `ENV` class:
```
module Figaro
  module ENV
    extend self
    # ...
    def has_key?(key)
      ::ENV.any? { |k, _| k.downcase == key }
    end
    # ...
  end
end
```
Could it be because Figaro also defines it's own module named `ENV`?  To test this out, let's use a custom made example:
```
[1] pry(main)> module MyModule
[1] pry(main)*   module ENV
[1] pry(main)*     extend self
[1] pry(main)*     def envvar(name)
[1] pry(main)*       ::ENV.fetch(name)
[1] pry(main)*     end  
[1] pry(main)*   end  
[1] pry(main)* end  
=> :envvar
[2] pry(main)> MyModule::ENV.envvar("HOME")
=> "/home/eric"
[3] pry(main)>
```
`::ENV` from the above example is using Ruby's built-in class, which knows about the [fetch](http://www.ruby-doc.org/core-2.1.2/ENV.html) method.  Now let's try:
```
[3] pry(main)> module MyModule
[3] pry(main)*   module ENV
[3] pry(main)*     extend self
[3] pry(main)*     def envvar(name)
[3] pry(main)*       ENV.fetch(name)
[3] pry(main)*     end  
[3] pry(main)*   end  
[3] pry(main)* end  
=> :envvar
[4] pry(main)> MyModule::ENV.envvar("HOME")
NoMethodError: undefined method `fetch' for MyModule::ENV:Module
from (pry):14:in `envvar'
[5] pry(main)> 
```
Here, `ENV` resolves to `MyModule::ENV`, which does not know about the `fetch` method.  How come `ENV` does not resolve to the built-in Ruby `ENV` class?  Based on [Stackoverflow Q&A](http://stackoverflow.com/questions/5032844/ruby-what-does-prefix-do), 
the prefix `::` refers to the global scope, so `::ENV` resolves to the build-in Ruby `ENV` class.  Without the prefix, `ENV` resolves to `MyModule::ENV` because when Ruby tries to resolve a constant, it begins searching within the current module or class, and continues searching the enclosing scope.  (For more details, check out [@valve's article](http://valve.github.io/blog/2013/10/26/constant-resolution-in-ruby/))
