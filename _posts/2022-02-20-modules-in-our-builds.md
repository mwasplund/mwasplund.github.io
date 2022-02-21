---
layout: default
title: "Modules In Our Builds"
categories: Build Modules Soup C++
---

Now that we are seeing progress in supporting C++ Modules we can begin to think in ernest about how to take full advantage of Modules in our build systems. This discussion is important to begin early to help influence the implementations being adopted by the compiler vendors. 

## Overview of Modules
I will go into some detail about modules, but for a great introduction to the nitty details of how modules are constructed check out [vector-of-bool's blog](https://vector-of-bool.github.io/2019/03/10/modules-1.html).

### Module Units
Translation units that contain a module declaration are a new concept with C++20 and indicate to the compiler that you are producing a module. It is worth noting that a standard Trnaslation unit that does not declare a module may still consume modules through an import declaration. A module can be divided into two categories: Interface Units, and Implementation Units. The key difference being the presence of an ```export``` keyword attached to the module declaration in an Interface Unit. Module units can also be a special type of Interface or Implementation unit if they contain a Partition definition. A partition module simply has a special ':' break in the name and requires there be a single Primary Interface Unit that brings together all the partitions with a single name.

## Discover or Define?
The question then becomes, do we let our build system discover the implicit dependencies for all of these modules inter-dependencies or do we explicitly tell the build system the entire dependency graph? There are pros and cons to each approach.

### Explicit Declare
Explicitly defining a dependency graph is straight forward and can support any combination of imports and exports with the extra cost of maintenance. 

We can also take advantage of the existing dependency graph between projects to automatically incorporate a module dependency too. This is where we can easily mirror a library boundary to have a single interface unit export the symbols that are public to the library. From here the build system can incorporate project references to automatically reference the module interface units for the upstream dependencies when building and link against the library symbols as is done today.

### Auto Discover
Auto discovery can lead to a more flued developer experience where a group of translation units are compiled as a collection where a preprocessing phase parses the module unit declarations and imports to determine the dynamic dependency graph and builds the translation units in the required order with the required references or the compiler can implement a callback design where it asks the build system to imported modules as they are discovered.




