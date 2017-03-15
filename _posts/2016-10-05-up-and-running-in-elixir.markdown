---
title: "Up and Running in Elixir"
layout: post
date: 2016-10-05 13:34
image: '/assets/images/'
description: All the ways you can get working programs in Elixir.
tag: elixir, OTP, tutorial
blog: true
jemoji:
author: geoff
---
Below is the general presentation from the December 2016 [Houston Elixir Meetup](https://www.meetup.com/Houston-Elixir-Meetup/events/234133414/) on writing code that can run and be distributed. 

# Runnable Code
Includes some of the most common ways to run elixir code in your development environment, and ways of creating OTP Releases.

## Outline:

* Compile from command line / REPL
* Running Elixir Script (`.exs`)
* Compile from Mix
* Application behaviour
* Escript
* Releases & Distillery
* Final Thoughts, Caveats



## Compile From Command Line/REPL
The REPL / iex is a great place to test out expressions and general functionality of a module in real time. If you have a file in Elixir that you want to compile and load into the REPL, use the `c/1` command. 
We're using [DemoModule](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/blob/master/without_mix/demo_module.ex), a contrived example.

{% highlight elixir %}
$ iex
Erlang/OTP 19 [erts-8.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.3.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> c "demo_module.ex"
[DemoModule]
iex(2)> DemoModule.say_it
Hello from Elixir.DemoModule
:ok
{% endhighlight %}

Additionally, the user can keep the same iex session after making changes to code. Use command `recompile/0` from iex.



## Running Elixir Script (.exs)
Elixir can run in an interpreted form. These files have the extension `.exs` to differentiate from compilable Elixir code. `.exs` files are typically used for configuration. These files can be exectured directly from the command line by calling on Elixir to run, or by loading it into iex. Note that loading the script into iex will execute the script eagerly.

{% highlight elixir %}
$ elixir demo_script.exs
Hello from an Elixir script!
{% endhighlight %}

or

{% highlight elixir %}
iex(1)> c "demo_script.exs"
Hello from an Elixir script!
[]
iex(2)>
{% endhighlight %}
[Code example of script here](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/blob/master/without_mix/demo_script.exs).

## Using Mix
Once your codebase expands beyond a few modules, or you start to include dependencies, you will want to start using a tool to manage the code. Elixir ships with Mix, which functions as a test runner, dependency manager, and a build tool. 

Make a new project with `$ mix new app_name`. This will create a directory structure, along with config files, a place for your dependencies and beam files to live, and a test directory. [View module examples here](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/tree/master/with_mix).

You can even start iex sessions and let mix compile and load in your modules and listed dependencies:

{% highlight elixir %}
$ iex -S mix
Erlang/OTP 19 [erts-8.1] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiling 1 file (.ex)
Generated with_mix app
Interactive Elixir (1.3.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> WithMix.say_it
Hello from Elixir.WithMix
:ok
{% endhighlight %}

## Detour: Using the Application Behaviour
The `Application` behaviour is used typically in a module that exposes a public-facing API. `Application` is responsible for starting your supervision trees, and calling any cleanup code when your application exits. [Example of directory and modules here](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/tree/master/demo_application).


The new application module looks like this (`demo_application.ex`):

{% highlight elixir %}
defmodule DemoApplication do
  alias DemoApplication.{DemoSupervisor, DemoModule}
  use Application 

  def start(_type, _args) do
    DemoSupervisor.start_link()
  end

  def use_module() do
    DemoModule.say_it(:demo_module)
  end
end
{% endhighlight %}



You would also specify this module in the `application/0` function in your `mix.exs` file:

{% highlight elixir %}
  def application do
    [
      applications: [:logger],
      mod: {DemoApplication, []}
    ]
  end
{% endhighlight %}

Mix will use the `start/2` callback in the module you specify, along with any arguments specified in the second element of the tuple. 

All subsequent examples will use both Mix and the Application behaviour.

## Escripts
Generates an executable from BEAM files. Requires the user already have the VM installed.

To use an Escript, you will need a separate module to run the code from the script. We'll create a module called `demo_CLI.ex` in our lib directory. It requires a function called `main` that accepts arguments from the command line. The module looks like this:

{% highlight elixir %}
defmodule DemoApplication.CLI do
  alias DemoApplication.DemoModule

  def main(args) do
    args |> parse_args() |> do_process()
  end

  def parse_args(args) do
    options = OptionParser.parse(args, aliases: [h: :help])

    case options do
      {[help: true], _, _} -> :help
      _ -> :run
    end
  end

  def do_process(:run) do
    DemoModule.say_it(:demo_module)
  end

  def do_process(:help) do
    IO.puts """
    Welcome to Demo App.

    """
  end
end
{% endhighlight %}


In `mix.exs`, include `escript: escript` within the `project/0` function and create a new `escript/0` function: 

{% highlight elixir %}
def escript, do: [main_module: HouTax.CLI]
{% endhighlight %}


Then compile the script:
{% highlight elixir %}
$ mix escript.build
Compiling 4 files (.ex)
Generated demo_application app
Generated escript demo_application with MIX_ENV=dev
{% endhighlight %}

This creates an executable called `demo_application`. You can run it immediately with `./demo_application`. [View full directory and module here](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/tree/master/demo_application_escript).



## Releases with Distillery
[Distillery](https://hex.pm/packages/distillery) is a library by Paul Schoenfelder (bitwalker), and is a ground-up rewrite/replacement of his popular repository ExRM (Elixir Release Manager). Distillery autmoates a lot of the process to generate OTP releases. 

An OTP release includes your compiled BEAM files, as well as some form of the Erlang run time system (ERTS/erts). Should be compiled on the target OS and CPU architecture. Optionally, you can elect to produce the release without the runtime, but you will need to ensure Erlang is installed on the targeted host machine.

We'll go through the minimum here, but [the documentation is very thorough](https://hexdocs.pm/distillery/walkthrough.html). 

[Example project here](https://github.com/HTX-Elixir-Meetup/2016-10-05-runnable-code/tree/master/demo_application_distillery).
Include distillery in your mixfile's dependencies:
{% highlight elixir %}
  defp deps do
    [{:distillery, "~> 0.10.0"}]
  end
{% endhighlight %}

and fetch the dependencies with `$ mix deps.get && mix compile`

Then, run `mix release.init` to generate a new `/rel` directory with a config file inside.

At this point, you can edit the config file before making the final release. 

Run `mix release` to generate you new release: 
For Development:
{% highlight elixir %}
$ mix release
==> Assembling release..
==> Building release demo_application:0.1.0 using environment dev
==> You have set dev_mode to true, skipping archival phase
==> Release successfully built!
    You can run it in one of the following ways:
      Interactive: rel/demo_application/bin/demo_application console
      Foreground: rel/demo_application/bin/demo_application foreground
      Daemon: rel/demo_application/bin/demo_application start
{% endhighlight %}

For Production:
{% highlight elixir %}
$ mix release --env=prod
==> Assembling release..
==> Building release demo_application:0.1.0 using environment prod
==> Including ERTS 8.1 from /usr/local/Cellar/erlang/19.1/lib/erlang/erts-8.1
==> Packaging release..
==> Release successfully built!
    You can run it in one of the following ways:
      Interactive: rel/demo_application/bin/demo_application console
      Foreground: rel/demo_application/bin/demo_application foreground
      Daemon: rel/demo_application/bin/demo_application start
{% endhighlight %}

The production variant will produce a gzipped tarball that you can deploy on your target machine.


## Final Thoughts, Caveats
Thanks for taking the ride on this tour of creating executable files in Elixir. As you can see, there are a number of options of generating code from quick throwaway scripts to full applications that can be run on machines that don't even have Erlang installed.

This is a surface-level view of the mosst common ways you can get your code into an executable state or out into the world. However, there is much much more to using Elixir/Erlang including running multiple nodes, running an application in the background with mix in `--detached` mode, hot upgrades with releases, and tuning the VM (like raising the maximum number of processes). 


## Useful sources: 
* Programming Elixir (Dave Thomas, Pragmatic Bookshelf) [Chapter 1](https://media.pragprog.com/titles/elixir/introduction.pdf)
* [Elixir in Action](https://www.manning.com/books/elixir-in-action) (Sasa Juric, Manning) Chapters 11-13 are probably the most useful for using mix, the Application behaviour, escripts, and releases. 
* [Distillery Documentation](https://hexdocs.pm/distillery/getting-started.html)
* Elixir-Lang.org's [Getting Started Guide](http://elixir-lang.org/getting-started/introduction.html)


