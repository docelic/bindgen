# Bindgen

Standalone C, C++, and/or Qt binding and wrapper generator.

![Logo](https://raw.githubusercontent.com/Papierkorb/bindgen/master/images/logo.png) [![Build Status](https://travis-ci.org/Papierkorb/bindgen.svg?branch=master)](https://travis-ci.org/Papierkorb/bindgen)

## Installation

Add the dependency to `shard.yml`:

```yaml
dependencies:
  bindgen:
    github: Papierkorb/bindgen
    version: ~> 0.7.0
```

# Table of Contents

<!--ts-->
   * [How To](#how-to)
   * [Projects using bindgen](#projects-using-bindgen)
   * [Mapping behaviour](#mapping-behaviour)
   * [Features](#features)
   * [Architecture of bindgen](#architecture-of-bindgen)
      * [The Graph](#the-graph)
      * [Parser step](#parser-step)
      * [Graph::Builder step](#graphbuilder-step)
      * [Processor step](#processor-step)
      * [Generator step](#generator-step)
   * [Processors](#processors)
      * [AutoContainerInstantiation](#autocontainerinstantiation)
      * [CopyStructs](#copystructs)
      * [CppWrapper](#cppwrapper)
      * [CrystalBinding](#crystalbinding)
      * [CrystalWrapper](#crystalwrapper)
      * [DefaultConstructor](#defaultconstructor)
      * [DumpGraph](#dumpgraph)
      * [Enums](#enums)
      * [ExternC](#externc)
      * [FilterMethods](#filtermethods)
      * [Functions](#functions)
      * [FunctionClass](#functionclass)
      * [Inheritance](#inheritance)
      * [InstantiateContainers](#instantiatecontainers)
      * [Macros](#macros)
      * [Qt](#qt)
      * [SanityCheck](#sanitycheck)
      * [VirtualOverride](#virtualoverride)
   * [Advanced configuration features](#advanced-configuration-features)
      * [Conditions](#conditions)
         * [Variables](#variables)
         * [Examples](#examples)
      * [Dependencies](#dependencies)
         * [Errors](#errors)
   * [Platform support](#platform-support)
   * [Contributing](#contributing)
      * [Contributors](#contributors)
   * [License](#license)

<!-- Added by: docelic, at: Thu 28 May 2020 09:55:45 PM CEST -->

<!--te-->

# How To

1. Add bindgen to your `shard.yml` as instructed above under Installation and run `shards`
2. Copy `lib/bindgen/assets/bindgen_helper.hpp` into your `ext/`
3. Copy `lib/bindgen/TEMPLATE.yml` into `your_template.yml` and customize it for the library you want to bind to
4. Run `lib/bindgen/tool.sh your_template.yml`. This will generate the bindings, by default in the `ext/` subdirectory
5. Develop your Crystal application as usual

See `TEMPLATE.yml` for the complete configuration and customization documentation.
(This documentation will soon be ported to a Markdown document as well.)

**Note**: If you ship the output produced by bindgen along with your application,
then `bindgen` will not be not required to compile it. In that case, you can move
its entry in `shard.yml` from `dependencies` to `development_dependencies`.

# Projects using bindgen

You can use the following projects' .yml files as a source of ideas or syntax for
your own bindings:

* [Qt5 Bindings](https://github.com/Papierkorb/qt5.cr)

*Have you created and published a usable binding with bindgen? Want to see it here? Send a PR!*

# Mapping behaviour

The following rules are automatically applied to all bindings:

* Method names get underscored: `addWidget() -> #add_widget`
  * Setter methods are rewritten: `setWindowTitle() -> #window_title=`
  * Getter methods are rewritten: `getWindowTitle() -> #window_title`
  * Bool getters are rewritten: `getAwesome() -> #awesome?`
  * `is` getters are rewritten: `isEmpty() -> #empty?`
  * `has` getters are rewritten: `hasSpace() -> #has_space?`
* On signal methods (For Qt signals):
  * Keep their name for the `emit` version: `pressed() -> #pressed`
  * Get an `on_` prefix for the connect version: `#on_pressed do .. end`
* Enum fields get title-cased if not already: `color0 -> Color0`

# Features

| Feature                                          | Support |
|--------------------------------------------------|---------|
| Automatic Crystal binding generation             | **YES** |
| Automatic Crystal wrapper generation             | **YES** |
| Mapping C++ classes                              |         |
|  +- Member methods                               | **YES** |
|  +- Static methods                               | **YES** |
|  +- Constructors                                 | **YES** |
|  +- Overloaded operators                         |   TBD   |
|  +- Conversion functions                         |   TBD   |
| Mapping C/C++ global functions                   |         |
|  +- Mapping global functions                     | **YES** |
|  +- Wrapping as Crystal class                    | **YES** |
| Overloaded methods (Also default arguments)      | **YES** |
| Copying default argument values                  |         |
|  +- Integer, float, boolean types                | **YES** |
|  +- String                                       | **YES** |
| Enumerations                                     | **YES** |
| Copying structures                               | **YES** |
| Custom type conversions between C/++ and Crystal | **YES** |
| Automatic type wrapping and conversion           | **YES** |
| Integration with Crystals GC                     | **YES** |
| C++ Template instantiation for containers types  | **YES** |
| Virtual methods                                  | **YES** |
| Override virtual methods from Crystal            | **YES** |
| Abstract classes                                 | **YES** |
| Multiple inheritance wrapping                    | **YES** |
| Qt integration                                   |         |
|  +- QObject signals                              | **YES** |
|  +- QFlags types                                 | **YES** |
|  +- QMetaObject generation (mimic `moc`)         |   TBD   |
| `#define` macro support                          |         |
|  +- Mapping as enumeration                       | **YES** |
|  +- Mapping as constant (Including strings)      | **YES** |
| Copying in-source docs                           |   TBD   |
| Platform specific type binding rules             | **YES** |
| Portable path finding for headers, libs, etc.    | **YES** |

# Architecture of bindgen

Bindgen employs a pipeline-inspired code architecture, which is strikingly
similar to what most compilers use.

The code flow is basically `Parser::Runner` to `Graph::Builder` to
`Processor::Runner` to `Generator::Runner`.

![Architecture flow diagram](https://raw.githubusercontent.com/Papierkorb/bindgen/master/images/architecture.png)

## The Graph

An important data structure used throughout the program is *the graph*.
Code-wise, it's represented by `Graph::Node` (And its sub-classes).  The nodes
can contain child nodes, making it a hierarchical structure.

This allows to represent (almost) arbitrary structures as defined by the user
configuration.

Say, we're wrapping `GreetLib`.  As any library, it comes with a bunch of
classes (`Greeter` and `Listener`), enums  (`Greetings`, `Type`) and other stuff
like constants (`PORT`).  The configuration file could look like this:

```yaml
module: GreetLib
classes: # We copy the structure of classes
  Greeter: Greeter
  Listener: Listener
enums: # But map the enums differently
  Type: Greeter::Type
  Greeter::Greetings: Greetings
```

Which will generate a graph looking like this:

![Graph example](https://raw.githubusercontent.com/Papierkorb/bindgen/master/images/graph.png)

**Note**: The concept is really similar to ASTs used by compilers.

## Parser step

The beginning of the actual execution pipeline.  Calls out to the clang-based parser
tool to read the C/C++ source code and write a JSON-formatted "database" onto
standard output.  This is directly caught by `bindgen` and subsequently parsed
as `Parser::Document`.

## Graph::Builder step

The second step takes the `Parser::Document` and transforms it into a
`Graph::Namespace`.  This step is where the user configuration mapping is used.

## Processor step

The third step runs all configured processors in order.  These work with the
`Graph` and mostly add methods and `Call`s so they can be bound later.  But
they're allowed to do whatever they want really, which makes it a good place
to add more complex rewriting rules if desired.

Processors are responsible for many core features of bindgen.  The `TEMPLATE.yml`
has an already set-up pipeline.

## Generator step

The final step now takes the finalized graph and writes the result into an
output of one or more files.  Generators do *not* change the graph in any way,
and also don't build anything on their own.  They only write to output.

# Processors

The processor pipeline can be configured through the `processors:` array.  Its
elements are run in the order they're defined, starting at the first element.

**Note**: Don't worry: The `TEMPLATE.yml` file already comes with the
recommended pipeline pre-configured.

There are three kinds of processors:
1. *Refining* ones modify the graph in some way, without a dependency to a later
   generator.
2. *Generation* processors add data to the graph so that the generators
   run later have all data they need to work.
3. *Information* processors don't modify the graph, but do checks or print data
   onto the screen for debugging purposes.

The order in the configured pipeline is to have *Refining* processors first,
*Generation* processors second. *Information* processors can be run at any time.

The following processors are available, in alphabetical order:

## `AutoContainerInstantiation`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: `InstantiateContainers`

When encountering a known container class on an instantiation that is not
registered yet, registers it.

Container classes still need to be declared in the configuration, but don't
require an explicit `instantiations` attribute anymore:

```yaml
containers: # At the top-level of the config
  - class: QList # Set the class name
    type: Sequential # And its type
    # instantiations: # Can be added, but doesn't need to be.
```

## `CopyStructs`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: No specific dependency

Copies structures of those types, that have `copy_structure: true` set in the
configuration.  A wrapper class of a `copy_structure` type will host the
structure directly (instead of a pointer) to it.

## `CppWrapper`

* **Kind**: Generation
* **Run after**: *Refining* processors
* **Run before**: `CrystalBinding`

Generates the C++ wrapper method `Call`s.

## `CrystalBinding`

* **Kind**: Generation
* **Run after**: `CppWrapper`, `VirtualOverride` and `CrystalWrapper`
* **Run before**: No specific dependency

Generates the `lib Binding` `fun`s.

## `CrystalWrapper`

* **Kind**: Generation
* **Run after**: *Refining* processors
* **Run before**:  `CrystalBinding` and `VirtualOverride`

Generates the Crystal methods in the wrapper classes.

## `DefaultConstructor`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: No specific dependency

Clang doesn't expose default constructors methods for implicit default
constructors.  This processor finds these cases and adds an explicit constructor.

## `DumpGraph`

* **Kind**: Information
* **Run after**: Any time
* **Run before**: Any time

Debugging processor dumping the current graph onto `STDERR`.

## `Enums`

* **Kind**: Refining
* **Run after**: `FunctionClass`
* **Run before**: No specific dependency

Adds the copied enums to the graph.  Should be run after other processors adding
classes, so that enums can be added into classes.

## `ExternC`

* **Kind**: Refining
* **Run after**: `Functions` and `FunctionClass`
* **Run before**: No specific dependency

Checks if a method require a C/C++ wrapper.  If not, marks the method to
bind directly to the target method instead of writing a "trampoline"
wrapper in C++.

**Note**: This processor is *required* for variadic functions to work.  A
variadic function looks like this: `void func(int c, ...);`

A method can be bound directly if all of these are true:

1. It uses the C ABI (`extern "C"`)
2. No argument uses a `to_cpp` converter
3. The return type doesn't use a `from_cpp` converter

**Note**: If all methods can be bound to directly, you can remove the `cpp`
generator completely from your configuration.

## `FilterMethods`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: No specific dependency

Removes all methods using an argument, or returning something, which is
configured as `ignore: true`.  Also removes methods that show up in the
`ignore_methods:` list.

This processor can be run at any time in theory, but should be run as first part
of the pipeline.

## `Functions`

* **Kind**: Refining
* **Run after**: `FunctionClass` and `ExternC`
* **Run before**: No specific dependency

Maps C functions, configured through the `functions:` map in the configuration.

## `FunctionClass`

* **Kind**: Refining
* **Run after**: `ExternC`
* **Run before**: `Inheritance` and `Functions`

Generates wrapper classes from OOP-like C APIs, using guidance from the user
through configuration in the `functions:` map.

## `Inheritance`

* **Kind**: Refining
* **Run after**: `FunctionClass`
* **Run before**: `FilterMethods` and `VirtualOverride`

Implements Crystal wrapper inheritance and adds `#as_X` conversion methods.
Also handles abstract classes in that it adds an `Impl` class, so code can
return instances to the (otherwise) abstract class.

## `InstantiateContainers`

* **Kind**: Refining
* **Run after**: `AutoContainerInstantiation` if used
* **Run before**: No specific dependency

Adds the container instantiation classes and wrappers.

## `Macros`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: No specific dependency

Maps `#define` macros into the graph.  The mapping is configured by the user in
the `macros:` list.  Only value-macros ("object-like macros") are supported,
function-like macros are silently skipped.

```c++
// Okay:
#define SOME_INT 1
#define SOME_STRING "Hello"
#define SOME_BOOL true

// Not mapped:
#define SOME_FUNCTION(x) (x + 1)
```

## `Qt`

* **Kind**: Refining
* **Run after**: No specific dependency
* **Run before**: No specific dependency

Adds Qt specific behaviour:

1. Removes the `qt_check_for_QGADGET_macro` fake method.
2. Provides `#on_SIGNAL` signal connection method.

```crystal
btn = Qt::PushButton.new
btn.on_clicked do |checked| # Generated by this processor
  puts "Checked: #{checked}"
end
```

## `SanityCheck`

* **Kind**: Information
* **Run after**: Any time, as very last pipeline element is ideal.
* **Run before**: Any time

Does sanity checks on the graph, focusing on Crystal bindings and wrappers.

Checks are as follows:
* Name of enums, libs, structures, classes, modules and aliases are valid
* Name of constants are valid
* Name of methods are valid
* Enumerations have at least one constant
* Flag-enumerations don't have `All` nor `None` constants
* Method arguments and result types are reachable
* Variadic methods are directly bound
* Alias targets are reachable
* Class base-classes are reachable

## `VirtualOverride`

* **Kind**: Refining, but ran after generation processors!
* **Run after**: `CrystalWrapper`!
* **Run before**: `CrystalBinding` and `CppWrapper`

Adds C++ and Crystal wrapper code to allow overriding C++ virtual methods from
within Crystal.  Requires the `Inheritance` processor.

**Important Note**: Make sure to run this processor after `CrystalWrapper` but
before `CrystalBinding`.

It needs to modify the `#initialize` methods, and generate `lib` structures,
bindings, and C++ code too.

This is the recommended processor order:

```yaml
processors:
  # ...
  - crystal_wrapper
  - virtual_override
  - cpp_wrapper
  - crystal_binding
```

After this, usage is the same as with any method:

```crystal
class MyAdder < VirtualCalculator
  # In C++: virtual int calculate(int a, int b);
  # In Crystal:
  def calculate(a, b)
    a + b
  end
end
```

# Platform support

<!-- Table is sorted from A-Z ascending, versions descending. -->

| Arch    | System            | CI          | Clang version  |
| ------- | ----------------- | ----------- | -------------- |
| x86_64  | ArchLinux         | Travis      | *Rolling*      |
| x86_64  | Debian 9          | Travis      | 6.0, 7.0       |
| x86_64  | Debian 7          | Travis      | 4.0, 5.0       |
| x86_64  | Ubuntu 17.04      | *None*      | 5.0            |
| x86_64  | Ubuntu 16.04      | Travis      | 4.0, 5.0       |
|         | Other systems     | Help wanted | ?              |

You require the LLVM and Clang development libraries and headers.  If you don't
have them already installed, bindgen will tell you. These packages are usually
named after the following pattern on Debian-based systems:
`clang-7 libclang-7-dev llvm-7 llvm-7-dev`.


# Contributing

1. Open a new issue on the project to discuss what you're going to do and possibly receive comments
2. Read the `STYLEGUIDE.md` for some tips.
3. Then do the rest, PR and all.  You know the drill :)

## Contributors

- [Papierkorb](https://github.com/Papierkorb) Stefan Merettig - creator
- [docelic](https://github.com/docelic) Davor Ocelic
- [kalinon](https://github.com/kalinon) Holden Omans
- [ZaWertun](https://github.com/ZaWertun) Yaroslav Sidlovsky

# License

This project (`bindgen`) and all of its sources, except those otherwise noted,
all fall under the `GPLv3` license.  You can find a copy of its complete license
text in the `LICENSE` file.

The configuration used to generate code, and all code generated by this project,
fall under the full copyright of the user of `bindgen`.  `bindgen` does not
claim any copyright, legal or otherwise, on your work.  Established projects
should define a license they want to use for the generated code and
configuration.
