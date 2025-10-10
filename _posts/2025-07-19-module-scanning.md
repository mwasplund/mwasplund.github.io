---
layout: default
title: "Dynamic Modules"
tags: build c++ modules soup
category: blog
---

## Dynamic Modules

This is the third installment of my original [blog post](https://mwasplund.github.io/blog/2022/03/14/modules-in-our-builds) outlining three ways in which Modules can be integrated into C++ builds: through dynamic expansion during build evaluation, by explicitly declaring the interface dependencies, or by running a preprocessing phase to dynamically detect dependencies. In the [second post](https://mwasplund.github.io/blog/2022/11/29/project-level-modules) I outlined how we can leverage the structure already present in the build system itself to generate a dependency graph between modules at the library level. In this post I dive into the internal process of scanning translation units to dynamically discover the dependencies between named modules and their internal partitions.

### Module Declaration

There are many aspects of the C++ Module feature that make it very hard to integrate with our build systems. The most glaring being the new requirement for order dependent builds which was not true in the past. The standards committee did provide us with a way to work around these issues. The new module declaration syntax was purposefully written to make it easy to parse the beginning of a translation unit to determine if the translation is a module unit and if so the required name and full set of imported modules. This well known and easy to parse grammar allows us to write a simple scanner tool to process the least amount of tokens at the start of the file to determine the input and output edges from a module unit. If we leverage a build that is broken into two phased approach we can preprocess all translation units to dynamically build up the full dependency graph between named modules, their internal partition units and implementation units.

### Module Parser 

The module parser is very simple and has a grammar made of only 7 token types (Identifier, Period, Colon, Semicolon, Module, Import, Export). Using this simple set of tokens we can write a grammar that only knows how to parse the `module declaration` at the beginning of the file. This grammar can then generate a lexer that will scan until it sees an invalid token or invalid grammar, at which point it can assume it is beyond the `module declaration`. The reason this works it that we only care about discovering the module export/import set and assume that a subsequent compiler call will validate the remainder of the translation unit.

Module Declaration
```
export? module module-name module-partition? attr? ;
```

In module units, all import declarations must be grouped after the module declaration and before all other declarations.

Import Declaration
```
export? import module-name attr? ;
```

### Summary

With a proper build system setup to handle preprocessing translation units we can build up a dependency graph for our builds to ensure build order is correct and we can dynamically build complex interdependencies without requiring extra build structuring by engineers and will make module adoption very easy.

In this example I use partition units that build up a named module, however the same dependency graph can be built up between multiple named modules.