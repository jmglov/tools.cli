# tools.cli

Tools for working with command line arguments, ported from
[clojure.tools.cli](https://github.com/clojure/tools.cli).

## Dependency Information

Add to your [Dust](https://github.com/pixie-lang/dust) `project.edn` file:

```clojure
:dependencies [[pixie-lang/tools.cli "0.3.4-alpha"]]
```

## Quick Start

```clojure
(ns my.program
  (:require [tools.cli :refer [parse-opts]]))

(def cli-options
  ;; An option with a required argument
  [["-p" "--port PORT" "Port number"
    :default 80
    :parse-fn read-string
    :validate [#(< 0 % 0x10000) "Must be a number between 0 and 65536"]]
   ;; A non-idempotent option
   ["-v" nil "Verbosity level"
    :id :verbosity
    :default 0
    :assoc-fn (fn [m k _] (update-in m [k] inc))]
   ;; A boolean option defaulting to nil
   ["-h" "--help"]])

(parse-opts program-arguments cli-options)
```

Execute the command line:

    my-program -vvvp8080 foo --help --invalid-opt

to produce the map:

```clojure
{:options   {:port 8080
             :verbosity 3
             :help true}

 :arguments ["foo"]

 :summary   "  -p, --port PORT  80  Port number
               -v                   Verbosity level
               -h, --help"

 :errors    ["Unknown option: \"--invalid-opt\""]}
```

**Note** that exceptions are _not_ thrown on parse errors, so errors must be
handled explicitly after checking the `:errors` entry for a truthy value.

Please see the [example program](#example-usage) for a more detailed example
and refer to the docstring of `parse-opts` for comprehensive documentation.

## New Features from clojure.tools.cli 0.3.x

### Better Option Tokenization

In accordance with the [GNU Program Argument Syntax Conventions][GNU], two
features have been added to the options tokenizer:

* Short options may be grouped together.

  For instance, `-abc` is equivalent to `-a -b -c`. If the `-b` option
  requires an argument, the same `-abc` is interpreted as `-a -b "c"`.

* Long option arguments may be specified with an equals sign.

  `--long-opt=ARG` is equivalent to `--long-opt "ARG"`.

  If the argument is omitted, it is interpreted as the empty string.
  e.g. `--long-opt=` is equivalent to `--long-opt ""`

[GNU]: https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html

### In-order Processing for Subcommands

Large programs are often divided into subcommands with their own sets of
options. To aid in designing such programs, `clojure.tools.cli/parse-opts`
accepts an `:in-order` option that directs it to stop processing arguments at
the first unrecognized token.

For instance, the `git` program has a set of top-level options that are
unrecognized by subcommands and vice-versa:

    git --git-dir=/other/proj/.git log --oneline --graph

By default, `clojure.tools.cli/parse-opts` interprets this command line as:

    options:   [[--git-dir /other/proj/.git]
                [--oneline]
                [--graph]]
    arguments: [log]

When :in-order is true however, the arguments are interpreted as:

    options:   [[--git-dir /other/proj/.git]]
    arguments: [log --oneline --graph]

Note that the options to `log` are not parsed, but remain in the unprocessed
arguments vector. These options could be handled by another call to
`parse-opts` from within the function that handles the `log` subcommand.

### Options Summary

`parse-opts` returns a minimal options summary string:

      -p, --port NUMBER  8080       Required option with default
          --host HOST    localhost  Short and long options may be omitted
      -d, --detach                  Boolean option
      -h, --help

This may be inserted into a larger usage summary, but it is up to the caller.

If the default formatting of the summary is unsatisfactory, a `:summary-fn`
may be supplied to `parse-opts`. This function will be passed the sequence
of compiled option specification maps and is expected to return an options
summary.

The default summary function `clojure.tools.cli/summarize` is public and may
be useful within your own `:summary-fn` for generating the default summary.

### Option Argument Validation

There is a new option entry `:validate`, which takes a tuple of
`[validation-fn validation-msg]`. The validation-fn receives an option's
argument *after* being parsed by `:parse-fn` if it exists.

    ["-p" "--port PORT" "A port number"
     :parse-fn #(Integer/parseInt %)
     :validate [#(< 0 % 0x10000) "Must be a number between 0 and 65536"]]

If the validation-fn returns a falsy value, the validation-msg is added to the
errors vector.

### Error Handling and Return Values

Instead of throwing errors, `parse-opts` collects error messages into a vector
and returns them to the caller. Unknown options, missing required arguments,
validation errors, and exceptions thrown during `:parse-fn` are all added to
the errors vector.

Correspondingly, `parse-opts` returns the following map of values:

    {:options     A map of default options merged with parsed values from the command line
     :arguments   A vector of unprocessed arguments
     :summary     An options summary string
     :errors      A vector of error messages, or nil if no errors}

During development, parse-opts asserts the uniqueness of option `:id`,
`:short-opt`, and `:long-opt` values and throws an error on failure.

## Developer Information

* [GitHub project](https://github.com/pixie-lang/tools.cli)

## License

Copyright (c) Rich Hickey and contributors. All rights reserved.

The use and distribution terms for this software are covered by the
Eclipse Public License 1.0 (http://opensource.org/licenses/eclipse-1.0.php)
which can be found in the file epl.html at the root of this distribution.
By using this software in any fashion, you are agreeing to be bound by
the terms of this license.

You must not remove this notice, or any other, from this software.
