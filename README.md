- [Introduction](#introduction)
- [First principles](#first-principles)
- [Development environment](#development-environment)
  - [Programming languages](#programming-languages)
  - [Version control](#version-control)
  - [Handling dependencies](#handling-dependencies)
  - [Continuous integration](#continuous-integration)
  - [Deployment](#deployment)
  - [Documentation](#documentation)
- [Runtime failures](#runtime-failures)
  - [Out of memory (OOM)](#out-of-memory-oom)
  - [Segmentation fault](#segmentation-fault)
  - [A note on return status](#a-note-on-return-status)
  - [Races](#races)
- [Architectural patterns](#architectural-patterns)
  - [Transformation API symmetry](#transformation-api-symmetry)
  - [Configuration-driven development](#configuration-driven-development)
  - [Process isolation](#process-isolation)
  - [Let it crash](#let-it-crash)
  - [Let it vanish](#let-it-vanish)
  - [I/O isolation](#io-isolation)
  - [State machines](#state-machines)
  - [Fail early](#fail-early)
  - [Single source shared parameters](#single-source-shared-parameters)
- [Project bootstrapping](#project-bootstrapping)
  - [Infrastructure first](#infrastructure-first)
  - [Simulation](#simulation)
  - [Integration tests](#integration-tests)
- [Algorithms and data structures](#algorithms-and-data-structures)
  - [Performance](#performance)
  - [Volumetric data](#volumetric-data)
  - [Rotations / orientations](#rotations-orientations)
- [Coordinate systems](#coordinate-systems)
  - [Geodetic coordinates](#geodetic-coordinates)
- [Date, time, and locale](#date-time-and-locale)
- [Protocols and serialization](#protocols-and-serialization)
- [General style policies](#general-style-policies)
  - [Naming](#naming)
  - [Formatting](#formatting)
  - [Use tools](#use-tools)
  - [Do not abuse tools](#do-not-abuse-tools)
- [Coding styles](#coding-styles)
  - [Follow common styles](#follow-common-styles)
  - [Don’t follow styles literally](#dont-follow-styles-literally)
  - [General rules](#general-rules)
  - [C++](#c-1)
- [Package conventions](#package-conventions)
  - [Naming](#naming-1)
  - [Layout](#layout)
  - [Installation paths](#installation-paths)
- [Networking](#networking)
  - [TCP](#tcp)
- [Fixing robot models](#fixing-robot-models)
  - [Inertia](#inertia)
- [Other](#other-1)
  - [ROS](#ros)
  - [Telemetry](#telemetry)

Introduction
============

This is a collection of my passive-aggressive notes on software development,
which are mostly obvious, but have to be repeated time to time. It primarily
concerns C++, Linux, and robotic applications, in other words, embedded /
headless systems consisting of a large number of heterogeneous software
components, running with minimal human intervention.

First principles
================

1.  Human friendliness: minimize everyone’s mental work.

    - The less you need to think the more you can do.

2.  Consistency: make your choices and stick to them.

    - Ensures (1).

3.  Automation: any action that is performed more than once must be automated.

    - Ensures (1) and (2).

Development environment
=======================

Programming languages
---------------------

- Minimize the number of programming languages used in your system, this
  dramatically decreases maintenance costs, facilitates code reviews, improves
  code quality, etc.

- Even more importantly: don’t mix languages in the same software component,
  e.g., don’t execute python code from C++ code. It is better to implement it in
  the same language, even if it is not the default one for the system as a
  whole.

- Likewise, use the same version of compiler and language versions
  across-the-board.

- Stick to the default compiler version provided by your distribution (this is
  particularly true for C++): new language features are never worth the
  maintenance costs associated with integration of a custom compiler version.

- Compiled languages are usually preferable to interpreted languages:
  compilation is pretty much a mandatory static analysis step. Sometimes the
  only way to check validity of an interpreted program is to execute it, which
  is quite inconvenient.

### Three elephants

- `C++` is the main language for implementation of onboard components. Its
  flaws, such as complexity, bad syntax, and lack of fool-proofing, are well
  compensated by performance, expressive power, vast amount of development tools
  and reusable open-source libraries. `C++` is also under active development
  currently, so it is catching up with new concepts relatively quickly.

- `python` is for off-board data processing and analysis, e.g., machine
  learning. Sometimes you can use other languages for this purpose, but
  `python`’s wide adoption and the number of 3rd party libraries makes other
  choices impractical. `python` is also often used for scripting, but I strongly
  believe that `shell` is a better alternative for such use cases.

- `shell / make` are often overlooked, but are perfect for various automation
  tasks, such as running tests in bulk, handling files and directories, etc.

    - The main power of `shell` is not in the language itself, but in various
      utilities that help to address your tasks in a much more expressive way
      than any scripting language: `sed`, `xargs`, `grep`, `cut`, `sort`, etc.

    - One can argue that `make` functionality is covered by `shell` scripts,
      but in my opinion it allows to achieve the same goals in a cleaner and
      more concise way.

    - Young developers often see `make` as a deprecated build tool, which is
      wrong -- it is a general purpose automation utility. It was not
      superseded by cmake or whatnot in this context.

    - There are a few modern alternatives to `make` such as
      <https://github.com/go-task/task> or <https://github.com/casey/just>.

    - Don't forget that `/bin/sh` is not the same thing as `/bin/bash`.

Version control
---------------

- Main branches, e.g., `master`, should be protected from direct pushes. The
  protection must include either review requirement or CI check, and should
  apply to repository owners as well.

- The main value of reviews is knowledge transfer in the team, both regarding
  the developed software system and programming in general.

Handling dependencies
---------------------

- Dependencies should be installed using system binary packages. If a system
  package is outdated, suck it up and use it anyway unless there are dangerous
  flaws that directly affect your application.

- If a system package does not exist or unusable, consider other options:
  3rd-party package repositories, `vcpkg`, `conan`. For new projects `nix` and
  `guix` should be considered first. However, it may be more convenient to build
  a package by yourself to handle dependencies consistently.

- ROS development is usually performed in workspaces
  (<http://wiki.ros.org/catkin/workspaces>), where you can work with multiple
  packages coming from various version control systems or tarballs. There exist
  several tools for building packages in workspaces taking dependencies into
  account: `catkin_make` (consider it to be deprecated), `catkin_tools`,
  `colcon`. `wstool` and `vcstool` help to manage code sources and versions. I
  find the ‘workspace’ approach to be very convenient, but it may require
  injection of package meta-information in non-ROS packages to handle
  dependencies properly (the process is sometimes referred to as catkinization).
  A description of the process can be found at
  <http://wiki.ros.org/ROS/Tutorials/catkin/CreatingPackage> -– it boils down to
  adding package description in `package.xml` file and optional special commands
  in `CMakeLists.txt`.

- You can also incorporate dependencies into your packages in several ways, all
  of which are inferior to workspaces, but may be needed, e.g., to perform
  non-invasive catkinization:

    - `git` submodules are not too bad, but depend on external repositories
      which makes them fragile and difficult to fork. Also, they are not
      handled by git transparently, so you have to remember to do recursive
      fetch, etc.

    - `cmake` external projects should never be used -– in addition to being
      fragile as git submodules, they are difficult to be used in the right way
      due to interface complexity, and introduce annoying build time
      limitations due to mixing of building and fetching phases.

    - Me personal favorite for addressing this task and the most robust
      approach is `git read-tree`, which allows to inject code directly into
      your project repository.

Continuous integration
----------------------

- Continuous integration is often perceived as an isolated environment, which
  leads to poor design choices in its implementation. Literally all tasks
  performed in CI, must also be performed manually during development:
  compilation. testing, static/dynamic analysis, binary package generation, etc.
  Hence, you should implement a development environment which supports those
  operations and then build CI based on it. A notable example is
  <https://github.com/asherikov/ccws>.

- Do not use CI pipelines for scripting -– all essential functionality must be
  performed in a generic scripting language to facilitate migration between
  different CI systems.

- It is usually beneficial to treat service failures differently in CI and in
  deployment: in the first case we want to detect such situations to be able to
  fix them, in the second we want the system to recover.

- Test failures in CI can be handled in two ways:

    - testing continues in order to collect results for all tests, which is
      good for periodic, e.g., nightly, jobs;

    - testing stops immediately, which is preferable for merge request tests
      since the author receives feedback as soon as possible; in this case it
      is also useful to arrange tests so that the quick ones are executed
      first.

Deployment
----------

Deployment can be divided into three stages:

1.  system image generation;
2.  system configuration;
3.  binary package deployment.

At this point `docker` is perceived as the way to go, but keep in mind that it
covers only the third stage and has other drawbacks. Binary packages, however,
for example Debian packages, can address the second stage as well using embedded
installation scripts.

### Test deployments

Development of robotic systems usually requires quick deployments of packages
for testing. A common approach to address this task in ROS environment is to
perform a plain copy of locally compiled packages to a target machine, e.g.,
from a workspace installation directory. This method does not facilitate
tracking of deployments, which may become a significant issue when the target is
shared by many developers. It is, however, even more inconvenient to perform
binary package releases for this purpose –- the ROS buildfarm system is not
designed for this, which is, in my opinion, a direct consequence of treating
this task as non-interactive and “pure-CI”. I’ve tried to address it in
<https://github.com/asherikov/ccws> by allowing developers to generate binary
packages locally.

Documentation
-------------

- Document your classes, methods, and source files using doxygen
  <http://www.doxygen.org/>. There are a some alternatives, e.g.
  <https://github.com/cppalliance/mrdocs>,
  <https://github.com/NaturalDocs/NaturalDocs>, <https://github.com/hdoc/hdoc>,
  <https://github.com/vovkos/doxyrest>,
  <https://github.com/copperspice/doxypress>, but they do not seem to be
  significantly better than doxygen ATM.

- Each repository must contain a README.md file with a brief description of its
  purpose.

- Diagrams are often useful or necessary to understand internals of a system.
  Classical “industrial” approach to software development implies that you
  design a system by drawing diagrams first and then translating them to actual
  code, ideally with some automatic code generation tool. I find this method to
  be rather inconvenient: I am much more efficient while working with code or
  text in general rather than graphics. Moreover, it is more difficult to keep
  design diagrams in sync with implementation. For these reasons, I prefer tools
  that extract information from the code, e.g., doxygen. Another interesting
  example is <https://github.com/boost-ext/sml> which allows generation of
  finite state machine diagrams from their C++ implementations.

Runtime failures
================

Out of memory (OOM)
-------------------

- Linux (at least Ubuntu) does not handle OOM well by default
  <https://bugs.launchpad.net/ubuntu/+source/linux/+bug/159356>, so you have to
  take extra measures to avoid such situations and protect your system from
  freezes.

Segmentation fault
------------------

- An important property of segmentation fault is that it is generally
  non-recoverable: if your program tried to write to someone else’s memory, it
  is quite likely that it has already trashed its own memory.

A note on return status
-----------------------

- If your application was terminated by a signal, the return code indicates the
  signal code, e.g., `-6` corresponds to `SIGABRT` and usually indicates an
  exception in C++ code, `-9` –- `SIGSEGV`. If exit code is unsigned, e.g., 134,
  subtract 128.

Races
-----

In my experience races are quite common and particularly difficult to debug: a
service behavior may depend on states of multiple other services, hardware
components, etc, in which case it is necessary to be extra cautious when
responding to changes in these states and avoid implicit assumptions on the
sequence of their appearance.

Architectural patterns
======================

Transformation API symmetry
---------------------------

If you transform data to a different representation, e.g., by writing it from
memory to a file, you or somebody else almost always is going to need to perform
the reverse transformation. Take this into account when designing new API and
don’t neglect implementation of reverse operations: that helps to find bugs,
facilitates testing, makes your code and data more coherent. Also, assuming that
users must manually compose input files for your code is a dick move.

Examples:

- `URDF` format is commonly used for robot model description in ROS. There is a
  standard parser for it, but no emitter. Unfortunately you may need to modify
  and store model automatically, e.g., when performing parameter identification,
  in which case you’ll have to implement some ugly workarounds.

- The second example comes from STL library where you can find `std::to_string`
  (since C++11) but no `std::from_string`. For this reason,
  `boost::lexical_cast` should always be preferred since it works both ways.

Configuration-driven development
--------------------------------

- Start implementing services (ROS nodes) with a configuration file, this way
  you can avoid hardcoded constants and save time during development and
  testing.

- Configuration files are better than command line arguments: they are more
  general and scalable.

- Choose `YAML` or `JSON` by default. `XML` is unnecessarily verbose, lacks
  array type, has ambiguous choice between attributes and child nodes. Custom
  formats such as `TOML` should also be avoided since they are generally
  inferior to `YAML`/`JSON`.

- Use serialization / reflection libraries to abstract from a particular file
  format, e.g., <https://github.com/asherikov/ariles>.

- Don’t forget to respect transformation API symmetry –- sooner or later you are
  going to need to modify configuration during execution and export it for
  future use.

- Global configuration of the system, e.g., via ROS parameter server, is quite
  convenient, but should always be used as read-only from services. Use messages
  or services to pass parameters between services instead.

- Environment variables should not be used for controlling your services, but
  might be necessary to alter behavior of 3rd-party applications and scripts.

Process isolation
-----------------

Isolation between processes is way better than between threads. Use this to your
advantage by isolating 3rd-party, potentially unreliable, or non-critical
components of you software system in standalone processes. For the same reason,
I recommend using ROS nodelets only when absolutely necessary, they are just a
workaround for poor IPC.

Let it crash
------------

“Let it crash” approach to failure handling comes from Erlang – a language
designed for telecommunication applications <a
href="https://en.wikipedia.org/wiki/Erlang\_(programming_language)#%22Let_it_crash%22_coding_style"
class="uri">https://en.wikipedia.org/wiki/Erlang\_(programming_language)#%22Let_it_crash%22_coding_style</a>

You have to accept that your programs are going to crash, which means that:

- you have to have a restarting mechanism in place, e.g., a service manager;

- in some cases it is better to crash than try to recover on the fly, e.g.,
  segmentation faults mentioned above;

- you should exploit process isolation as described above to localize failures.

Let it vanish
-------------

This is also an old and ubiquitous design pattern, but I don’t recall a specific
term for it so I named it to relate to “let it crash”. The main idea is that you
have to design your software to lose data when appropriate:

- If you are working with large amounts of data you may simply run out of memory
  if you don’t limit size of you buffers.

- Even if your data buffers are limited you may run into situations when they
  are filled to a degree when data gets too old by the time the processing code
  receives it. This is relevant for many data-rich sensors such as cameras,
  lidars, etc.

I/O isolation
-------------

I/O is expensive, especially when it is performed via interactive terminals. It
is a common practice to localize I/O in separate threads to avoid interference
with time critical operations. For a similar reason, you should generally delay
transferring of debug and logging information until the end of time critical
methods, such as control loops.

State machines
--------------

State machines are often employed for representing robot behaviors, but in my
opinion they should be used more for implementation of individual services. Any
time you work on a service that changes its behavior in response to some command
messages, for example using <http://wiki.ros.org/actionlib>, it is necessary to
consider a finite state machine.

Fail early
----------

Early failures usually have lower cost, for example, if you can detect a failure
during source code compilation you are saving time on deployment and tests, if
you can validate drone state before takeoff you can potentially avoid a crash,
and so on.

Single source shared parameters
-------------------------------

Parameters shared by different components of the stack should come from the same
source. If they are stored in multiple independent locations, they are
inevitably going to get out of sync, which, in turn, would lead to failures or
poor performance. Extraction and distribution of shared parameters should not
necessarily happen at runtime, on the contrary, it may be preferable to perform
this during startup or build phases in order to comply with “fail early”
principle.

For example, an URDF / SDF model of a robot can be the source of its total mass
and geometric dimensions.

Project bootstrapping
=====================

There are a few things that should be done first when starting a new project.

Infrastructure first
--------------------

- Prepare templates for new packages, source code files, copyright notice, etc.

- Bring up build and development infrastructure: CI, version control system.

- Integrate static / dynamic analysis tools before starting coding:

    - such tools are the most valuable at the early stages of development;

    - their late integration may require too much resources and is unlikely to
      be ever fully completed.

Simulation
----------

Simulation is a crucial component for testing your system. All code should
always be validated in simulation before deployment to save time and reduce
risks. For this reason simulation must be implemented as soon as possible.

Integration tests
-----------------

Integration tests are more important on early stages of development: a simple
test that starts your system with a simulator and terminates immediately has
more value than a single thoroughly unit-tested component. Integration tests
focus your attention on the whole system rather than its parts. However, complex
integration tests are difficult to maintain and are prone to become fragile on
later stages of development, at which point you should support them with
component specific tests.

Algorithms and data structures
==============================

Performance
-----------

- Performance optimization is often focused on computational complexity of
  algorithms, i.e., the mount of resources required to run them
  (<https://en.wikipedia.org/wiki/Computational_complexity>). In practice, it is
  usually a bad approach when you work with non-trivial data: the type of
  resources and access to them are much more important. Pay attention to memory
  access and especially I/O. For example, loading a geographic model from a file
  every time you perform a coordinate conversion is not going to be a good
  solution no matter how much you reduce the number of conversions.

- Legacy algorithms often measure their complexity in number of single floating
  point operations. Modern hardware is actually much better at performing
  arithmetic operations in bulk due to vectorization instructions, e.g., see
  <https://eigen.tuxfamily.org/index.php?title=FAQ#Vectorization>. For this
  reason, brute-force algorithms that use plain linear algebra may perform
  better than classic algorithms containing loops, conditionals, and recursion.
  Note that interpreted languages, such as `Matlab` and `python`, can also
  benefit from matrix-based operations for slightly different reasons
  <https://www.mathworks.com/help/matlab/matlab_prog/vectorization.html>.

Volumetric data
---------------

OcTree (<https://octomap.github.io/>) is commonly used for representation of
volumetric data, but it is not always a good solution:

- when you build a map based on readings from range sensors such as lidars don’t
  expect to benefit from tree pruning of occupied cells -– the scans give you a
  thin surface of objects, so the tree leafs cannot be merged together;

- in terms of performance OcTree is inferior to VDB
  (<https://www.openvdb.org/>);

- OcTrees, however, are useful when you need to work with different resolutions
  of the same map, this structure naturally supports such slicing.

Rotations / orientations
------------------------

There are multiple ways to represent rotations. Some people claim than
quaternion is always the right thing to use – they are wrong.

### Euler angles

- Euler angles are bad, you should always avoid them, but there is one
  exception: user interfaces. Euler angles are more intuitive than other
  representations and are good enough for simple orientations like ‘pitch
  forward by 30 degrees’.

- Stick to roll-pitch-yaw convention (RPY) to minimize confusion. Be careful
  when using third party software, sometimes it uses mislabeled YPR convention.

### Rotation matrices

- Rotation matrices are very handy when you need to construct rotation using
  basis vectors or vice versa.

- Application of rotations using matrices may be faster than using quaternions,
  since it is a plain matrix multiplication.

- Rotation matrices have redundant variables and therefore tend to accumulate
  numerical errors.

### Angle-axis

- Convenient complement for rotation matrices when you need to rotate a frame by
  a certain angle along specific axis.

### Quaternions

- Should be used by default.

Coordinate systems
==================

Geodetic coordinates
--------------------

- Do not use `LLA` abbreviation for geodetic coordinates – it is ambiguous since
  both `Lat-Lon` and `Lon-Lat` orders are common in practice.

- There are two commonly used altitude measurements: with respect to the mean
  sea level, and to the WGS84 ellipsoid. Mean sea level (MSL) is more fragile
  and computationally expensive due to sea level determination logic. Ellipsoid
  altitude should be preferred in practice.

Date, time, and locale
======================

- Dates must always be specified in YYYY-MM-DD format in order to facilitate
  sorting, e.g., `2018_10_02`. Pad months and days with zeros when necessary.
  While we are on this topic we should also mention absurd MM-DD-YYYY convention
  and make fun of people who use it.

- Time zones and summer/winter time transitions can be confusing, for this
  reason I am inclined to use UTC time in deployment.

- 24h, aka ‘military’, time format is easier to read and parse and should always
  be preferred to a.m./p.m. convention.

- Don’t forget that some data, including dates and floats, is sometimes
  automatically formatted during I/O in accordance with system locale. For
  example, French locale uses comma to separate decimal part of floating point
  numbers instead of dot, which leads to funky issues like this
  <https://github.com/zeux/pugixml/issues/469>. To be on a safe side, enforce
  `C` (`POSIX`) locale in deployment and while performing formatted I/O.

Protocols and serialization
===========================

- Do not use 32-bit floats for storing geodetic coordinates, this leads to a
  substantial, a couple of meters, errors simply due to representation limits.
  If memory usage is a concern, use 32-bit integers instead to get much smaller
  and uniform errors.

- Do not store UUID as string – weirdly enough it is a common thing, each UUID
  is 128 bit long label
  (<https://en.wikipedia.org/wiki/Universally_unique_identifier>) and should be
  stored like that.

General style policies
======================

Naming
------

### Ordering numbers

- Start numbering with 0.

- Pad numbers with zeros: `001`, not `1`.

### Filenames

- Numbers in filenames should be strictly increasing, don’t fill the gaps when
  adding new files in order to avoid naming collisions in repository history.
  For example, if there are `test_005` and `test_010`, a new test must be named
  `test_011`.

- Avoid all punctuation symbols except `-`, `_`, and `.`. Other symbols often
  have to be escaped in the command line and break word auto-selection logic in
  terminals.

- Filenames should be all lowercase with dashes or underscores as word
  separators.

### Hierarchies

- Names are often organized in hierarchies: namespaces, field names in `YAML` or
  `JSON` files, filenames in directories, etc. It is a common stylistic mistake
  to duplicate parent names in child names, e.g.,
  `namespace logger {   class LoggerParameters; }`. This repetition is redundant
  and should be avoided: `namespace logger { class Parameters; }`. Another
  example is `ROS` convention of subdirectory naming in robot description
  repositories, e.g., <https://github.com/ros-naoqi/pepper_robot>, where each
  subdirectory includes redundant robot name. Yes, the reason is to match
  directory and package names, but practical value of this convention is next to
  zero.

### Versions

- Classic three numbers or dates are ok, code names as used by `ROS` are not:
  most developers are not native English speakers –- remembering these names (I
  literally had to check `eloquent` in a dictionary) and their alphabetical
  ordering is by no means easier than numbers.

### Prefix

- Names often include prefixes that narrow scope left to right. The general goal
  is to indicate a specific subsystems the named object belongs to in a
  non-ambiguous, but concise way.

- Choose a company name prefix, e.g., ‘mcy’ for ‘mycompany’, and use it
  consistently when needed to identify your packages, code, variables, etc. Try
  to pick something that is unlikely to lead to a naming collision.

### Other

- Usage of abbreviations in any names should be avoided.

- Don’t waste your time on renaming things in accordance with the current
  political agenda. This is just sad.

Formatting
----------

- Configure your text/source editor to drop trailing whitespaces to avoid
  polluting repository history and noisy diffs.

- Use tabulations only if required, e.g, in makefiles. There is a special place
  in hell for people who mix tabs and whitespaces for indentation.

- Formatting of human-readable files should never favor horizontal or vertical
  space preservation over readability: separate logical blocks with multiple
  empty lines and/or comments, add extra linebreaks and whitespaces, etc.

- Use 4 spaces for indentation: 2 is not enough, 8 is too much, anything else is
  a perversion.

- When editing files try to minimize the number of affected lines – this makes
  diffs more compact and readable. There are certain formatting conventions that
  can be helpful, e.g., when multiple parameters are passed to a function each
  of them, as well as the closing brace, should be on separate lines.

Use tools
---------

- If your conventions are not enforced with some autoformatter or linter they
  are inevitably going to be broken.

- Compile and lint your code and files with all warnings enabled and treat
  warnings as errors, otherwise they are useless. There can be exceptions for
  specific warnings of course. Counterarguments like “-Werror Introduces a
  Toolchain Dependency”
  (<https://embeddedartistry.com/blog/2017/05/22/werror-is-not-your-friend/>)
  are weak, instead of dropping `-Werror` enforce language standard and test
  with all supported toolchains.

Do not abuse tools
------------------

- LLM-based code auto-completion tools, such a `GitHub Copilot` are quite good
  at generating boilerplate comments. The value of such comments, however, is
  often next to zero. Do not pollute your sources with comments like
  `/* vector of integers */`, they are not only useless and distracting, but may
  also be misleading if they get out of sync with the code.

Coding styles
=============

Follow common styles
--------------------

### C++

- <https://google.github.io/styleguide/cppguide.html>
- <http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>
- <http://www.stroustrup.com/JSF-AV-rules.pdf>
- <https://github.com/janwilmans/guidelines>
- <https://github.com/cpp-best-practices/cppbestpractices>

Don’t follow styles literally
-----------------------------

Even widely accepted conventions are sometimes inconvenient or dangerous, such
as double space indentations or usage of `auto`. Ignore non-compromising zealots
that defend bad practices because “our father did it that way” or “this is the
modern style”.

General rules
-------------

- Function names, variable names, and filenames should be descriptive; avoid
  abbreviation. Types and variables should be nouns, while functions should be
  “command” verbs, e.g., `openFile()`.

- Prefixes are usually more apparent than suffixes, so the former is preferred
  in special names, e.g., `g_global_variable` instead of `global_variable_g`.

- Minimize scope of entities –- that improves your code granularity and makes it
  easier to maintain.

- Structurize data: Sometimes, you may see code where multiple independent
  variables are stacked in a single vector, e.g., position and orientation. Such
  style must not be allowed in modern C/C++/python code. Use classes or
  structures with appropriately named members instead, if not possible –-
  implement wrappers.

C++
---

### General rules

- Enforce language standard using corresponding compiler flag. Don’t use
  standard extensions.

- Dependency on Boost is almost inevitable in large projects, don’t try to fight
  it by integrating newer compiler with the latest standard support. Don’t
  forget that unlike STL you can cut out parts of Boost if necessary.

- Avoid paradigm mixing: C++ is primarily an object oriented language, don’t try
  to turn it into something else. If you find yourself implementing a 30-line
  long method inside a return statement of a lambda function that returns
  another lambda function you have made a few wrong turns.

### Fundamental (built-in) types

- In general, there is no reason in trying to optimize something by being smart
  with integer types, e.g., using `short` type, or `uint8_t`. STL actively uses
  `std::size_t` for unsigned integers and `std::ptrdiff_t` for signed integers,
  so those should be the default integer types.

- The same applies to floating point numbers, e.g., use `double` unless `float`
  is really necessary.

- If you need custom integer types use those that are explicit regarding their
  size and sign, e.g., `uint16_t`.

### Type names

- Type names (classes, structures, typedefs, and enumerations) start with a
  capital letter and have a capital letter for each new word, with no
  underscores: `MyExcitingClass`, `MyExcitingEnum`.

### Enumerations

- Names of the enumerators should be in upper case with underscores.

- Do not use global variables or defines in order to represent logically related
  values –- always use an enumeration in such cases.

- All enumerations must be defined within some container class scope. It is
  recommended to use some wrappers, e.g,
  <http://aantron.github.io/better-enums/>.

- The general rule for handling enumerations is to use a switch: known values
  should be handled as needed, all unknown values must be captured by `default`
  case and result in a failure. This approach ensures that any future extensions
  of the enumeration are going to result in failures which are easy to detect
  and fix. For example:

    BETTER_ENUM(DroneStatus, int, UNDEFINED = 0, INACTIVE, FLIGHT)

    DroneStatus status = DroneStatus::UNDEFINED;
    switch(status)
    {
    case DroneStatus::INACTIVE:
        break;

    case DroneStatus::FLIGHT:
        break;

    case default:
        throw();
    }

### Variables

- Variable names are all lowercase, with underscores between words, i.e.,
  `my_table_name`.

- Global variables must be avoided. If it is not possible, their names must have
  `g_` prefix, e.g., `g_my_global_variable`.

### Functions and methods

- Functions should start with a lowercase letter and have a capital letter for
  each new word without underscores.

- Parameters which do not serve as outputs must be `const`.

- Input parameters must be passed by reference unless their type is fundamental:
  integral, floating point, or void, see
  <https://en.cppreference.com/w/cpp/language/types>.

- Output parameters must be gathered at the end of the parameter list:
  `doSomething(input1, output1, output2, input2 = <default_value>)`.

- Avoid passing a value as a function parameter, use a variable instead, or add
  a comment, e.g., `doSomething(/*verbose*/ false)`.

### Classes

- It is not allowed to mix definitions with different access modifiers, i.e.,
  methods and members with the same access modifier should be gathered together.

- It is recommended to use access modifiers multiple times to visually group
  members or methods.

- Names of member variables should have trailing underscores, i.e.,
  `member_name_`.

- Methods, which do not change the corresponding class, must always be marked
  with `const`.

- In general, destructors of base classes must be defined and must be protected.
  If it is necessary to allow polymorphic destruction, the destructor should be
  defined as public and virtual.

- Constructors of base classes should be protected as well.

- Avoid implementing complex initialization in class constructors, use
  `initialize()` methods instead:

    - If constructor accepts dynamic parameters it may force using pointers for
      its instantiation, which is a good approach in some cases, but in my
      experience this it is a bad practice to enforce this pattern on
      developers –- let them choose how to instantiate classes.

    - Templated constructors do not allow explicit parameter specification --
      templated parameters must be deduced from constructor inputs.

- Many common styles and static analysis tools insist on initialization of
  member variables on declaration and in member initializer lists of
  constructors when possible, which is obviously not always the case. When such
  conventions are enforced you end up with member initialization logic scattered
  all over the place. In my opinion a dedicated initialization method called
  from a constructor is the most transparent approach.

### Macro

- If macro can help you to reduce code verbosity and avoid repetition, you
  should use it unless there is another way to achieve the same results. Banning
  macro completely, especially in third party libraries, is simply retarded.
  Copy-pasting is a much bigger sin than presumable obscurity introduced by
  macro.

- Macro name must be in all capitals with underscores and have a prefix to avoid
  collisions, e.g., `MYCOMPANY_DEBUG`.

- If you have a macro that enables / disables debugging, make it numeric in
  order to have finer control over debugging level, e.g.,

    - greater than 0: enables printing/generation of extra debug data, usually,
      at the expense of performance.

    - greater than 10: disables recovery behaviors, i.e., services are going to
      crash rather than recover in some cases, which is easier to detect in
      tests.

### Namespaces

- Names of namespaces should be in lower case with possible underscores.

- Minimize scope of `using namespace ...` directives, never use them in public
  header files.

- All your code must be enclosed in `mycompany` namespace.

- As a rule of thumb, source code in a particular package should be additionally
  enclosed in a corresponding namespace. For example, code in `mypackage` should
  be in `mycompany::mypackage` namespace.

- Omit leading namespaces when possible and reasonable, e.g.,
  `mypackage::doSomething(...)` instead of
  `mycompany::mypackage::doSomething(...)`.

### Templates

- Names of template parameters should follow the same conventions as the
  category they belong to (typenames, variables), but their names must include
  `t_` prefix: `t_BaseClass`, `t_integer_variable`.

### Header guards

- Although `#pragma once` is not part of the standard, it is widely supported
  and is more concise and less error-prone than classic header guards
  (`#ifndef ... #define ... #endif`).

### Avoid auto

`auto` can break semantics of your program without breaking its syntax. Consider
the following program using `Eigen`:

    #include <iostream>
    #include <Eigen/Core>

    namespace
    {
    Eigen::Matrix3d getRandomSymmetricMatrix()
    {
        Eigen::Matrix3d m = Eigen::Matrix3d::Random();

        return (m.transpose() * m);
    }

    Eigen::Matrix3d getRandomSymmetricMatrix2()
    {
        auto m = Eigen::Matrix3d::Random();

        return (m.transpose() * m);
    }
    }

    int main()
    {
    std::cout << getRandomSymmetricMatrix() << std::endl << std::endl;
    std::cout << getRandomSymmetricMatrix2() << std::endl;

    return (EXIT_SUCCESS);
    }

Example output:

      1.22374 -0.234887  0.579107
    -0.234887  0.994478 -0.659169
     0.579107 -0.659169   1.07338

    -0.482262  0.839256 0.0753774
     -1.02165 -0.202315  0.942273
      0.15891  -0.84441  0.247298

Note that output of the second function is wrong, the reason for that is that
`Random()` returns a random matrix generator rather than a matrix, so the second
function returns a product between two different matrices generated on spot.
Such errors cannot be detected by compiler or sanitizers, since the code is 100%
correct. They are also difficult to pick up by reading the code – you have to
know how Eigen API works and what to look for. Even though the issue is well
known and documented <https://eigen.tuxfamily.org/dox/TopicPitfalls.html#title3>
developers following the ‘modern’ style fall for it over and over, e.g.

- <https://stackoverflow.com/questions/59586537/eigen-gives-wrong-result-when-not-storing-intermediate-result>
- <https://stackoverflow.com/questions/55962829/eigen-c-how-can-i-fixed-the-values-after-a-random-matrix-initialization>

Another possible side-effect of using `auto` with `Eigen` is performance
degradation: `auto mat3 = mat2 * mat1` here `mat3` is an expression rather than
a result of multiplication – it is going to be reevaluated every time it is used
in the code.

Sometimes `auto` is encouraged to avoid typing long typenames, e.g.,
`std::map<std::string, std::pair<int, std::string>>::const_iterator`. This is a
wrong solution to this problem:

- you should always define proxy types to encapsulate name complexity, e.g.,
  `using MyMap = std::map<std::string, std::pair<int, std::string>>`, which
  allows to simply write `MyMap::const_iterator` and significantly improves code
  readability;

- in modern C++ you can also avoid using iterators explicitly in many cases.

This brings us to another important point: type is documentation which is
automatically verified and enforced by compiler. Type omission makes the code
more difficult to comprehend, e.g., consider an example from
<http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>

    auto hello = "Hello!"s; // a std::string
    auto world = "world"; // a C-style string

C++ is a language where the difference between `std::string` and C-style string
can be important, so we should avoid obscuring such details.

Package conventions
===================

Naming
------

- Package naming convention can be surprisingly difficult to agree on and can
  hardly be universal:

    - Should a robot name precede package type or otherwise, i.e., `robot_model`
      or `model_robot`?

    - If version control system supports grouping of packages in folders, such
      as `GitLab`, should the groups be included in the package name?

    - Should the name indicate a programming language used in package? Is it
      possible that that would be necessary for disambiguation?

- ROS package naming conventions are a good starting point
  <http://www.ros.org/reps/rep-0144.html>.

Layout
------

- ROS1 catkin package template example
  <https://github.com/asherikov/ccws/tree/master/pkg_template/catkin>

- Public headers must be located in `include/mypackage/` subfolder. Non-public
  headers should be kept together with source files in `src`.

Installation paths
------------------

- Read man pages for `hier` and `systemd-unit`, conventions may vary slightly
  depending on distribution.

- Don’t put your headers in the root of `**/include/` folders, it is like peeing
  in public. Always use package specific subdirectories.

Networking
==========

TCP
---

Sometimes `TCP` may seem to be a good choice for critical real-time data
transfer since it doesn’t lose data. However, a better design choice is to
embrace possible data loss and build a system that can tolerate it. Moreover,
there are important implementation details in `TCP` that need to be kept in
mind:

- Bufferization logic is more complicated and slower than in UDP, but, more
  importantly, transferring buffer may accumulate small independent packets to
  be sent in bursts. For example, if an application sends small telemetry
  packets such as GPS coordinates or encoder data at high frequency, they may
  stick together and arrive to the receiving application in small groups at
  lower frequency. See `TCP_NODELAY` in `man tcp` for a workaround.

- Networking stack takes measures to prevent conflicts between TCP sessions, in
  particular a side that initiated session termination blocks corresponding
  socket in `TIME_WAIT` state
  <https://serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html>
  Default timeout on Linux for this state is 60 second, i.e., under certain
  circumstances you won’t be able to reestablish a TCP connection for a whole
  minute, which is more than enough to be fatal in robotic applications.
  `SO_REUSEADDR` socket option may be helpful for alleviating this issue.

Fixing robot models
===================

Inertia
-------

Ironically some commercial CAD systems incorrectly export inertia matrices of
rigid bodies to `URDF`, so it is a good idea to verify them. One way to achieve
this is to perform eigendecomposition of the matrix
<https://en.wikipedia.org/wiki/Moment_of_inertia#Principal_axes> to obtain
principal axes and moments of inertia, which can be used to specify orientation
and extent of a rectangular cuboid. Obtained cuboid should roughly match visual
representation of a rigid body.

Other
=====

ROS
---

- Try to follow standard ROS conventions listed at
  <http://www.ros.org/reps/rep-0000.html>

- Avoid scripting in launch files: `if` and `unless` attributes, python code
  injection.

Telemetry
---------

- Collect telemetry from everywhere: robots, servers, workstations, etc. It pays
  off in long term.

- Standard Linux / UNIX telemetry tools are usually not meant to handle data at
  sub-second resolution, so you are going to need a custom solution for robotic
  applications.

- Must have: `grafana`, `PlotJuggler`.
