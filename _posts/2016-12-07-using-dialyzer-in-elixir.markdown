---
title: "Using Dialyzer in Elixir"
layout: post
date: 2016-12-07 13:26
image: '/assets/images/'
description:  "Type checking in Elixir with Dialyzer"
tag: elixir, types, dialyzer
blog: true
jemoji:
author: geoff
---
Below is the general presentation from the December 2016 [Houston Elixir Meetup](https://www.meetup.com/Houston-Elixir-Meetup/events/235921105/) on strings and binaries.

# DialyzerDemo

## Typing in Erlang
Erlang (and by Elixir) is a dynamically typed language. As a dynamic language, some type errors cannot by cought until runtime. Some parts of the language inherently make type errors harder to get than other dynamic languages: compilation, pattern matching, guard clauses, and the approach to errors go a long way. However, it does not grant the absolute type safety that you might get from ML-family languages or even languages that are just statically typed.  

## What is dialyzer?
`The Dialyzer, a DIscrepancy AnalYZer for ERlang programs`.
Dialyzer is a static analysis tool used for checking types and other discrepancies such as dead or unreachable code. 

Dialyzer examines .erl or .beam files (not .ex or .exs) -- more on this in a minute.


## Success typing
Success typing takea a very optomistic treatment of your code. Dialyzer will only emit errors when it can guarantee there is a problem (no false positives). While problematic code can still pass through the analysis, if dialyzer says there is a problem, there definitely is a problem.

As Dialyzer examines your code, it develops constraints on your functions. Dialyzer will assume your functions are being used properly until it can prove that the function will always be wrong. 


### Example: Boolean `and` function

The `and` function in a static language might look something like this: 

{% highlight elixir %}
and :: Bool -> Bool -> Bool
and x y | x == True && y == True = True
        | otherwise = False
{% endhighlight %}

In this static language, the compiler would throw and error at you if you used the function with anything other than a boolean value for both arguments. 

An Elixir variant may look like this:

{% highlight elixir %}
defmodule BooleanFuns do
   def and(true, true) do
    true
  end
  
  def and(false, _) do
    false
  end
  
  def and(_, false) do
    false
  end 
end
{% endhighlight %}

This code would be too uncertain for a static language to compile, but is A-OK in Elixir. Dialyzer is fine with it, too!

Constraint generation occurs with functions that it knows can only be of a certain type. For example, using the `+` is enough information for dialyzer to require this function receive only numbers. Guard clauses like `is_atom` or `is_list` will also develop more constraints for your functions. 

Once the constraints are generated, it is time to solve them, just like a puzzle. The solution to the puzzle is the success typing of the function. Conversely, if no solution is found, the constraints are unsatisfiable and you have a type violation on your hands.


## Using dialyzer
Dialyzer comes with current distributions of Erlang. If you have Elixir, you already have dialyzer.

Dialyzer uses a Persistent Lookup Table (PLT) to keep track of all of your constraints. As it analyzes your code, it includes more information onto the table.

The PLT can be built with the following command:
`dialyzer plt`
Keep in mind that the PLT takes a while to build, but only needs to be performed once. 

Dialyzer can only work on .erl and .beam code, so we have to compile our code in order to use the dialyzer on Elixir code.

## Using [dialyxir](https://hex.pm/packages/dialyxir)
Remembering to compile each time we want to use dialyzer is a chore. Luckily, the dialyxir library includes handy mix tasks to compile and do other favors for us when we use dialyzer. After including dialyxir as a dependency in `mix.exs`, pulling the dependency, and building the PLT, you can run dialyzer through mix:

`mix dialyzer`

Alternatively, you can install dialyxir globally on your machine. 

{% highlight elixir %}
git clone https://github.com/jeremyjh/dialyxir
cd dialyxir
MIX_ENV=prod mix do compile, archive.build, archive.install
{% endhighlight %}

I recommend doing this, so you can use it on any project without adding it to the development dependencies.

## Tip

Remember, you can always check types in `iex` with the `i` function:

{% highlight elixir %}
iex(2)> i "elixir"
Term
  "elixir"
Data type
  BitString
Byte size
  6
Description
  This is a string: a UTF-8 encoded binary. It's printed surrounded by
  "double quotes" because all UTF-8 encoded codepoints in it are printable.
Raw representation
  <<101, 108, 105, 120, 105, 114>>
Reference modules
  String, :binary
iex(3)> i 'elixir'
Term
  'elixir'
Data type
  List
Description
  This is a list of integers that is printed as a sequence of characters
  delimited by single quotes because all the integers in it represent valid
  ASCII characters. Conventionally, such lists of integers are referred to as
  "charlists" (more precisely, a charlist is a list of Unicode codepoints,
  and ASCII is a subset of Unicode).
Raw representation
  [101, 108, 105, 120, 105, 114]
Reference modules
  List
{% endhighlight %}

## Using @specs
Dialyzer doesn't require typespecs to run and catch errors. However, the @specs act as hints for the program and can help it with developing better constraints. The @spec flag will tell dialyzer to include this information in the PLT.


## Sources:
[Dialyzer](http://erlang.org/doc/man/dialyzer.html) documentation

[Dialyxir](https://hex.pm/packages/dialyxir) on Hex.pm

[Learn You Some Erlang](http://learnyousomeerlang.com/dialyzer): Fred Hebert

[Little Elixir & OTP Guidebook](https://www.manning.com/books/the-little-elixir-and-otp-guidebook): Benjamin Tan

[Success Typing Paper](http://www.it.uu.se/research/group/hipe/papers/succ_types.pdf): Tobias Lindahl & Konstantinos Sagonas 
