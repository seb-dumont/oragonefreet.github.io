---
layout: post
title: CMake-Compliant libraries
published: true
---

# So you made a library

C or C++? Static, shared or header-only? Maybe a combination of all that.  
Anyways your library is ready to be used by everyone and you made a super simple README.md that says :

> Here's my header folder, here's my library file. Good luck, have fun!

Ok well... not really *that* simple but you get the point.

With CMake, libraries are available through `find_package()`. This is a very trivial command and you may already have used it:

```cmake
cmake_minimum_required(VERSION 3.10.0 FATAL_ERROR)

project(MySuperGame)

find_package(OpenGL REQUIRED)

add_executable(game game.cpp)
target_link_libraries(game PRIVATE OpenGL::GL)
```

`find_package()` searches for OpenGL "somewhere" in your environment and provides you with a target named `OpenGL::GL`. Fairly easy, right?

Let's go through the looking glass and see how to make your library importable like this.

# Our example library

Let's say we make a maths library. Static, in C.  
This first part of the CMakeLists is quite trivial:

```cmake
cmake_minimum_required(VERSION 3.10.0 FATAL_ERROR)

project(Maths VERSION 1.0.0)

add_library(maths STATIC
  inc/m_areas.h
  inc/m_volumes.h
  src/m_areas.c
  src/m_volumes.c
)

target_include_directories(maths PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/inc
)
```

Everything should be ok so far.
Note that I explicitly set the version in the `project` command. This will be useful for our export later.

Before writing the full CMakeLists we need to understand how `find_package()` works.

# find_package explained

`find_package()` comes in two flavours: Module mode and Config mode.

> `find_package()` will first run in **Module** mode and go to **Config** mode if it fails. In that order.

## Module mode

I won't really go into details about Module mode because it's not the one we're interested in here.

Yet, let's summarize here like this: in Module mode, `find_package()` looks for a script named `FindXXX.cmake` in your CMake installation itself. `XXX` is the name of the module, such as [`FindOpenGL.cmake`](https://github.com/Kitware/CMake/blob/master/Modules/FindOpenGL.cmake).  
This script is good for looking for anything looking like OpenGL in your entire system.

Module mode is well suited for popular packages (like OpenGL is), and CMake provides [a lot of find modules](https://github.com/Kitware/CMake/tree/master/Modules) by default.

> Module mode is when the package doesn't know anything about CMake, but CMake knows where to find it anyways.

If the script fails finding the package, or even if the script file doesn't exist at first, `find_package()` retries in Config mode.

## Config Mode

In Config mode CMake looks for a file named `XXXConfig.cmake` (or `xxx-config.cmake`) where `XXX` is the name of the package you're searching for. For example: `OpenGLConfig.cmake`.

But it is not a "find" script. This script is actually part of the package and perfectly knows where everything is installed.

> In *module* mode, CMake searches for `FindXXX.cmake`. In *config* mode it is `XXXConfig.cmake`.

Let's say for example, our maths library is installed like this:

```
MathsLib
├── include/
│   ├── include/m_areas.h
│   └── include/m_volumes.h
├── lib/
│   └── maths.a
├── README.md
├── LICENCE
└── MathsConfig.cmake
```

This structure is enforced by our own CMake install script and thus everything is where we want it to be. Our `MathsConfig.cmake` could be as simple as:

```cmake
add_library(CO::Maths IMPORTED GLOBAL)

set_target_properties(CO::Maths PROPERTIES
  IMPORTED_LOCATION
    ${CMAKE_CURRENT_LIST_DIR}/lib/maths.a
  INTERFACE_INCLUDE_DIRECTORIES
    ${CMAKE_CURRENT_LIST_DIR}/include
)
```

If you don't know what imported targets are, this is not a big deal. Just keep in mind that imported targets are libraries that are not built by your project. It is a very convenient way to express already built binaries as a CMake compliant target, and it is exactly what we do here.

> Configuration files essentially about creating an `IMPORTED` target for us.

As you probably guessed, we have to create this file for our maths library. But don't worry CMake will do most of the job.

## `CMAKE_PREFIX_PATH`

One last this. When searching for your config file, CMake will build potential search folders starting from a root directory call the `prefix`.

This is up to the user of your library to configure this path by setting the variable `CMAKE_PREFIX_PATH`.

The search patterns for finding the config file are:

```
<prefix>/
<prefix>/(cmake|CMake)/
<prefix>/<name>*/
<prefix>/<name>*/(cmake|CMake)/
<prefix>/(lib/<arch>|lib|share)/cmake/<name>*/
<prefix>/(lib/<arch>|lib|share)/<name>*/
<prefix>/(lib/<arch>|lib|share)/<name>*/(cmake|CMake)/
```

It means that if `CMAKE_PREFIX_PATH` is `/usr`, any of these paths are search for `MathsConfig.cmake`, among others:

```
/usr/
/usr/CMake/
/usr/lib/cmake/
/usr/Maths/
/usr/MathsLibs/
/usr/Maths/CMake/
```

> `CMAKE_PREFIX_PATH` is a user defined paths. Your config file is searched from it.


# The CMakeLists step by step

CMake install step is the last one your build system does (Configure, Build, Install).  
Installing is the process of taking everything you need to package your library and put it in a root folder called the **install prefix**.

In our example above, our install prefix is `MathsLib/` :

```
MathsLib
├── include/
│   ├── include/m_areas.h
│   └── include/m_volumes.h
├── lib/
│   └── maths.a
├── README.md
├── LICENCE
└── MathsConfig.cmake
```

Actually the install prefix is an absolute path you set at configuration step with the `CMAKE_INSTALL_PREFIX` option. In your CMakeLists file you must do everything relatively to this folder.  

## Installing the library

Your library is before all a `.a` file which will probably be placed into the `lib` subfolder or your install prefix.

```cmake
install(TARGETS maths ARCHIVE DESTINATION lib)
```

This is all, technically. Here `libMaths.a`, represented by the target `Maths` will be placed into `lib`.

Note the `ARCHIVE` option here, which is because we built a static library. The `install` command supports a lot of target types. I'll list the three most common:

- `ARCHIVE` is used for static libraries, so `lib*.a` on UNIX-like systems and `*.lib` on Windows. It is also the type used for the `*.lib` files of DLLs. Remember that on Windows, dynamic libraries come into two parts, the `*.dll` and the `*.lib`.

- `RUNTIME` is for executables and `*.dll` files.

- `LIBRARY` is for shared object files on UNIX systems, so `*.so` files.

An `install(TARGETS ...)` command can specify the installation of several targets at once. That's why you can set the destination of all target types in one command:

```
install(TARGETS maths
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION bin
  RUNTIME DESTINATION bin
)
```

In our example we _know_ `Maths` is a static library so why bother putting everything? Usually when you provide a C or C++ library you don't enforce the type (`STATIC` or `SHARED`). you instead let your the guy who runs your CMake decide by setting the `BUILD_SHARED_LIBS` option. So you don't really known what your target is. The best is to write the `install`  command for both cases.

Installing the rest of the library is trivial and uses other forms of the `install` command:

```cmake
install(DIRECTORY inc/ DESTINATION include)
install(FILES LICENSE README.md DESTINATION .)
```

Ok with all that, we should have something like this upon installation:

```
MathsLib
├── include/
│   ├── include/m_areas.h
│   └── include/m_volumes.h
├── lib/
│   └── maths.a
├── README.md
└── LICENCE
```

Not bad. Let's work on the config file!

## Exporting the target

You may have guessed, there's a relation between an `IMPORTED` target and exporting a target.

Exporting a target simply consists in writing a CMake file that configures the imported target. If this script can be included by anyone, you successfully exported the target.

Previously, we wrote such a file:


```cmake
add_library(CO::Maths IMPORTED GLOBAL)

set_target_properties(CO::Maths PROPERTIES
  IMPORTED_LOCATION
    ${CMAKE_CURRENT_LIST_DIR}/lib/maths.a
  INTERFACE_INCLUDE_DIRECTORIES
    ${CMAKE_CURRENT_LIST_DIR}/include
)
```

In theory exporting your target is very simple, in practice you have to deal juggle between relative paths, target types, configurations, ... and it's not that simple.

Fortunately CMake helps here by writing the export file for us:

```cpp
install(EXPORT MathsExport
  DESTINATION lib/cmake/Maths
  NAMESPACE "CO::"
  FILE MathsConfig.cmake
)
```

The first parameters is the name of the export: `MathsExport`. There is no strict rule here, you can choose what you want.

With `DESTINATION` and `FILE` you specify the path of the config file in the install prefix. Here if we want our library to be found by calling `find_package(Maths ...)` we **must** set the file name to `MathsConfig.cmake`.

We also set a namespace. Well... CMake doesn't really have a concept of namespace. It is actually a prefix which is prepended to the origin target name `Maths`. So the user will have to link to `CO::Maths`. The two colons `::` are not special characters and can be anything:

- `NAMESPACE CO_` -> `CO_Maths`
- `NAMESPACE COLib` -> `COLibMaths`
- Nothing -> `Maths`

> The use of namespaces is encouraged because you really don't know which other targets your user will already have when importing yours.

Installing the exported taget won't be of any use if you don't specify which target(s). This is done by setting your export name as a parameters of your `install(TARGETS ...)` command. So let's go back to it:

```
install(TARGETS maths EXPORT MathsExport
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION bin
  RUNTIME DESTINATION bin
)
```

As you see we added `EXPORT MathsExport`. The name of the export must match exactly the one you wrote in `install(EXPORT ...)`.

With this, you can export several targets in one export file.

> Use `install(EXPORT ...)` to create the config file.

## Install include directories

There's one more thing to do with our export : setting the include directories.

Currently if you run CMake with such a CMakeLists, you'll end up with an error like this:

```
CMake Error in CMakeLists.txt:
  Target "Maths" INTERFACE_INCLUDE_DIRECTORIES property contains path:

    "/<YOUR_SOURCE_DIRECTORY>/include"

  which is prefixed in the source directory.
```

Problem: this is not because you copied your entire include directory into the install prefix by saying:

```cmake
install(DIRECTORY inc/ DESTINATION include)
```

... that CMake will magically understand that your exported target uses them instead of the directory in your source code.

To make things short, at the very beginning of your CMakeLists you said:

```cmake
target_include_directories(maths PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/inc
)
```

And as far as it knows, it's still the case even for your `IMPORTED` target.

We must provide CMake with two informations:

- `${CMAKE_CURRENT_SOURCE_DIR}/inc` is only when I and building the project.
- `<prefix>/include` must be used by the `IMPORTED` target.

Both statements can be done using generator expressions.

A generator expression is a value which depends on the build and thus is evaluated at that moment, and not during configuration step.

Here we are interested in two expressions: `$<BUILD_INTERFACE:Value>` expands to `Value` if we are building our target while `$<INSTALL_INTERFACE:Value>` will do it if we are installing the target.

Let's change our `target_include_directories`:

```cmake
target_include_directories(maths PRIVATE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/inc>
  $<INSTALL_INTERFACE:include>
)
```

# Checking the version

Generally your library will change over the time. You will thus deliver new versions of it and sometimes people will want to use an incompatible version.

With `find_package()` your users can ask for any specific version:

```cmake
# Any version would fit
find_package(Maths REQUIRED)

# Any version compatible with this one would fit
find_package(Maths 1.3.7 REQUIRED)

# I want THAT version exactly
find_package(Maths 1.3.7 EXACT REQUIRED)
```

> A version number take the form `MAJOR[.MINOR[.PATCH[.TWEAK]]]`.

For this to work with our library, we need a secondary file, the "version config" file. The name must match our config file with "Version" appended to it: `MathsConfigVersion.cmake`. This file must be placed in the same folder as the config file:

```
MathsLib
├── include/
│   ├── include/m_areas.h
│   └── include/m_volumes.h
├── lib/
│   └── maths.a
├── README.md
├── LICENCE
├── MathsConfig.cmake
└── MathsConfigVersion.cmake
```

The "config version" script is an optional file. If `find_package()` finds it and a version was asked by the user, it is ran. Checking the version is done by comparing the version the in `find_package()` by the user and the version we set in `project()`.

All we have to do is to asks CMake to generate the version file and to install it.

This is done by using the `CMakePackageConfigHelpers` module. Include it at the beginning of your CMakeLists:

```cmake
include(CMakePackageConfigHelpers)
```

Now we can call `write_basic_package_version_file`:

```cmake
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/MathsConfigVersion.cmake
  COMPATIBILITY SameMajorVersion
)
```

`COMPATIBILITY` is used to specify how we consider two different versions as compatibles. We use `SameMajorVersion` which means that as long as the first number of both version match, it's ok. We could also accept that any version more recent is ok with `AnyNewerVersion` or simply enforce a strict version matching with `Exact`.

> Usually, `SameMajorVersion` is a good choice for compatibility policy.

Now we have generated the version file, we want to install it:

```cmake
install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/MathsConfigVersion.cmake
  DESTINATION lib/cmake/Maths
)
```

# Done!

Good! Our library is now totally usable by another CMake project by simply calling `find_package`!

Ok, so let's put everything in one file again:

```cmake
cmake_minimum_required(VERSION 3.10.0 FATAL_ERROR)

project(Maths VERSION 1.0.0)

include(CMakePackageConfigHelpers)

add_library(maths STATIC
  inc/m_areas.h
  inc/m_volumes.h
  src/m_areas.c
  src/m_volumes.c
)

target_include_directories(maths PRIVATE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/inc>
  $<INSTALL_INTERFACE:include>
)

# Install the target and export it as MathsExport
install(TARGETS maths EXPORT MathsExport
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION bin
  RUNTIME DESTINATION bin
)

# Install other files
install(DIRECTORY inc/ DESTINATION include)
install(FILES LICENSE README.md DESTINATION .)

# Create the imported target using MathsExport
# Into MathsConfig.cmake
install(EXPORT MathsExport
  DESTINATION lib/cmake/Maths
  NAMESPACE "CO::"
  FILE MathsConfig.cmake
)

# Create and install the version checking file
write_basic_package_version_file(
  ${CMAKE_CURRENT_BINARY_DIR}/MathsConfigVersion.cmake
  COMPATIBILITY SameMajorVersion
)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/MathsConfigVersion.cmake
  DESTINATION lib/cmake/Maths
)
```

# Bonus: export from binary directory

Let's say we work on two different repositories at once: an executable and our awesome Maths library. How convenient would it be to use `find_package(Maths)` from our executable CMakeLists.txt. But constantly installing your Maths library doesn't seem right.

It's possible to write our target export in a file dedicated to the binary directory of our library. Simply add this line:

```cpp
export(EXPORT MathsExport NAMESPACE "CO::" FILE MathsConfig.cmake)
```

No need to install anything! Now we can import our Maths package from its binary directory. Just let's don't forget to set `CMAKE_PREFIX_PATH` correctly.

