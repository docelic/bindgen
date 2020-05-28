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

This documents describes configuration options that are available in
.yml files and used as the project's config from which the bindings
are generated.

Some configuration values are "templated", meaning that they are
implicitly of type String and can contain percent signs ("%") in
their content. All occurences of the percent-sign ("%") will be
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

## module (required)

Defines the `module X` into which *all* code will be put.

```
module: MyStuff
```

## cookbook

Defines how conversions in C/C++ shall happen. Use `boehmgc-cpp` for C++,
or `boehmgc-c` for pure C. Don't worry too much about this setting at first.

```
cookbook: boehmgc-cpp # Default!
```
