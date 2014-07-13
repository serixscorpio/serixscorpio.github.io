---
layout: post
title: Ruby Splat Operator
---
In Figaro's [env.rb](https://github.com/laserlemon/figaro/blob/master/lib/figaro/env.rb), the `method_missing` method has `*` as part of its arguments:

```ruby
def method_missing(method, *) 
  ...
end
```

The `*` is known as a splat operator, which allows a method to take a variable number of arguments.  Let's try it out in irb:

```ruby
> def foo(*args)
>   args
> end
> foo
=> []
> foo('hello')
=> ["hello"]
> foo('hello', :world)
=> ["hello", :world]
```

The above example uses the splat operator with a variable name `*args`.  We can also use `*` without a variable name:

```ruby
> def bar(*)
>   # do nothing
> end
> bar
=> nil
> bar('hello')
=> nil
> bar('hello', :world)
=> nil
```

Hmm... but that doesn't seem useful to have `*` without a variable name, since the `bar` method can no longer access the arguments.  How come Figaro uses the splat operator this way?  A [search](http://stackoverflow.com/questions/17984476/what-does-a-splat-operator-do-when-it-has-no-variable-name) shows that using `*` by itself prevents the current method from accessing the arguments, but allows the corresponding method in the superclass to access the arguments.  To illustrate:

```ruby
> class Parent
>   def baz(*args)
>     args
>   end
> end
>
> class Child < Parent
>   def baz(*)
>     super
>   end
> end
>
> child = Child.new
> child.baz
=> []
> child.baz('hello')
=> ["hello"]
> child.baz('hello', :world)
=> ["hello", :world]
```

In `Child`, `baz` can't access `'hello'`, but in `Parent`, `baz` can.  Using `*` without a variable name makes sure `Child` doesn't mess with the argument(s), but still allow `Parent` to process the argument(s).

Coming back to Figaro's [env.rb](https://github.com/laserlemon/figaro/blob/master/lib/figaro/env.rb), `method_missing(method, *)` is saying it doesn't mess with the argument(s), but makes them available to `method_missing` in  [BasicObject](http://www.ruby-doc.org/core-2.1.2/BasicObject.html#method-i-method_missing), which is the superclass.  Hopefully next time I see a splat operator, it won't be a stranger!
