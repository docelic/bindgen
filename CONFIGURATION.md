# Bindgen Configuration Reference

# Table of Contents

<!--ts-->
   * [Introduction](#introduction)
   * [YAML syntax](#yaml-syntax)
      * [Conditions](#conditions)
      * [Variables](#variables)
         * [Examples](#examples)
      * [Templated Strings](#templated-strings)
      * [Dependencies](#dependencies)
         * [Errors](#errors)
   * [Configuration options](#configuration-options)
      * [Module (required)](#module-required)
      * [Cookbook](#cookbook)
      * [Library](#library)
      * [Processors](#processors)
      * [Generators](#generators)
      * [Find_paths](#find_paths)

<!-- Added by: docelic, at: Thu 28 May 2020 11:26:04 PM CEST -->

<!--te-->

# Introduction

This document describes the syntax and configuration options available in
bindgen's .yml files. They are used as the config from which the bindings
are generated.

# YAML syntax

Bindgen's YAML configuration files support conditions, variables, templated
strings, and loading other YAML files. This is implemented using a notation
that is still valid YAML.

**Note**: Conditionals and dependencies are *only* supported in
*mappings* (`Hash` in Crystal).  Any such syntax encountered in something
other than a *mapping* will not trigger any special behaviour.

## Conditions

YAML documents can define conditional parts in *mappings* by having a
conditional key, with *mapping* value.  If the condition matches, the
*mapping* value will be transparently embedded.  If it does not match, the
value will be transparently skipped.

Condition keys look like `if_X` or `elsif_X` or `else`.  `X` is the
condition, and it looks like `Y_is_Z` or `Y_match_Z`.  You can also use
(one or more) spaces (` `) instead of exactly one underscore (`_`) to
separate the words, although the underscore notation is more common.

* `Y_is_Z` is true if the variable Y equals Z case-sensitively
* `Y_isnt_Z` is true if the variable Y doesn't equal Z case-sensitively
* `Y_match_Z` is true if the variable Y is matched by the regular expression
in `Z`.  The regular expression is created case-sensitively
* `Y_newer_or_Z` is true when variable Y is newer or equal (>=) to Z.
Both variables are treated as versions
* `Y_older_or_Z` is true when variable Y is older or equal (=<) to Z.
Both variables are treated as versions

A condition block is opened by the first `if`.  Later condition keys can
use `elsif` or `else` (or `if` to open a *new* condition block).

**Note**: `elsif` or `else` without an `if` will raise an exception.

Their behaviour is like in Crystal: `if` starts a condition block, `elsif`
starts an alternative condition block, and `else` is used if none of `if` or
`elsif` matched.  It's possible to mix condition key-values with normal
key-values.

**Note**: Conditions can be used in every *mapping*, even in *mappings* of
a conditional.  Each *mapping* acts as its own scope.

## Variables

Variables are set by the user of the class (probably through
`ConfigReader.from_yaml`).  All variable values are strings.

Variable names are **case-sensitive**.  A missing variable will be treated
as having an empty value (`""`).

### Examples

```yaml
foo: # A normal mapping
  bar: 1

# A condition: Matches if `platform` equals "arm".
if_platform_is_arm: # In Crystal: `if platform == "arm"`
  company: ARM et al

# You can mix in values between conditionals.  It won't "break" following
# elsif or else blocks.
not_a_condition: Hello

# An elsif: Matches if 1) the previous conditions didn't match
# 2) its own condition matches.
elsif_platform_match_x86: # In Crystal: `elsif platform =~ /x86/`
  company: Many different

elsif_os_is_windows:
  build: mingw

# An else: Matches if all previous conditions didn't match.
else:
  company: No idea

# At any time, you can start a new if sequence.
"if today is friday": # You can use spaces instead of underscores too
  hooray: true
```

## Templated Strings

Some configuration values are "templated", meaning that they are
implicitly of type String and can contain "%" and "{...}" in
their content.

All occurrences of the percent-sign ("%") will be
replaced by an assumed, computed value relevant for the option.

All occurrences of curly braces ("{...}") will be expanded to contents of
environment variables. `{CC}` would be expanded to the value of `ENV["CC"]`.
It is
also possible to provide a fallback value that will be used if the given
environment variable doesn't exist. `{CC|gcc}` would expand to `ENV["CC"]`,
or if it is not set, to `gcc`.  You can also put a percent-sign in there
to prefer the environment variable before the templated value:
`{LIBRARY_PATH|%}` will expand to `ENV["LIBRARY_PATH"]`, falling back to
the replacement value if unset.

## Dependencies

To modularize the configuration, you can load/merge external YAML
files from within your configuration.

This is triggered by using a key named `<<`, and writing the file name as
value: `<<: my_dependency.yml`.  The file-extension can also be omitted:
`<<: my_dependency` in which case an `.yml` extension is assumed.

The dependency path is relative to the currently processed YAML file.

You can also require multiple dependencies into the same *mapping*:

```yaml
types:
  Something: true # You can mix dependencies with normal fields.
  <<: simple_types.yml
  <<: complex_types.yml
  <<: ignores.yml
```

The dependency will be embedded into the open *mapping*: It's transparent
to the client code.

It's perfectly possible to mix conditionals with dependencies:

```yaml
if_os_is_windows:
  <<: windows-specific.yml
```

### Errors

An exception will be raised if any of the following occur:

* The maximum dependency depth of `10` (`MAX_DEPTH`) is exceeded.
* The dependency name contains a dot: `../foo.yml` won't work.
* The dependency name is absolute: `/foo/bar.yml` won't work.

# Configuration options

The options in this list are sorted by typical order of use.

## Module (required)

Defines the `module X` into which *all* code will be put.

```
module: MyStuff
```

## Cookbook

Defines how conversions in C/C++ shall happen. Use `boehmgc-cpp` for C++,
or `boehmgc-c` for pure C. Don't worry too much about this setting at first.

```
cookbook: boehmgc-cpp # Default!
```

## Library

Defines the `ld_flags` value for the `@[Link]` directive of the generated `lib`.
`%` will be replaced by the path to the base-directory of your project, relative
to the path of the generated `.cr` file.

```
library: "%/ext/binding.a"
```

## Processors

Processors pipeline.  See `README.md` for details on each.
Defaults to the following:

```
processors:
  # Graph-refining processors:
  - default_constructor # Create default constructors where possible
  - function_class      # Turn OOP-y C APIs into real classes
  - inheritance         # Mirror inheritance hierarchy from C++
  - copy_structs        # Copy structures as marked
  - macros              # Support for macro mapping
  - functions           # Add non-class functions
  - filter_methods      # Throw out filtered methods
  - extern_c            # Directly bind to pure C functions
  - instantiate_containers # Actually instantiate containers
  - enums               # Add enums
  # Preliminary generation processors:
  - crystal_wrapper     # Create Crystal wrappers
  - virtual_override    # Allow overriding C++ virtual methods
  - cpp_wrapper         # Create C++ <-> C wrappers
  - crystal_binding     # Create `lib` bindings for the C wrapper
  - sanity_check        # Shows issues, if any
```

## Generators

Generators' configuration. These write the actual output to disk.

```
generators:

  cpp: # C++ generator
    output: ext/my_bindings.cpp # Output file path  (Mandatory)
    preamble: |-        # Output file preamble, can be multiline  (Optional)
      #include "bindgen_helper.hpp"
    build: make         # Command to run after the generator.  (Optional!)
                        # Will be executed as-written in the output directory.
                        # If the command signals failure, bindgen will halt too.
                        # You can compile a small project directly, e.g. using value:
                        # build: "{CXX|c++} -std=c++11 -c -o binding.o -lMyLib my_bindings.cpp"

  # Crystal generator.  Configuration style is exactly the same as for cpp above.
  crystal:
    output: src/my_lib/binding.cr # You'll most likely only need the `output` option.
```

## Find_paths

This lets you find paths to your dependencies.  If you don't need it, just
omit it.

The key is the environment variable that a match will be put into.  These are
exposed to build-steps, so you can access them in e.g. a Makefile.  You can
also access these in all templated strings, just as if you set it right
away.

If an environment variable of this name is already set, and is not empty,
the match will *not* run!  This allows your users to supply a custom path
in non-standard installations.

The searches are run in the order they are defined.  Thus, later searches
have access to the result of previous searches by accessing the environment
variables.

Attention when using conditionals: The conditionals are evaluated first,
then the paths are found.  This means that you can't check for a found
path in a conditional!

All regular expressions in this section match case-sensitive and are
in multi-line mode (`^` will always match at the beginning of a line).
