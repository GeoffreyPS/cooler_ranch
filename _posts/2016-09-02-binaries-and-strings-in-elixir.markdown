---
title: "Binaries and Strings in Elixir"
layout: post
date: 2016-11-02 13:12
image: '/assets/images/'
description: "Presentation on Strings and Binaries in Elixir for Houston Elixir Meetup"
tag: Elixir, strings
blog: true
jemoji:
author: geoff
---
Below is the general presentation from the November 2016 [Houston Elixir Meetup](https://www.meetup.com/Houston-Elixir-Meetup/events/234134327/) on strings and binaries. It synthesizes a lot of sources about strings on the BEAM, but provides a quick and dirty guide to how strings/binaries work on this platform.

The talk was given days after Halloween, so we indulged in spooky examples.

# Binaries and strings

## What are binaries? 
  Strings are UTF-8 Encoded binaries.
  
  Binaries are a sequence of bytes, denoted between the `<<` and `>>` angle brackets. 


  A Binary is a bitstring where the number of bits is divisible by 8.

  Binaries that are valid UTF - 8 will be represented as strings.

  Strings have a data structure and a printable representation.

{% highlight elixir %}
  iex> <<71, 45, 71, 45, 71, 45, 71, 72, 79, 83, 84>>
  "G-G-G-GHOST"
{% endhighlight %}

  This example shows when you enter the codepoints into IEX, you receive a representation as a string. 

  You can also get a character's codepoint by using the `?` operator.

{% highlight elixir %}
  iex> ?👻
  128123
{% endhighlight %}

  Elixir also supports a shorthand for referring to unicode characters:

{% highlight elixir %}
  iex> "\u0068"
  "h"
{% endhighlight %}

## Graphemes vs. codepoints
  The data structure is composed of codepoints, which can combine to create individual graphemes.
  In unicode, some codepoints are used to modify other characters, like `é`.

{% highlight elixir %}
  iex> string = "\u0065\u0301"
  "é"
  iex> String.codepoints string
  ["e", "́"]
{% endhighlight %}

  Keep in mind that in order to know a string's length (`String.length/1`, Elixir will have to traverse this sequence of bytes to return the number of printable graphemes from your data (and O(N) operation). You can also use `Kernel.byte_size/1` to know the size of a given binary, which is an O(1) operation.  
  
{% highlight elixir %}
  iex> byte_size("é")
  2
  iex> String.length("é")
  1
{% endhighlight %}

### There be dragons 🐉

  Some characters can be represented as one codepoint or as a combination of multiple codepoints:
  
{% highlight elixir %}
  iex> {combined, individual} = {"e\u0308", "\u00EB"}
  {"ë", "ë"}

  iex> combined == individual
  false

  iex> String.equivalent?(combined, individual)
  true
{% endhighlight %}


## Charlists
  Charlists are typically used for compatability reasons with Erlang. They are represented as single-quoted strings.

{% highlight elixir %}
  iex> "Spooky scary skeletons" |> String.to_charlist
  'Spooky scary skeletons'
{% endhighlight %}

  One advantage/feature of this is that you can iterate over the sequence with functions you would use to work with lists.

{% highlight elixir %}
  iex> spooky_charlist = 'Spooky scary skeletons'
  'Spooky scary skeletons'

  iex> hd spooky_charlist
  83

  iex> <<83>>
  "S"

  iex> spooky_string = "Spooky scary skeletons"
  "Spooky scary skeletons"

  iex> hd spooky_string
  ** (ArgumentError) argument error
      :erlang.hd("Spooky scary skeletons")

  iex> is_list spooky_string
  false

  iex> is_list spooky_charlist
  true
{% endhighlight %}

## Large binary space

  Large binaries (> 64 bytes) stored in separate area from a process' heap.
  
  This allows for fast message passing as only pointer sent between processes

  Can save a lot of memory as well (no needless copying)
  
  Can be long delay before being reclaimed by GC. The large binary has a counter of all of the other processes that are using it. In order to be garbage collected, all processes which have “seen” the binary must first do a garbage collection.
  
  If garbage collection does not happen often enough, the large binaries can grow and crash system.


  If your process is only referencing a tiny part of a large binary, it might be worthwhile to use `:binary.copy` to explicitly copy the portion you need from a large binary so that the large binary can be garbage collected sooner. [More info in erlang's docs](http://erlang.org/doc/man/binary.html#copy-1).

  See also Evan Miller's [Template of Doom](http://www.evanmiller.org/elixir-ram-and-the-template-of-doom.html) post.


## IO Lists

  Concatenating strings costs memory to compose items into a single string. Since our data is immutable, it can be cheaper and faster to work with lists of references that can easily be appended. 

  IO lists allow you to create sequences of binaries and other IO lists, which will ultimately be flattened when the list is written (to disk for example). This has advantages because that final string never requires memory within your application. It only exists in the destination. Additionally, IO lists work with lists have have layers of nesting. 

{% highlight elixir %}
  iex> first = "David "
  "David "
  iex> mi = "S. "
  "S. "
  iex(41)> last = "Pumpkins"
  "Pumpkins"
  iex> IO.puts [[first, [mi]], last]
  David S. Pumpkins
  :ok
{% endhighlight %}


## Additional Sources / Further Reading
- [Elixir and Unicode, Part 2: Working with Unicode Strings](https://www.bignerdranch.com/blog/elixir-and-unicode-part-2-working-with-unicode-strings/) from Nathan Long / Big Nerd Ranch 
- [Hitchhiker's Guide to the BEAM](https://www.youtube.com/watch?v=_Pwlvy3zz9M)  Lecture from Robert Virding, Includes information on the large binary space on the BEAM, and lots of other information about the VM.
- [Erlang's binary manpage](http://erlang.org/doc/man/binary.html)
- [String Module](https://hexdocs.pm/elixir/String.html#content) from Elixir.
- [Template of Doom](http://www.evanmiller.org/elixir-ram-and-the-template-of-doom.html) from Evan Miller
