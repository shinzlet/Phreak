# PSA
Since this library was written, [RX14](https://github.com/RX14) and others have
integrated many of Phreak's features into [`OptionParser`](https://crystal-lang.org/api/latest/OptionParser.html), the CLI builder in the
standard library. I'm thrilled to see that these changes have been made, but
Phreak now brings much less to the table than it did in it's hayday.

So, before using Phreak, please check out `OptionParser`'s updated feature set - 
you likely don't need a library at all!

# Phreak
[![Crystal CI](https://github.com/shinzlet/phreak/actions/workflows/crystal.yml/badge.svg)](https://github.com/shinzlet/phreak/actions/workflows/crystal.yml)

Phreak is a CLI builder in the style of Crystal's builtin OptionParser. It aims
to provide greater flexibility by integrating subcommands natively, all while
retaining a very simple callback based code style.

If you use Phreak in a project, please let me know! I'd love to see what you've
made, and would be happy to put your project in the [examples](##examples)
section. (My email address is in my github profile.)

## Table of contents

- [Features](#features)
- [Installation](#installation)
- [Examples](#examples)
- [Development](#development)
- [Contributing](#contributing)
- [Contributors](#contributors)

## Features

- [Basic flags](#basic-commands)
- [Automatic documentation](#automatic-documentation)
- [Subcommands](#subcommands)
- [Nested subcommands](#nested-subcommands)
- [Command types](#command-types)
- [Fuzzy matching](#fuzzy-matching)
- [Compound flags](#compound-flags)
- [Default actions](#default-actions)
- [Basic error handling](#basic-error-handling)
- [Advanced Help Menus](#advanced-help-menus)
- [Error bubbling](#error-bubbling)
- [Reusable parsers](#reusable-parsers)
- [Planned features](#planned-features)

Much like OptionParser, Phreak makes registering commands incredibly easy:

### Basic commands

```crystal
require "phreak"

Phreak.parse! do
  # This will respond to the arguments "-a" or "--flag1"
  root.bind(short_flag: 'a', long_flag: "flag1") do
    puts "Someone called?"
  end
end
```

### Automatic Documentation
You can also attach a description to a command to easily print a help menu. These are optional,
but documenting a CLI as you work is rather convenient.
```crystal
require "phreak"

Phreak.parse! do |root|
  # Sets the helpmenu banner for `root`
  root.banner = "A cli."

  root.bind(short_flag: 'd', long_flag: "do-nothing",
            description: "Has no effect.") do
  end

  root.bind(short_flag: 'h', long_flag: "help",
            description: "Prints a help menu.") do
    # Printing a `Subparser` (the objects that Phreak yields) will
    # display a formatted help menu showing the commands bound to that
    # object and their descriptions. In this case, as `root` is bound to
    # two things, both will be printed with invocation instructions and
    # descriptions.
    puts root
  end
end
```

The block above, when invoked `cli -h`, will print the following menu:

```
A cli.

Commands:
    --do-nothing, -d          Has no effect.
    --help, -h                Prints a help menu.
```

There are some more advanced formatting tricks that can be used, here -
see [advanced help menus](#advanced-help-menus) for more information. Also,
for more information about how to use these menus in a more complex way,
see [nested subcommands.](#nested-subcommands)

The above two features capture most of the scope of OptionParser. However, you're
probably here for more extensive functionality! Let's get to the good stuff.

### Subcommands

Phreak allows you to create clear, human-readable CLIs with ease, thanks to
subcommands.

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(word: "throw") do |sub|
    # Responds to "./binary throw -p" or "subcommand throw --party"
    sub.bind(short_flag: 'p', long_flag: "party") do
      puts "Whoo!"
    end
  end

  root.bind(word: "info") do |sub|
    # Responds to "./binary info -d" or "subcommand info --dogs"
    sub.bind(short_flag: 'd', long_flag: "dogs") do
      puts "dogs are just incredible."
    end
  end
end
```

### Nested subcommands

Building on that more fluent subcommand syntax mentioned above, Phreak also lets
you create heirarchical commands (think `nmcli device wifi connect ...`, for
example).

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(word: "wifi", description: "Configure or control wireless connections.") do |wifi|
    wifi.banner = "Manage wireless connections. Invocation: cli wifi [subcommand]"

    wifi.bind(word: "status", description: "Get the current status of the modem.") do
      # Reponds to "nested wifi status"
    end

    wifi.bind(word: "set", description: "Set the the wifi state.") do |set|
      # Responds to "nested wifi set"
    end

    wifi.bind(word: "help",
              description: "display specific help about the `wifi` subcommand.") do |help|
      puts wifi
    end
  end

  root.bind(word: "help", description: "display this help menu.") do |help|
    puts root
  end
end
```

This command also demonstrates how to use documentation with nested commands! Each
subcommand has a help command defined on it, which allows invocation like
`./binary help` for general help, or `./binary wifi help` for more specific
information!

### Command types

Phreak allows you to create three primary types of commands:

- Short flags (e.g. `ls -a`)
- Long flags (e.g. `ls --all`)
- Words (e.g. `git push`)

You can alias one of each to a command in one line, too:

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(short_flag: 'a', long_flag: "all", word: "all") do
    # Responds to -a, --all, or 'all'
  end
end
```

### Fuzzy matching

Phreak allows you to fuzzy match commands as desired! For example:

```crystal
require "phreak"

Phreak.parse! do |root|
  root.fuzzy_bind(word: "enable") do |sub, match|
    # Responds to words close to enable - enab, enablt, for example
    puts "Fuzzy matched #{match}"
  end
end
```

### Compound flags

Many CLIs allow flags to be stacked to run several processes in a row. For
example, `ls -al` tells ls to print **a**ll files in a **l**ong format.

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(short_flag: 'a') do
    puts "A!"
  end

  root.bind(short_flag: 'b') do
    puts "B!"
  end

  root.bind(short_flag: 'c') do
    puts "C!"
  end
end
```

Given the above program, compiled to *binary*:

```sh
./binary -abc     # Prints "A!B!C!"
./binary -ac      # Prints "A!C!"
./binary -bbb     # Prints "B!B!B!"
```

### Default actions

In some cases, CLIs should have a default behaviour only when no arguments are
provided. Phreak makes that easy:

```crystal
require "phreak"

Phreak.parse! do |root|
  root.default do
    puts "no arguments provided"
  end
end
```

### Basic error handling

Phreak makes it easy to detect incorrect usage of your CLI.

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(word: "say") do |sub|
    sub.bind(word: "hi") do
      puts "Hi!"
    end
  end

  root.default do
    puts "No arguments provided"
  end

  root.missing_args do |apex|
    puts "Missing an argument after #{apex}"
  end

  root.unrecognized_args do |arg|
    puts "Unrecognized argument: #{arg}"
  end
end
```

Here, if we run `./binary say`, the `missing_args` handler will be called. If we
ran `./binary say goodbye`, the `unrecognized_args` handler would be run.
If we don't provide an argument at all, the `default` handler is called.
Finally, if we correctly invoke `./binary say hi`, the CLI will print out "Hi!"

### Advanced Help Menus
Help menus can be tweaked in many ways. All the configuration is done on a
per-subparser basis, which allows help menus to be tailored to subcommands.
As described before, subparsers can be given a custom command title with the
`banner` property:

```crystal
subparser.banner = "this is a command"
```

Indentation, too, can be configured. Below is a snippet of `Subparser.cr`
showing the relevant properties:

```crystal
# An optional banner to display in the autogenerated help menu.
property banner : String | Nil

# The left indentation unit used in printing default help menus.
property help_indent : String = " " * 4

# The minumum width (word, long flag, and spaces) of the command section of
# the default help menu.
property command_section_width : Int8 = 30

# If the command section exceeds the width defined above, the fallback
# padding will be added to ensure there is at least some separation
# between command and description.
property fallback_description_padding : String = " " * 4
```

### Error bubbling

Due to the way that Phreak works, you can actually bind a `missing_args` or
`unrecognized_args` handler to any part of a nested command. For example:

```crystal
require "phreak"

Phreak.parse! do |root|
  root.bind(word: "say") do |sub|
    sub.bind(word: "hi") do
      puts "Hi!"
    end

    sub.bind(word: "hello") do |sub|
      sub.bind(word: "tomorrow") do
        puts "Sure thing!"
      end

      sub.missing_args do
        puts "Hello!"
      end

      sub.unrecognized_args do |arg|
        puts "When's #{arg}?"
      end
    end

    sub.unrecognized_args do |arg|
      puts "I can't say #{arg}!"
    end
  end
end
```

Let's see what happens in a few example inputs:

`./binary say hi`

- The binding for 'say' is recognized, and a `Subparser` (the `sub` variable) is created.
- The binding for 'hi' is invoked, and the program terminates.

`./binary say goodbye`

- The binding for 'say' is recognized.
- None of the bindings match 'goodbye', so the `unrecognized_args` handler on `say` is invoked.

`./binary say hello`

- The binding for 'say' is recognized.
- The binding for 'hello' is recognized.
- We tell the `Subparser` to bind the word 'tomorrow' to a callback, but we're
    out of keywords!
- The `missing_args` handler is invoked in the scope of the 'hello' binding.
- The handler says hello!

The lattermost case may seem very similar to the `default` handler, but there
are some important differences. `default` is only ever called if the size of the
arguments array is zero (there are absolutely no arguments). It can also only be
bound to the root `Parser` (which is actuallly a `Subparser+`).

Errors also bubble - that is, if we had not defined a `missing_args` handler on
'hello', Phreak would then try to invoke a `missing_args` event on 'say', then
on the root subparser. If no handlers are found on the traversal back up, an
exception is raised.

### Reusable parsers
What we've seen above is incredibly useful for non-interactive CLIs, but
it isn't always the case that you want to throw away your parser after the
first use. Here's an example of how you can make an interactive CLI
with Phreak:

```crystal
require "phreak"

# Ignore this variable for now - it is specific to this example.
running = true

# This doesn't actually run the parser at all -
# it just returns a convenient handle to it.
parser = Phreak.create_parser do |root|
  root.bind(word: "create") do |sub|
    sub.grab do |sub, name|
      puts "Creating partition #{name}"
    end
  end

  root.bind(word: "quit") do
    puts "Exiting."
    running = false
  end
end

puts "Disk formatting utility"

# Here's where we will actually run the parser!
while running
  # Print a prompt, then read the next line.
  printf ">> "
  next_line = gets || ""

  parser.parse(next_line.split(" "))
end
```

This example is quite different than the others, but it showcases some
fascinating things. We are creating a reusable parser using 
`Phreak.create_parser`, and creating our bindings once. Then, later in
the program, we are repeatedly calling that parser on the user's input.
When the user enters the 'quit' command, the matching endpoint is run
in the parser, which sets `running` to be false, thus terminating the
loop.

Here is some example output:

```
Disk formatting utility
>> create p1    
Creating partition p1
>> foobar
Unrecognized command foobar.
>> create
The command create requires more parameters.
>> quit
Exiting.
```

## Planned features

- Autogenerated documentation (at compile time)
- Autogenerated fish completions
- Typo fix suggestions (did you mean 'commit'?)

Check out the [project Trello](https://trello.com/b/23uhIc6G/phreak) for more.

## Installation

1. Add phreak to your `shard.yml`:

   ```yaml
   dependencies:
     phreak:
       github: shinzlet/phreak
   ```

2. Run `shards install`

## Examples

I wrote [sd](https://github.com/shinzlet/sd) using Phreak! It was a great case
study of what features I needed to add to get a truly flexible CLI builder.

## Development

Please see [phreak's vivisection](vivisection.md) for an overview of how Phreak
works under the hood!

## Contributing

1. Fork it (<https://github.com/your-github-user/./fork>)
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request

## Contributors

- [shinzlet](https://github.com/shinzlet) - creator and maintainer
