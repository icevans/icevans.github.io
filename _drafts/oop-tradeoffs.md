---
title: Readable vs Useable
layout: post
---

I have only just begun the journey of learning object-oriented programming, but it is already very clear that OO design is all about tradeoffs. For example, trying to adhere to the "Tell, Don't Ask" paradigm requires pushing behavior into collaborator objects, but this can easily lead to classes that violate the "Single Responsibility Principle." Deciding which principle to weight more heavily in a particular case requires considering the context, and trusting your experience and intuition.

Of course, this can be tricky if you don't have much experience or intuition! In light of that, I've decided to not study classic OOP design patterns or principles at this stage in my learning. I want to first build lots of OOP programs, and make my own mistakes. I want to learn some lessons the hard way and develop my own intuitions about tradeoffs, anti-patterns, etc.

Today I want to write about one tradeoff, or _tension_, that I have already seen come up several times: between an interface with simple _readable_ code, and an interface that is _easy-to-use_. I don't mean to say that you can't interfaces that are both readable and easy to use, but simply that optimizing one of these tends to come at the expense of the other -- you need to find the right balance.

I ran into a perfect example of this while helping a fellow Launch School student refactor his Rock, Paper, Scissors game. Having expanded the game to "Rock, Paper, Scissors, Spock, Lizard", his logic for determing which moves beat which had become quite complicated. He had a `Move` class with a `>` method that looked something like this:

```ruby
def >(other_move)
  rock?     && other_move.value == 'scissors' ||
  rock?     && other_move.value == 'lizard'   ||
  paper?    && other_move.value == 'rock'     ||
  paper?    && other_move.value == 'spock'    ||
  scissors  && other_move.value == 'paper'    ||
  scissors? && other_move.value == 'lizard'   ||
  lizard?   && other_move.value == 'paper'    ||
  lizard?   && other_move.value == 'spock'    ||
  spock?    && other_move.value == 'scissors' ||
  spock?    && other_move.value == 'rock'
end
```

This isn't the worst thing in the world, but it's quite a bit of logic for a single method, and it will be annoying if we want to add another move or change the rules in the future. So my friend decided to try subclassing to simplify the situation:

```ruby
class Rock < Move
  # ... other code

  def >(other_move)
    other_move.value == 'scissors' ||
    other_move.value == 'lizard'
  end
end
```

That's a much nicer `#>` method! It's much easier to read, and would be much less intimidating to change down the line. We simply add subclasses with similar methods for the rest of the move types, and our program is better... right?

It's not so simple: unfortunately, simplifying the implementation of `>` in this way comes at a heavy cost. In the old implementation, we could instantiate a new `Move` object simply: `Move.new('rock')`, for example. This simplicity was useful because the type of `Move` that is generated is determined dynamically at runtime (when the human or computer players pick their move). Now, we need to determine somehow which subclass to instantiate each time we need a new move:

```ruby
choice == gets.chomp

case choice
when 'rock' then human.move = Rock.new
when 'paper' then human.move = Paper.new
when 'scissors' then human.move = Scissors.new
when 'Lizard' then human.move = Lizard.new
when 'Spock' then human.move = Spock.new
end
```

Ugh. That's a lot of work just to create a move object -- there's probably only two times we'd have to write that in our RPSLS game (once for the human's turn, once for the computer's turn), but still, the interface for our moves has become _much_ harder to work with. There's also a maintenance downside: if we add a new move type in the future, not only will we have to update the move classes' `>` methods, we'll also have to update the `case` statements that instantiate moves. This seems like a bug just waiting to happen.

We made it much easier to understand our moves' `>` methods, but at the cost of making the moves' interface much harder to use.

In this particular case, we can improve the situation a bit by making a new class that is responsible for instantiating moves:

```ruby
class MoveGenerator
  def generate(choice)
    case choice
    when 'rock' then = Rock.new
    when 'paper' then = Paper.new
    when 'scissors' then = Scissors.new
    when 'Lizard' then = Lizard.new
    when 'Spock' then = Spock.new
    end
  end
end
```

Now we it's simpler to create a new move: `human.move = MoveGenerator.new.generate('rock')`.

We've created a new interface to shield us from the awkward move interfaces. That may make the initial refactoring worth it in the end. Still, the general point holds: we simplified our moves at the expense of making their interface much harder to use.

