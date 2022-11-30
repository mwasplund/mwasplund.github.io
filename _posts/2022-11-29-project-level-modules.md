---
layout: default
title: "Project Level Modules"
tags: build c++ modules soup
category: blog
---

## Project Level Modules

A while back I wrote a [blog post](https://mwasplund.github.io/blog/2022/03/14/modules-in-our-builds) outlining the three ways in which Modules can be integrated into C++ builds; through dynamic expansion during build evaluation, by utilizing a preprocessing phase or by explicitly declaring the Interface dependencies. I believe that over time we will come to adopt all three variants to fit their own unique scenarios. Personally, I am exploring the possibility of creating a build system that has complete knowledge of project dependencies, whereby an implicit dependency graph exists that allows for using Modules as inter-library references. Existing package dependencies will directly map to the high-level Module Interface Graph by assigning each package a unique public Module Interface that corresponds to a single static or dynamic library.

To that end, I have been working on a [build system](https://github.com/SoupBuild/Soup) that contains all the required components to build and share self-contained Source Packages. The Build System is finally at a point where I believe the major components are in place and others can begin evaluating it for its viability as a unified C++ ecosystem united through Modules. To help illustrate how this new system will work, I walk through the process of authoring and consuming a shared package that exposes only a single Module Interface using concrete examples that map to a real-world scenario.

### Library Author

Let us consider the popular, although recently archived, [json11](https://github.com/dropbox/json11) project. This project is a great test case since it is relatively simple but is not a header only library, and everyone needs to parse a Json file or two at some point in their career. The library itself consists of a single header file and a single source file which take very little effort to convert into a Public Module Interface.

As a Library Author it is entirely up to personal preference whether to continue to utilize header-based declarations within the library or entirely migrate to a design that relies on Module Partition Units to build up the core functionality. To keep this discussion as simple as possible, I will perform the minimal number of changes to convert the existing project over to export a single Module Interface. This is accomplished by:
1. Adding a public named module declaration to the single translation unit **json11.cpp**.
	```c++
	export module json11;
	```
1. Remove external standard library includes from the public header and move them into the global module purview so they do not get included in the module and inadvertently assigned Module Linkage.
1. Add an `export` modifier to the core Json class.
	```c++
	export class Json final {
	}; 
	```
1. The final step is to hook up a build system that understands how to interact with Modules. In this case we create a Soup Recipe to declare the Package. The Recipe has a single Unique Package Name, a language identifier and build logic version, a version for the package itself and a declaration to identify the single translation unit is a Public Module Interface.
	```sml
	Name: "json11"
	Language: "C++|0.4"
	Version: "1.1.0"
	Interface: "json11.cpp"
	```

For a full changeset, take a look at the comparison of my [forked branch](https://github.com/dropbox/json11/compare/master...mwasplund:json11:master). Note that extra care was taken to ensure the code can compile continue to function using legacy header includes as well as a module interface.
Once the project is built as a Soup Package with no local dependencies it can be [published](https://github.com/SoupBuild/Soup/blob/main/Docs/CLI/Publish.md) to the public feed. This will generate an archive of the source files and Recipe definition which is all that is required to build the project on any system that has Soup installed.
```console
soup publish
```

### Library Consumer

Changing hats, we are now the proud owner of a client application that needs to read in some Json data from file. We create a second Soup Package in much the same way as the Json11 Package. In this case the Type is changed to be an Executable and we are using plain old Translation Unit to define our main method.
```sml
Name: "ParseJsonFile"
Language: "C++|0.4"
Version: "1.0.0"
Type: "Executable"
Source: [
  "Main.cpp"
]
```
The json11 package can now be consumed by including it in the set of dependencies from the Recipe. This can be accomplished by either manually editing the Recipe declaration to include the Runtime Dependency directly. Otherwise the Soup CLI includes a handy [install](https://github.com/SoupBuild/Soup/blob/main/Docs/CLI/Install.md) command that will discover the latest version in the registry.
```console
soup install json11
```
Once registered in the Recipe declaration, the package is downloaded from the public registry and unpacked on the local machine. 
```sml
Dependencies: {
  Runtime: [
    "json11@1.1.0"
  ]
}
```
The runtime dependency instructs Soup to build the Json11 project and automatically inject the correct compiled module interface references to allow us to import the Json11 Public Interface. All that is left is to import the Module and begin parsing json content.
```c++
#include <fstream>
#include <iostream>
#include <streambuf>
#include <string>

import json11;

int main()
{
  // Read in the contents of the json file
  auto jsonFile = std::ifstream("./Message.json");
  auto jsonContent = std::string(
    std::istreambuf_iterator<char>(jsonFile),
    std::istreambuf_iterator<char>());

  // Parse the json
  std::string errorMessage;
  auto json = json11::Json::parse(jsonContent, errorMessage);

  // Print the single property value
  std::cout << "Message: " << json["message"].string_value() << std::endl;

  return 0;
}
```
A full working example can be found in the [Soup Build Samples](https://github.com/SoupBuild/Soup/tree/main/Samples/Cpp/ParseJsonFile).

### Summary

This post aims to demonstrate the simplest real-world example of consuming an external library. Soup Build allows for the ability to map package dependencies to interface dependencies to create a package management solution that alleviates many of the pain points of sharing C++ libraries today. I am happy to invite others to begin [testing out](https://github.com/SoupBuild/Soup/blob/main/Docs/Getting-Started.md) the system and provide feedback.

> Note: While Soup Build's core [Architecture](https://github.com/SoupBuild/Soup/blob/main/Docs/Architecture.md) is in place, the [Cpp Build Extension](https://github.com/SoupBuild/SoupCpp) is a bare bones implementation that exposes only the smallest set of controls to get up and running.

For a more fleshed out example of a complex system being built, please explore the [Soup Build CLI](https://github.com/SoupBuild/Soup/tree/main/Source/Client/CLI) which heavily dogfoods itself as its own build system.
