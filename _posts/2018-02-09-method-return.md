---
title: Method Return Values in Ruby
layout: post
---

When writing a method, you often want to return different values depending on whether some condition holds. In ruby, the standard is generally not to explicitly use the `return` keyword, but to simply rely on the fact that a method in ruby will return the value of the last statement in the method that gets evaluated.

This generally produces predictable results, but there are some gotchas if you're new to ruby. Consider the following method:

```ruby
def sign(number)
  if number < 0
    '-'
  elsif number > 0
    '+'
  else
    ''
  end
end
```

This works as expected:

```ruby
sign(1)  #=> '+'
sign(-5) #=> '-'
sign(0)  #=> ''
```

You might think to tighten this up by using ruby's nice one-line `if` statements

```ruby
def sign(number)
  '-' if number < 0
  '+' if number > 0
  ''  if number == 0
end
```

Unfortunately, this won't work for you:

```ruby
sign(1)  #=> nil
sign(-5) #=> nil
sign(0)  #=> ''
```

Instead, if you use the one-line `if`s, you need to use `return` keywords:

```ruby
def sign(number)
  return '-' if number < 0
  return '+' if number > 0
  return ''  if number == 0
end
```

What gives? And what's up with the `nil` return values when we didn't use `return`? To understand this, we need to remember that ruby uses _lazy evaluation_ for `if` statements, and that the value of an `if` statement that evaluates to `false` is `nil`.

## Lazy Evaluation

Consider the following:

```ruby
if 2 > 1
  puts 'foo'
elsif if 2 > 0
  puts 'bar'
else
  puts 'baz'
end
```

According to the ruby documentation, "Once a condition matches, either the if condition or any elsif condition, the if expression is complete and no further tests will be performed." That means in this case that only the first conditional
branch will be taken, and the others _will not even be evaluated_. It follows directly that if that code were at the end of a method, `puts 'foo'` would be the last statement evaluated in the method, and so its value (`nil` as it happens) would be the return value of the method.

If, however, you were to put additional statements below this `if...end` statement, they would of course be evaluated as usual. And this is precisely what happens when you use a series of one-line `if` statements, which is equivalent to

```ruby
if ...
  ...
end
if ...
  ...
end
if ...
  ...
end
```

Even if the first of those `if` statements evaluates to true, the next two are evaluated as well. If that construction were the body of your method, your return value would _not_ be the value of the statement inside the first true conditional, but instead the value of the last conditional.

## A False Conditional Returns nil

Let's return to our original ill-formed method:

```ruby
def sign(number)
  '-' if number < 0
  '+' if number > 0
  ''  if number == 0
end
```

We know from the above that the return value of `sign(1)` will be the value of the statement `'' if number == 0`. In ruby, a conditional whose test isn't true returns `nil`

```ruby
puts 'hi' if 0 > 1 #=> nil
```

So this method will always return `nil` except when we pass `0` as an argument, in which case it will return `''`. Adding `return` statements to the beginning of these lines fixes the issue simply because the `return` method tells ruby to immediately stop exit from the method, with the return value of whatever argument is passed to `return`.

---

Like my earlier post today, nothing revelatory here, just another example of how understanding the details of a language can explain why we use common patterns.
