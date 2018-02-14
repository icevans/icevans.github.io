---
title: With Index
layout: post
---

I discovered the `Enumerable#with_index` method yesterday and keep finding places to use it. I've found it especially handy when combined with `#map`, where your mapping function needs to take into account the indices of the values you're mapping. Combining these methods goes like this:

```ruby
array.map.with_index { |item, index| ... }
```

Most recently, I was writing a method to convert an integer in string form into a proper string (as an exercise; I am aware of `String#to_i`). My logic was straightforward:

1. Split the string into an array of numerals (i.e. digits in string form)
1. Map the array of numerals to their integer counterparts, by doing a lookup in a string to digits hash
1. Loop through the resulting array of digits backwards, multiplying each digit by `10**i`, where `i` is that digits index in the array
1. Sum the resulting numbers

To complete steps 3 and 4, `Array#reduce` struck me as a natural candidate. One form of `reduce` accepts both a starting value and a block; I figured I'd pass in a block that multiplies the current digit by `10**i` and then adds it to the running sum.

To do this, I'd need to keep track of the index at each iteration. A perfect opportunity to call `#with_index`... or so I thought:

```ruby
digits.reduce(0).with_index { |sum, num, i| sum += num * 10**i }
```

Unfortunately, the above throws an error: `TypeError: 0 is not a symbol nor a string`. Ruby thinks the above code is using the shorthand version of `#reduce` where you pass a symbol as an argument instead of passing an explicit block. Of course, the _intention_ was to pass an initial value for `sum` and then a block.

The mistake is that this line of code passes the block to `with_index`, so `reduce` never sees it, and so thinks I'm simply attempting to pass it a symbol. At first, I thought this was just a limitation of using `reduce` with `with_index`, and that my algorithm might be able to work without setting the initial value to 0.

But then it dawned on me that there was a bigger problem. `#with_index` is an `Enumerable` method. So you can't just call it on an `Array` (unless `Array` had an implentation, which it doesn't). Why then does

```ruby
array.map.with_index { |item, index| ... }
```

work? As usual, the ruby documentation tells the story: calling `#map` with no arguments and no block returns an `Enumerator` instance. Of course, there is no corresponding form for `#reduce` because the whole point of that method is to reduce an array to a single value, for which there would be no sensible conversion to an `Enumerator`. 

So there is, unfortunately, no way to use `with_index` with `reduce`, and so no obvious way to keep track of the index as you reduce an array.

On the upside, the way around this problem was straightforward: map the array of digits to the result of multiplying them by `10**i`, and then just call `sum` on the result:

```ruby
digits.map.with_index { |num, i| num * 10**i }.sum
```

This is nothing revelatory and is all straightforwardly in the documentation. But it does serve as useful reminder of the sort of trouble you can get into when doing something like `array.map.with_index { ... }` without really understanding what's going on.
