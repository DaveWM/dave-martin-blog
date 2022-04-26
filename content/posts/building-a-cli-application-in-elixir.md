+++
date = 2022-04-25T23:00:00Z
description = "An of how I built a CLI for Intention in Elixir"
draft = true
title = "Building a CLI Application in Elixir"

+++
In this blog post, I'll recount my experience building a CLI application in [Elixir](https://elixir-lang.org/). I needed to build a CLI for [Intention](https://about.i.ntention.app/), a web app for goal tracking that I wrote last year. The CLI wasn't very complicated - it just needed to authenticate, call a couple of HTTP endpoints in Intention's backend API, and then format the results nicely. This will be a fairly high level overview, if you'd prefer a more detailed step by step guide, I've linked to a couple at the bottom of the page.

**Choosing a Language**

I use [Clojure](https://clojure.org/) for my day-to-day work, but I wanted to try out a different language. Also, Clojure's slow startup time makes it slightly suboptimal for CLI applications. I initially decided on [Haskell](https://www.haskell.org/). I'd previously only written a few small scripts in Haskell, and was eager to see how I fared writing a full application. Unfortunately, to my despair I quickly found myself bogged down in type errors. I quickly abandoned Haskell after realising that either it's too hard to use for small applications, or that I lack the necessary brainpower to use it properly.

I started looking instead for a dynamically typed language that would be suitable . I found this in [Elixir](https://elixir-lang.org/ "Elixir Language"). Elixir is a dynamically typed, functional language with a Ruby style syntax. It runs on the Erlang VM (BEAM) and has been going since 2012, so it's a fairly mature language. The Erlang VM starts up very quickly, so it's a good fit for CLI applications. It has a nice interactive REPL (IEx), plus a few features inspired by Clojure such as macros and protocols. All this made the language very appealing, so I decided to give it a go.

**Getting Started**

Elixir's build tool is called [Mix](https://hexdocs.pm/mix/1.12/Mix.html). Mix manages your project's dependencies, compiles your application, runs tests, and can generate new project skeletons. To get started, all you need to do is install Elixir and Mix, then run `mix new [application name]`. This generates a basic project structure like this:

![](/screenshot-from-2022-04-25-16-28-02.png)

You can then start a REPL by running `iex -S mix`, and run some commands:

![](/iex.gif)

**Parsing Arguments**

Now I had a project set up, I needed a way of parsing command line arguments. I wanted to handle commands like `intention login` and `intention list --all`. I used the excellent Optimus library for this. Optimus made it dead easy to set up multiple subcommands, each with their own allowed arguments. You can also set a help message for each subcommand and argument, which makes it easy to create a great user experience. To get started, it's as simple as calling `optimus = Optimus.new!(...)`, then `Optimus.parse(optimus, args)`. Here's my (slightly shortened) argument parsing code:

```elixir
optimus =
  Optimus.new!(
    name: "intention",
    description: "CLI for Intention",
    version: "0.1.0",
    allow_unknown_args: false,
    parse_double_dash: true,
    subcommands: [
      login: [
        name: "login",
        about: "Log in to Intention. Required to run other commands."
      ],
      list: [
        name: "list",
        about: "List your intentions in a tree format.",
        flags: [
          all: [
            short: "-a",
            long: "--all",
            help: "Show all intentions, including completed."
          ]
        ],
        args: [
          view: [
            value_name: "view",
            help: "View to display",
            parser: fn s ->
              case Integer.parse(s) do
                {:error, _} -> {:error, "invalid view id - should be an integer"}
                {i, _} -> {:ok, i}
              end
            end,
            required: false
          ]
        ]
      ]
    ]
  )

args = Optimus.parse!(optimus, argv)

case args do
  # display help when not given any args
  %{args: %{}} -> Optimus.parse!(optimus, ["--help"])
  {[:login], args} -> handle_auth()
  {[:list], args} -> list_intentions(args)
end
```

And here's what you get when you run `intention --help`:

![](/screenshot-from-2022-04-25-17-35-29.png)

**Making HTTP Requests, with Error Handling**

My next task was to figure out how to make HTTP requests to Intention's JSON API. Luckily, this is very simple in Elixir. I used the [HTTPotion](https://github.com/unrelentingtech/httpotion) library for making the actual requests, plus the [Jason](https://github.com/michalmuskala/jason) library to parse the response JSON. Making a request can then be done like so:

```elixir
HTTPotion.get(url)
|> Map.get(:body)
|> Jason.decode()
```

This works, but unfortunately doesn't account for errors. The HTTP request may fail due to a bad WiFi connection, or perhaps because the response body is not valid JSON. Elixir has a try/catch mechanism for error handling, and it's also common for library functions to return error tuples in the format `{:ok, value} | {:error, reason}`. As in other languages which take this approach, it can be unclear when to use which mechanism. I've found this is especially true when dealing with HTTP requests - should a `500` response trigger an exception, or be returned as `{:error, "error response"}`?

I decided to use error tuples as much as possible. To help me with this, I used the [OK](https://github.com/CrowdHailer/OK) library. `OK` provides some very useful macros for working with error tuples, most notably: 

* [for](https://github.com/CrowdHailer/OK#okfor) - similar to Haskell's `do` notation
* [~>](https://github.com/CrowdHailer/OK#ok-pipe) - a pipe equivalent to `fmap`
* [~>>](https://github.com/CrowdHailer/OK#ok-pipe) - another pipe quivalent to monadic bind (i.e. `>>=`)

I found taking this approach simplified my error handling code. However, due to Elixir's dynamic typing you do have to be careful to use it correctly. It's very easy to accidentally use `~>>` instead of `~>` or `|>`. 

As an example, here's a `handle_response` function I wrote for handling HTTP responses:

```elixir
def handle_response(res) do
  case res do
    %{status_code: 401} ->
      {:error, :unauthorized}

    %{status_code: 200} = r ->
      parse_json_body(r)

    res ->
      OK.for do
        body <- res |> parse_json_body()

        reason <-
          case Map.fetch(body, :reason) do
            {:ok, _} = ok ->
              ok

            :error ->
              {:error, :unknown_error}
          end
      after
        {:error, reason}
      end
  end
end

def parse_json_body(response) do
  response
  |> Map.fetch(:body)
  ~>> Jason.decode(%{keys: :atoms})
end
```

**Output Formatting**

Every CLI app needs a way to nicely format its outputs. For this, I used Elixir's built-in [IO.ANSI](https://hexdocs.pm/elixir/1.12/IO.ANSI.html) module. Using ANSI sequences allow you to do basic text formatting, like outputting bold or coloured text. I also used the [cli_spinners](https://github.com/blackode/elixir_cli_spinners) library to show some fancy loading spinners. Both modules are pretty straightforward to use. Here's an example code snippet, that shows a spinner while waiting for the user to log in:

```elixir
OK.for do
  ["Go to: ", :bright, auth_uri] |> IO.ANSI.format() |> IO.puts()

  token <-
    CliSpinners.spin_fun(
      [frames: :dots, text: "Waiting for token...", done: "Got token"],
      fn -> Auth.poll_token(device_code) end
    )
after
  IO.puts("Login complete")
end
```

The result looks like this:

![](/intention-login.gif)

**Distribution**

Now we come to the final piece of the puzzle - how do you bundle your code into a distributable application? Handily, Elixir comes bundled with the [escript](https://elixirschool.com/en/lessons/intermediate/escripts#building-2) utility for this. You simply put your code in a `main` function, add an `escript` field to your project's `mix.exs`, and then run `mix escript.build`. This will create an executable file. The only caveat I found is that the resulting executable depends on the Erlang VM. I tried out [bakeware](https://github.com/bake-bake-bake/bakeware) and [burrito](https://github.com/burrito-elixir/burrito) to get around this, but neither of them seem to play nicely with my OS (NixOS) and unfortunately I couldn't get them working.

**Summary**

I really liked working with Elixir. I found it easy to get started, IEx is great, and the overall language is well designed and intuitive. If you're using Emacs, [Alchemist](https://alchemist.readthedocs.io/en/latest/) gives you some fantastic tooling, including an integrated REPL. I'd definitely use Elixir again for writing a CLI app. I'd also like to try out [Phoenix](https://www.phoenixframework.org/), a Ruby On Rails style web framework for Elixir. [LiveView](https://github.com/phoenixframework/phoenix_live_view) in particular looks very interesting. If you're looking to try out a new language, I'd highly recommend Elixir. If you want to check out the source code for the Intention CLI, you can find it here. Thanks for reading!

_Further Reading_

[https://hackernoon.com/elixir-console-application-with-json-parsing-lets-print-to-console-b701abf1cb14](https://hackernoon.com/elixir-console-application-with-json-parsing-lets-print-to-console-b701abf1cb14 "https://hackernoon.com/elixir-console-application-with-json-parsing-lets-print-to-console-b701abf1cb14")

[https://medium.com/blackode/writing-the-command-line-application-in-elixir-78a8d1b1850](https://medium.com/blackode/writing-the-command-line-application-in-elixir-78a8d1b1850 "https://medium.com/blackode/writing-the-command-line-application-in-elixir-78a8d1b1850")