---
title: Accessing Substrings by Range — an Exploration
layout: post
---

In ruby, you can access a substring of a string by passing the `#[]` method a range. I was recently writing a simple 'swap first and last character' method for a Launch School exercise. I wanted my method to be non-mutating, so my first though was: (1) set a variable to the first character (`word[0]`), set another variable to the last character (`word[-1]`), and then get the word minus first and last with `word[1..-2]`. My next thought was, "Oh, that won't work with 1 and 2 character words." I expected:

```ruby
'a'[1..-2]  # => nil
'ab'[1..-2] # => 'ab' or 'ba' or 'a'   ??
```

I expected `nil` in the first case because, it would seem, we are attempting to access the value at an index that doesn't exist. After all, `'a'[1]` returns `nil`. The second case I wasn't entirely sure what to expect. In this case, `[1]` is the last character and `[-2]` is the first character. If we preserve the order of the original string, all the characters between `[1]` and `[-2]` are `ab`; if we go with the order implied by the range then it's `ba`; if we start at the beginning of the range and say, "oh, we're already at the end of the string, nothing more to get," then it's `a`.

Unsurprisingly, my intuitions here were completely wrong. Here is what actually happens:

```ruby
'a'[1..-2]  # => ''
'ab'[1..-2] # => ''
```

To understand the result, we'll have to consult the documentation for `String#[]`:

> ...If passed a range, its beginning and end are interpreted as offsets delimiting the substring to be returned.
> In these three cases, if an index is negative, it is counted from the end of the string. For the `start` and `range` cases the starting index is just before a character and an index matching the string's size. Additionally, an empty string is returned when the starting index for a character range is at the end of the string.
> Returns `nil` if the initial index falls outside the string or the length is negative.`

This is actually fairly unclear. Let's start with the second-to-last sentence. At first glance, it would seem to imply that `'ab'[1..n]` return `''`. But no! `'ab'[1..n]` returns `'b'`! Unless `n` is negative, in which case it returns `''`! But doesn't the second to last sentence say that an empty string is returned "when the starting index for a character range is at the end of string"??

Assuming the documentation is consistent, we can infer that in `'ab'[1..n]`, `1` is not the starting index. The preceding sentence gives us a clue about how to determine the staring index:

> ...the starting index is just before a character and an index matching the string's size. 

I'm not sure what it would mean for "a character and an index" to match a string's size, so I don't find this helpful. However, a little irb sleuthing reveals:

```ruby
'ab'[2..n] # => '' (for now, ignoring the case where n is negative)
'ab'[3..n] # => nil
```

So by "at the end of the string," I suppose the documentation means "the first index past the last index containing a value," or, what comes to the same thing since strings are 0-indexed, "string[n], where n is the length of the string. This leads me to postulate **Rule 1**:

> For strings of length n, `string[n..x]` returns `''`

And, from the last sentence of the documentation quoted above, **Rule 2**:

> For strings of length n, `string[n+1..x]` returns `nil`

These rules explain why `'a'[1..-2]` returns `''`. 

Unfortunately, they are insufficient to explain all of our initial observations. Rule 1 doesn't explain why `'ab'[1..-2]` returns `''`, since `1` is less than `'ab'.length`. To explain this, we need to think more about ranges. `'ab'[-2]` is the same as `'ab'[0]`, so `'ab'[1..-2]` is essentially `'ab'[1..0]`. In ruby, ranges where the end is less than the start are allowed, but they're degenerate. If you try to iterate through such a range, iteration stops before it even gets started, since `Range#each` uses `Integer#succ` to get the next iteration, and compares each iteration to the end of the range using `<=>` to determine when to stop: `1.succ <=> 0 == 1`, so we're done before we do anything. And you get weird answers like `(1..0).cover?(1) # => false`. 

Now let's suppose the implementation of `#[]` is something like this (to simplify, covering only the case where the input is a range):

```ruby
def [](string, range)
  sub_string = ''
  range.each { |i| sub_string << string[i] }
  sub_string
end
```

It's tempting to postulate **Rule 3**

> When x > y, `string[x..y]` returns `''`

However, `'ab'[3..1]` returns `nil` (I guess, honoring Rule 2?). But we can explain this if we slightly complicate our definition of `[]` to accomodate Rule 2:

```ruby
def [](string, range)
  return nil if range.first > string.length

  sub_string = ''
  range.each { |i| sub_string << string[i] }
  sub_string
end
```

Obviously, the actual implementation is different since it's a class method and it's written in C. But it's got to be doing something like that. At the very least, supposing it works like that is a mental model that explains all of the data, so we can go with it.

Let's sum up by re-numbering, and re-stating, our rules:

**Rule 1**: `string[x..y]` returns nil if `x > string.length`
**Rule 2**: If x == `string.length`, `string[x..y]` returns `''`
**Rule 3**: If x < `string.length` and x > y, `string[x..y]` returns `''`

I think the final take-home is that that's a fairly complicated set of rules, and relying on them is likely to generate bugs. When accessing substrings by range, it's probaby best to avoid ending the range with negative indices so you don't end up with a degenerate range. 
