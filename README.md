![Erlang Versions][erlang version badge]
![Build Tool][build tool]

Meck
====

A mocking library for Erlang.

  * [Features](#features)
  * [Examples](#examples)
  * [Use](#use)
  * [Build](#build)
  * [Caveats](#caveats)
  * [Contribute](#contribute)

Notice
------

This is [Basho's](http://www.basho.com) fork of Meck.
For the original version, please use the [upstream repository](http://eproxus.github.com/meck).

This file is edited in in accordance with our changes, which include:

* Builds _only_ with [Rebar 3][rebar_3]<br />
  Files supporting `Rebar 2` and `make` have been removed.
* The supported/tested versions of Erlang/OTP may differ from the upstream version.
* We do not presently publish hex packages.
* Our fork is not presently built with Travis CI.

Features
--------

See what's new in [0.8 Release Notes][release_notes_0.8].

  * Dynamic return values using sequences and loops of static values
  * Compact definition of mock arguments, clauses and return values
  * Pass through: call functions in the original module
  * Complete call history showing calls, return values and exceptions
  * Mock validation, will invalidate mocks that were not called correctly
  * Throwing of expected exceptions that keeps the module valid
  * Throws an error when mocking a module that doesn't exist or has been
    renamed (disable with option `non_strict`)
  * Support for [Hamcrest][hamcrest] matchers
  * Automatic backup and restore of cover data
  * Mock is linked to the creating process and will unload automatically
    when a crash occurs (disable with option `no_link`)
  * Mocking of sticky modules (using the option `unstick`)

Examples
--------
Here's an example of using Meck in the Erlang shell:

```erl
Eshell V5.8.4  (abort with ^G)
1> meck:new(dog, [non_strict]). % non_strict is used to create modules that don't exist
ok
2> meck:expect(dog, bark, fun() -> "Woof!" end).
ok
3> dog:bark().
"Woof!"
4> meck:validate(dog).
true
5> meck:unload(dog).
ok
6> dog:bark().
** exception error: undefined function dog:bark/0
```

Exceptions can be anticipated by Meck (resulting in validation still passing).
This is intended to be used to test code that can and should handle certain
exceptions indeed does take care of them:

```erl
5> meck:expect(dog, meow, fun() -> meck:exception(error, not_a_cat) end).
ok
6> catch dog:meow().
{'EXIT',{not_a_cat,[{meck,exception,2},
                    {meck,exec,4},
                    {dog,meow,[]},
                    {erl_eval,do_apply,5},
                    {erl_eval,expr,5},
                    {shell,exprs,6},
                    {shell,eval_exprs,6},
                    {shell,eval_loop,3}]}}
7> meck:validate(dog).
true
```

Normal Erlang exceptions result in a failed validation. The following example is
just to demonstrate the behavior, in real test code the exception would normally
come from the code under test (which should, if not expected, invalidate the
mocked module):

```erl
8> meck:expect(dog, jump, fun(Height) when Height > 3 ->
                                  erlang:error(too_high);
                             (Height) ->
                                  ok
                          end).
ok
9> dog:jump(2).
ok
10> catch dog:jump(5).
{'EXIT',{too_high,[{meck,exec,4},
                   {dog,jump,[5]},
                   {erl_eval,do_apply,5},
                   {erl_eval,expr,5},
                   {shell,exprs,6},
                   {shell,eval_exprs,6},
                   {shell,eval_loop,3}]}}
11> meck:validate(dog).
false
```

Here's an example of using Meck inside an EUnit test case:

```erlang
my_test() ->
    meck:new(my_library_module),
    meck:expect(my_library_module, fib, fun(8) -> 21 end),
    ?assertEqual(21, code_under_test:run(fib, 8)), % Uses my_library_module
    ?assert(meck:validate(my_library_module)),
    meck:unload(my_library_module).
```

Pass-through is used when the original functionality of a module should be kept.
When the option `passthrough` is used when calling `new/2` all functions in the
original module will be kept in the mock. These can later be overridden by
calling `expect/3` or `expect/4`.

```erl
Eshell V5.8.4  (abort with ^G)
1> meck:new(string, [unstick, passthrough]).
ok
2> string:strip("  test  ").
"test"
```

It's also possible to pass calls to the original function allowing us to
override only a certain behavior of a function (this usage is compatible with
the `passthrough` option). `passthrough/1` will always call the original
function with the same name as the expect is defined in):

```erl
Eshell V5.8.4  (abort with ^G)
1> meck:new(string, [unstick]).
ok
2> meck:expect(string, strip, fun(String) -> meck:passthrough([String]) end).
ok
3> string:strip("  test  ").
"test"
4> meck:unload(string).
ok
5> string:strip("  test  ").
"test"
```

Use
---

Meck is best used via [Rebar 3][rebar_3]. Add the following dependency to your
`rebar.config` in your project root:

```erlang
{deps, [
    {meck,
        {git, "https://github.com/basho/meck.git",
        {branch, "feature/riak-2903/rebar3"} }}
]}.
```

Build
-----

Meck requires [Rebar 3][rebar_3] to build.
To build Meck go to the Meck directory and simply type:

```sh
rebar3 as prod compile
```
> _The 'prod' Rebar3 profile is used automatically when meck is a dependency, so that's our standard target. The default profile will work, but isn't as strict about warnings._

In order to run all tests for Meck type the following command from the same
directory:

```sh
rebar3 eunit
```

Two things might seem alarming when running the tests:

  1. Warnings emitted by cover
  2. An exception printed by SASL

Both are expected due to the way Erlang currently prints errors. The important
line you should look for is `All XX tests passed`, if that appears all is
correct.

Documentation can be generated through the use of the following command:

```sh
rebar3 edoc
```

Caveats
-------

Meck will have trouble mocking certain modules since Meck works by recompiling
and reloading modules. Since Erlang have a flat module namespace, replacing a
module has to be done globally in the Erlang VM. This means certain modules
cannot be mocked. The following is a non-exhaustive list of modules that can
either be problematic to mock or not possible at all:

* `erlang`
* `os`
* `crypto`
* `compile`
* `global`
* `timer` (possible to mock, but used by some test frameworks, like Elixir's
  ExUnit)

Also, a meck expectation set up for a function _f_ does not apply to the module-
local invocation of _f_ within the mocked module. Consider the following module:

```
-module(test).
-export([a/0, b/0, c/0]).

a() ->
  c().

b() ->
  ?MODULE:c().

c() ->
  original.
```

Note how the module-local call to `c/0` in `a/0` stays unchanged even though the
expectation changes the externally visible behaviour of `c/0`:

```
3> meck:new(test, [passthrough]).
ok
4> meck:expect(test,c,0,changed).
ok
5> test:a().
original
6> test:b().
changed
6> test:c().
changed
```

Contribute
----------

Patches are greatly appreciated! For a much nicer history, please [write good
commit messages][commit_messages]. Use a branch name prefixed by `feature/`
(e.g. `feature/my_example_branch`) for easier integration when developing new
features or fixes for meck.

Should you find yourself using Meck and have issues, comments or feedback please
[create an issue here on GitHub][issues].

Meck has been greatly improved by [many contributors]
(https://github.com/basho/meck/graphs/contributors)!


<!-- Badges -->
[erlang version badge]: https://img.shields.io/badge/erlang-R16B02.basho10%20to%2020.rc0-blue.svg?style=flat-square
[build tool]: https://img.shields.io/badge/build%20tool-rebar3-orange.svg?style=flat-square

<!-- Links -->
[release_notes_0.8]: https://github.com/eproxus/meck/wiki/0.8-Release-Notes
[hamcrest]: https://github.com/basho/hamcrest-erlang
[rebar_2]: https://github.com/rebar/rebar
[rebar_3]: https://github.com/erlang/rebar3
[issues]: http://github.com/basho/meck/issues
[commit_messages]: http://chris.beams.io/posts/git-commit/
