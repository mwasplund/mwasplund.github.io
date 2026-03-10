---
layout: default
title: "Dynamic Modules"
tags: build c++ modules soup
category: blog
---

## Dynamic Modules

This is the third installment of my series [blog posts](https://mwasplund.github.io/blog/2022/03/14/modules-in-our-builds) outlining the three ways in which Modules can be integrated into C++ builds. For the first time in C++ history, Modules introduce a dependency hierarchy and implicit order to our builds. The order can be determined through dynamic expansion during build evaluation, by explicitly declaring the interface dependencies, or by running a preprocessing phase to dynamically detect dependencies. In the [second post](https://mwasplund.github.io/blog/2022/11/29/project-level-modules) I outlined how we can leverage the structure already present in the build system itself to generate a dependency graph between modules at the library level. In this post, I dive into the internal process of scanning translation units to dynamically discover the dependencies between named modules and their internal partitions.

### Module Declaration

There are many aspects of the C++ Module feature that make Modules difficult to integrate within our build systems. The most glaring challenge is the new requirement for order dependent builds which was not necessary in the past. However, the standards committee did provide us with a way to work around these issues. The new module declaration syntax was purposefully written to make it easy to parse the beginning of a translation unit to determine if a given unit is a module unit. If it is a module unit, we can then extract the required name and full set of imported Modules. This well known, and easy to parse grammar, allows us to write a simple scanner tool to process the least amount of tokens at the start of the file to determine the complete set of input dependencies and output module name from a module unit. If we leverage a build system that is broken into a two phased approach, we can preprocess all translation units to dynamically build up the full dependency graph between named modules, their internal partition units, and implementation units. Once parsed it is a trivial problem to ensure all translation units are build in the correct order.

### Module Parser

The module parser is very simple and has a lexicon made of only 7 token types (Identifier, Period, Colon, Semicolon, Module, Import, Export). Using this simple set of tokens we can write a formal grammar that consists of a subset of the full C++ language which can strictly process the `module declaration` at the beginning of the file.

#### Module Declaration
```
export? module module-name module-partition? attr? ;
```

In module units, all import declarations must be grouped after the module declaration and before all other declarations.

#### Import Declaration
```
export? import module-name attr? ;
```

This grammar can generate a lexer that will scan until it sees an invalid token or invalid syntax, at which point it can assume it is beyond the `module declaration.` The reason this works is because we only care about discovering the module export and import set. We can safely assume that a subsequent compiler invocation will validate the remainder of the translation unit.

### Two Phase Build

With the simple parser in hand, we can then integrate it into our build process. We do this by breaking our builds into two phases: a preprocessing phase and a build phase. During the initial preprocessing phase, we can feed the full set of translation units into the light weight module parser which will generate the extra metadata required to build up the dependency graph. It will identify if the current translation unit is a module unit. It can then identify interface units along with the name and optional partition. Once the module declaration is fully parsed, the import declarations will also be parsed to generate an full set of upstream dependencies.

With a now complete dependency graph available to the build system, we can easily transform these references into implicit compilation dependencies. It is now a simple task for the build system to ensure that all upstream dependencies for a given translation unit have completed before we begin compilation of that unit. 

### Summary

With a proper build system setup to handle preprocessing translation units, we can build up a dependency graph for our builds to ensure build order is correct. We can also dynamically build complex interdependencies without requiring extra build structuring by engineers and make module adoption effortless.

In this example I use partition units to build up a named module; however, the same dependency graph can be built up between multiple named modules. If you would like to see this in action, I have a [sample](https://github.com/soup-build/soup/blob/main/docs/samples/cpp/module-partititons.md) working in the [Soup Build](https://www.soupbuild.com) system.