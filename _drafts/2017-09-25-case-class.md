---
title: Case and Class
layout: post
---

Recently, [Mike Fatula](https://launchschool.com/users/0a7bff73) raised an interesting question after posting his solution to the Array Average "Small Problems" exercise. Mike was attempting to evaluate the class of some user input (checking whether it was an `Array` or a `String`) in a case statement. The puzzle was that the natural way of attempting this wasn't working. To simplify the example a bit, consider the following:

```ruby
the_class = [].class

case the_class
when Array
  # do something
else
  # do something else
end
```

In this case, you'd expect the `when Array` condition to be triggered. In fact, however, the `when` condition evaluates to false and the `else` code gets triggered. To add to the puzzle, this seemingly equivalent construction _does_ work:

```ruby
if the_class == Array
  # do something
else
  # do something else
end
```

The first condition evaluates to true and the intended code gets executed. What's going on? After poring over the ruby docs and doing a bunch of tests in irb, I figured out the answer, and I thought it was interesting enough to write up and share (this is probably old hat for long time rubyists, but I think it's a bit inobvious for a novice like me).

## Introducing .=== Operator

It will be helpful later to recall that `==` is really just a method, so we can rewrite the above `if` statement as:

```ruby
if the_class.==(Array)
  #do something
else
  #do something else
end
```

One might have expected that, behind the scenes, this is how our case statement works: call the `#==` method on `the_class` with `Array` as the parameter. Clearly, though, that's not what happens. Unfortunately, the documentation for `case` is silent on implementation details. 

However, a bit of googling reveals [a helpful Stack Overflow thread](https://stackoverflow.com/questions/3801469/how-to-catch-errnoeconnreset-class-in-case-when), which points out that `case` statements actually call the `===` operator. Still, I don't think this thread really gets at what's happening under the hood. For a properly [clear and distinct understanding](https://plato.stanford.edu/entries/descartes-epistemology/#5.1), we'll have to dive deep into the ruby docs.

The first thing that we find is that the `===` "operator" is, like `==`, actually a method. At the highest level, `#===` is an instance method of the `Object` class. The [documentation](http://ruby-doc.org/core-2.4.1/Object.html#method-i-3D-3D-3D) states that:

> Case Equality -- For class Object, effectively the same as calling #==, but typically overridden by descendants to provide meaningful semantics in case statements.

Since we've already established that our results are different from `#==`, some relevant descendent  of `Object` must be overriding this. What is the descendent? Since our `case` statement is evaluating classes, it is presumably a `Class`. Although the `Class` documentation doesn't list `#===` as a method, `Class`'s parent, `Module`, _does_. The [documentation](http://ruby-doc.org/core-2.4.1/Module.html#method-i-3D-3D-3D) states that:

> mod === obj → true or false
> Case Equality -- Returns true if obj is an instance of mod or an instance of one of mod’s descendants. Of limited use for modules, but can be used in case statements to classify objects by class."

Now we're getting somewhere: `===`, when called on a `Module` (or an instance of any of its descendants, like, say a `Class`), checks whether the object on the right side is _an instance of_ the object on the left. So:

```ruby
Array.===([])    # => true
Array.===(Array) # => false
```

The first is true because `[]` is an instance of `Array`. The second is false because `Array` is not an instance of itself (not that this is in general disallowed: `Class` is a member of `Class`, raising the spectre of [Russellian paradox](https://en.wikipedia.org/wiki/Russell%27s_paradox)) -- it is an instance of `Class`. So:

```ruby
Class.===(Array) # => true
```

## Anti-Symmetry

This last example also reveals an additional interesting twist: `#===` is not [symmetric](https://en.wikipedia.org/wiki/Symmetric_relation)! After all, `Class` is not an instance of `Array`, so `Array.===(Class)` returns `false`. This means that when we have:

```ruby
case a
when b
```

we need to ask whether `.===` is called on `a` with `b` as a parameter, or on `b` with `a` as a parameter (i.e. whether it's equivalent to `if a.===(b)` or `if b.===(a)`). I couldn't find any documentation on this, but a few tests in irb reveal it's the latter.

## The Payoff

We're finally ready for the payoff. Our original code:

```ruby
the_class = [].class

case the_class
when Array
  # do something
else
  # do something else
end
```
is actually equivalent to:
```ruby
the_class = [].class

if Array.===(the_class)
  # do something
else
  # do something else
end
```
Since we're calling `#===` on `Array`, which is an instance of `Class`, which is a descendent of `Module`, we're playing by `Module#===` rules. That means the `if` condition is checking whether `the_class` is an instance of `Array`. But, of course it isn't (remember: `Array` is not an instance of `Array`, it is an instance of `Class`). Hence the first condition does not evaluate to true and we fall into the `else` clause. Mystery solved! 

Err... sort of. Mike probably still wants ask: how _should_ we use a case statement to check the class of an object? Well, `Array.===(the_class)` does not return the result we're interested in, but `Array.===([])` does. So there's no need to explicitly get the class of `[]` and then pass that to the case statement. Instead, we can simply do:

```ruby
my_array = []

case my_array
when Array
  # do something
else
  #do something else
end
```
and we get the desired results. This is because behind the scenes, that first `when` is checking whether `Array.===(my_array)` -- that is, whether `my_array` is an instance of `Array`, which, of course it is.

Although it at first seems counterintuitive, this way of writing the case statement really is much more readable and direct. And it's little bits of language design like this that make ruby so pleasant to use.