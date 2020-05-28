# Bindgen Configuration Reference

# Table of Contents

<!--ts-->
   * [Introduction](#introduction)
   * [Configuration options](#configuration-options)
      * [module (required)](#module-required)
      * [cookbook](#cookbook)

<!-- Added by: docelic, at: Thu 28 May 2020 10:54:48 PM CEST -->

<!--te-->

# Introduction

This document describes the configuration options available in
bindgen's .yml files. They are used as the project's config
from which the bindings are generated.

Some configuration values are "templated", meaning that they are
implicitly of type String and can contain percent signs ("%") in
their content. All occurrences of the percent-sign ("%") will be
replaced by an assumed, computed value relevant for the option.

Additionally, templated strings allow access to environment variables using
curly braces: `{CC}` would be expanded to the value of `ENV["CC"]`.  It is
also possible to provide a fallback value that will be used if the given
environment variable doesn't exist: `{CC|gcc}` would expand to `ENV["CC"]`,
or if it is not set, to `gcc`.  You can also put a percent-sign in there
to prefer the environment variable before the templated value:
`{LIBRARY_PATH|%}` will expand to `ENV["LIBRARY_PATH"]`, falling back to
the replacement value if unset.

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

    # Do you have complex dependencies?  Use a conditional!
    if_os_is_windows: # Described under "Conditions" in README.md
       build: mingw-make

  # Crystal generator.  Configuration style is exactly the same as for cpp above.
  crystal:
    output: src/my_lib/binding.cr # You'll most likely only need the `output` option.
```
