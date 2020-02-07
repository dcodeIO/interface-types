# Linking

This explainer introduces the WebAssembly Linking proposal. The explainer
starts by describing a set of motivating use cases that have appeared
in the WebAssembly ecosystem followed by a hypothetical core WebAssembly
feature, *first-class modules*, which could address these use cases.
Limitations of first-class modules (viz. dependency on GC and dynamism) are
explained and then addressed by lifting first-class modules into the
*adapter functions* introduced by the [Interface Types] proposal. The resulting
proposal enables WebAssembly modules to control how their dependencies'
instances are instantiated and linked in a manner that is host-independent,
predictably optimizable and without implicit dependency on garbage collection.

While the Linking proposal started as an extension of Interface Types, in its
current form, the proposal suggests that Interface Types may be more naturally
defined as an extension of Linking. In particular, the Linking proposal
provides a more simple, orthogonal mechanism for describing how a core wasm
module's imports and exports are *linked* to the adapter functions of a
containing *interface module*, with the Interface Types proposal only adding
new value types (like `string` and `array`) and their associated instructions.

The explainer is organized as follows:
1. [Motivation](#motivation)
    1. [Command modules](#command-modules)
    2. [Dynamic shared-everything linking](#dynamic-shared-everything-linking)
    3. [Link-time virtualization](#link-time-virtualization)
2. [First-class modules](#first-class-modules)
    1. [Module and instance types](#module-and-instance-types)
    2. [Single-level imports](#single-level-imports)
    3. [Module and instance imports](#module-and-instance-imports)
    4. [Module and instance references](#module-and-instance-references)
    5. [Nested modules](#nested-modules)
    6. [Module and instance exports](#module-and-instance-exports)
    7. [Constructors](#constructors)
    8. [Determinate module import linking](#determinate-module-import-linking)
3. [Use cases revisited](#use-cases-revisited)
    1. [Command modules revisited](#command-modules-revisited)
    2. [Dynamic shared-everything linking revisited](#dynamic-shared-everything-linking-revisited)
    3. [Link-time virtualization revisited](#link-time-virtualization-revisited)
4. [Second-class modules](#second-class-modules)
    1. [The static def-use property of adapter functions](#the-static-def-us-property-of-adapter-functions)
    2. [Naively lifting first-class modules into Interface Types](#naively-lifting-first-class-modules-into-interface-types)
    3. [Rebasing Interface Types onto Interface Modules](#interface-modules)
    4. [When can a module be destroyed?](#when-can-a-module-be-destroyed)


## Motivation

To realize the potential of WebAssembly, it should be possible to build
reusable, composable WebAssembly components without requiring all developers to
use the same language, toolchain or memory representation (where a "component"
is one or more tightly-coupled wasm modules, in a sense to be more-precisely
defined [below](#dynamic-shared-everything-linking)). One half of this
requirement is satisfied by adopting a [shared-nothing] model wherein each
component encapsulates its own state (memory, table and globals) using
[Interface Types] and [Abstract Types] to pass values and references,
respectively, across component interface boundaries.

The remaining problem to be solved is to enable WebAssembly components to be
**linked** together in a language-, toolchain- and host-agnostic manner.
Currently in WebAssembly, if a wasm module contains an import:
```
(import "m" "f" (func ...))
```
the way in which the module name `m` and the field name `f` are resolved to a
function is entirely host-defined. Even within a single host there can be
multiple modes of resolution. For example, in a JS embedding, a module can
either be instantiated by the [JS API] or [ESM-integration]. Given all this
ambiguity, non-Web WebAssembly hosts sometimes simply don't support
wasm-to-wasm imports at all, only allowing imports from a fixed set of built-in
modules supplied by the host.

To standardize host-independent linking, obviously the [JS API] is ruled out,
but perhaps a wasm-only subset of [ESM-integration] could be defined, such
that a collection of N wasm modules could be equivalently loaded in both JS and
non-JS hosts? Such a solution would be attractive in allowing WebAssembly
modules to be seamlessly integrated into the largest existing package
ecosystem. Unfortunately, the [ESM-integration] model is not expressive enough
to support the following 3 important use cases:

### Command modules

If a C program with a `main()` function is compiled to a wasm module,
and the `main()` function is exported (directly, or indirectly via some
exported wrapper function), the natural way to instantiate the module is once
each time `main()` is called. Otherwise, if a single instance is created and
has its `main()` called multiple times, fundamental C program assumptions may
be broken. In general, programs are often architected to either be run as
`main()`-style *commands* (run from a command-line or shell script) or 
[daemons]/services (run in the background and responding to a host- or
user-defined sequence of requests) and it can be hard to convert between the
two architectures.

Moreover, creating and destroying short-lived instances more-eagerly releases
memory to the system, reduces fragmentation and avoids a class of leaks and
bugs. This is essentially the motivation for the "many small Unix processes"
style of program design, but with lower overhead due to wasm instances being
lighter-weight than OS processes.

Lastly, command modules should themselves be able to import and call other
commands modules, similar to how POSIX processes can recursively spawn and
wait on child processes (via `system()` et al). The desired instantiation
behavior over time is illustrated below with `clang` and `make` as command
modules:

<p align="center"><img src="./command.svg" width="850"></p>

Here, the `call`s and `return`s refer to calling an export of the instance
pointed to by the call. As shown by the "width" of the instance on the x
axis, execution time, the instance is thus created right before the export
call and destroyed right after.

*This use case rules out any linking solution that forces instantiation to happen
eagerly or at most once for a given module.*

### Dynamic shared-everything linking

The principle behind the [shared-nothing] model is that, since it's difficult
and error prone to get unrelated toolchains and languages to share low-level
state, like linear memory and tables, this state should be encapsulated by a
module by default. However, many C/C++/Rust libraries cannot be factored to
share nothing, either due to fundamental dependencies on sharing memory in the
interface or practical reasons like performance or difficulty refactoring. By
default, such libraries are currently [statically linked] into a containing wasm
module. Statically linking everything leads to code duplication which is
classically addressed in native code environments by [dynamic linking]. In the
context of wasm, we'll call this "dynamic *shared-everything* linking" to
distinguish it from dynamic *shared-nothing* linking.

A challenge with dynamic shared-everything linking is to control which
instances share which memory. In particular, it should be possible to write a
"main" module (the analogue of a native executable) that creates its own private
memory, explicitly shares it with N other "shared" modules (the analogue of
native `.dll`/`.so`s), but encapsulates its memory from *all other* modules so
that the use of shared-everything linking is kept an implementation detail from
the point of view of the main module's clients. Thus, we want a two-level
hierarchy, where multiple same-language/ABI wasm modules are composed into a
single logical unit whose public interface is determined by its main module.
For convenience (and until a better term emerges), this explainer will refer to
this "single logical unit" as a **component**.

For example, consider the diagram below showing a component `viz` that depends
on two components `zip` and `img` which happen to share library code (`libc` and
`libm`). `viz` shouldn't need to know the implementation language of `zip` and
`img`, much less that they share library code; in the future, if `zip` uses
different libraries or a different version of the same library, `viz` shouldn't
have to know or care. This high-level requirement boils down to a low-level
requirement that each component should get a *new* instance of each shared
library in the component. Thus, the static dependency graph shown on the left
should produce the dynamic instance graph shown on the right:

<p align="center"><img src="./shared-everything.svg" width="650"></p>

The orange rectangles on the right show the (dynamic) component instances
that are created at runtime from the orange (static) components on the left.
Note that `viz`, `zip` and `img` are components, but `libc` and `libm` are not;
they are just wasm modules that implement a particular toolchain's ABI. Thus,
the interfaces of `libc` and `libm` will use `i32` offsets into a shared linear
memory while the interfaces of `viz`, `zip` and `img` will use interface and
abstract types.

While the preceding discussion has focused on sharing linear memory for
languages like C/C++ and Rust, there will still be a need for shared-everything
linking with GC languages even after the [GC proposal]. Instead of sharing linear
memory, GC languages will need to share metadata, vtables, runtime type tags,
runtime support machinery, etc. Thus, the two-level module/component hierarchy
is essential to the whole wasm ecosystem, now and in the future.

*This use case rules out any linking solution where main modules are not able
to explicitly create and link together private instances of a language runtime
and side modules.*

### Link-time virtualization

When using a first-class instantiation API like the JS API's [`instantiate`],
the imports of the being-instantiated module appear as explicit parameters
supplied by the calling code. This has several useful properties that any wasm
linking proposal should preserve:

First, it supports applications that wish to adhere to the 
[Principle of Least Authority], passing modules only the imports necessary to do
their job. Some Web applications are already relying on this property to
sandbox untrusted wasm plugins.

Second, it enables client modules to fully or partially [virtualize] the imports
of their dependencies without extraordinary challenge or performance overhead.
For example, if a module imports a set of file operations and a client wants to
reuse this module on a system without builtin file I/O, the file operations and
handles can be virtualized with an in-memory implementation. In general, if
virtualization is well-supported and efficient, software reusability and
composability are increased. Virtualization also enhances the ability of
developers to implement the Principle of Least Authority, by allowing a client
module to not only pass *subsets* of capabilities, but also pass [attenuated]
capabilities, where additional *runtime* policies are applied to limit granted
authority.

While it is possible to perform virtualization at run-time, e.g., passing
function references for all virtualizable operations, this approach would be
problematic as a basis for a highly-virtualizable ecosystem since it would
require modules to intentionally opt into virtualization by choosing to receive
first-class function references instead of using function imports. In the limit,
to provide maximum flexibility to client code, toolchains would need to avoid
*all* use of imports, effectively annulling a core part of wasm. Because of
their more-static nature, imports are inherently more efficient and optimizable
than first-class function references, so this would also have a negative
performance impact.

To avoid the problems of run-time virtualization, wasm linking should therefore
enable **link-time virtualization**, such that a parent module can specify
all the imports of its dependencies (without any explicit opt-in on the part
of those dependencies), just as with the JS API. As an example, it should be
possible to use wasm linking to achieve the following instance graph:

<p align="center"><img src="./virtualization.svg" width="400"></p>

Here, despite the fact that `parent.wasm` and `child.wasm` both import the same
`wasi:file`, the parent is able to create an anntenuated version of `wasi:file`
which it passes to the child. Thus, `wasi:file` doesn't identify a single
global implementation; rather it names an *interface* which can be variously
implemented. Indeed, even `parent.wasm` doesn't know whether it received the
host's native implementation of `wasi:file` or some other implementation
supplied by another module that imports `parent.wasm` (hence the `? instance` in
the diagram). There may not even *be* a "native" implementation of `wasi:file`.
Crtically, though, there is no way for `child.wasm` to bypass `parent.wasm` in
its request for `wasi:file`.

*This use case rules out any linking solution where a module is not able to
explicitly create and supply the imports of its dependencies.*


## First-class modules

Based on the preceding use cases, it's clear something more expressive than
ESM-integration is required. While the JS API's first-class modules, instances
and instantiation functions could enable all 3 use cases, wasm of course cannot
take a universal dependency on JS. But could something similarly powerful be
added to core wasm instead? Indeed, such a feature has been under consideration
since the beginning of WebAssembly standardization. This section outlines how
such feature, which will be referred to as "first-class modules", could work
and satisfy the above use cases.

First-class modules do have some downsides, though, which will be discussed in
a subsequent section. However, these downsides can be addressed by lifting
first-class modules into the adapter layer introduced by Interface Types, which
will be discussed thereafter. In this section, though, we exclusively consider
first-class modules in core wasm.


### Module and instance types

While the feature is named "first-class modules", there are actually two
new first-class things: modules and instances. Like every other first-class
thing wasm manipulates, modules and instances have a type. The [Module Types]
proposal conveniently already aims to define the textual and abstract syntax for
module and instance types, so we reuse what is proposed there here.

Conceptually, a wasm module (that has been [decoded] or [parsed] and then
[validated]) is just a *function* that constructs module instances given a set
of parameters and so a wasm module *type* is conceptually just a *function type*.
Concretely, however, modules are defined in terms of imports and exports, where
the imports serve as the parameters and the exports describe the resulting
instance. Thus, instead of writing a module type as literally as a function
type, module types are written in terms of imports and exports and instance
types are written in terms of exports.

For example, the module:
```wasm
(module
  (memory (import "mod" "mem") 1)
  (func (import "mod" "f") (param i32) (result i32)))
  (func (export "run") (param i32 i32) (result i32) (i32.add (local.get 0) (local.get 1)))
  (table funcref 1)
)
```
has the module type:
```wasm
(module
  (memory (import "mod" "mem") 1)
  (func (import "mod" "f") (param i32) (result i32))
  (func (export "run") (param i32 i32) (result i32))
)
```
and its constructed instances have the instance type:
```wasm
(instance
  (func (export "run") (param i32 i32) (result i32))
)
```

The above module and instance types are using syntactic sugar
(defined as [abbreviations in the text format][import-abbrev]). The equivalent
desugared form, extended slightly to allow exports to be symmetric, is:
```wasm
(module
  (import "mod" "mem" (memory 1))
  (import "mod" "f" (func (param i32) (result i32)))
  (export "run" (func (param i32 i32) (result i32)))
)
(instance
  (export "run" (func (param i32 i32) (result i32)))
)
```
This desugared form will be used throughout this explainer since it will
more-naturally extend to imports/exports of interface-typed values
later on.


### Single-level imports

Before we can introduce module and instance imports, there is another change to
core wasm which will be convenient: *single-level imports*. A single-level
import has only one string, instead of the usual two:
```wasm
(import "x" (func))  ;; becomes legal
```

While not semantically required in the core wasm spec, the general idea of the
existing two-level imports is that the first name is the name of a parameter
which *implicitly* takes an instance and the second name names a field of that
instance. However, as we'll see next, when importing an instance *explicitly*,
only one name makes sense, since the field name is already part of the
instance. Thus, requiring instance imports to be two-level would end up
introducing unnecessary baggage in the common case.

While a special case could be made for instance imports, as we'll see
later when looking at the `module.instantiate` instruction, single-level
imports make sense for *all* kinds of imports. Thus, as a general extension to
wasm, imports would be allowed to provide only a single import name.

From a core wasm perspective, import names are insignificant; the
[`module_instantiate`] function just takes an ordered sequence of values. It's
only the *host embedding* that inspects a module's import strings to determine
how to materialize the sequence of import values. Thus, each host embedding
will need to determine how to interpret single-name imports.

In the case of the [JS API], the obvious interpretation of:
```wasm
(module
  (import "x" "y" (func))
  (import "z" (func))
)
```
is to expect an import object of the form:
```js
const importObj = {
  x: {
    y: () => {}
  }
  z: () => {}
};
WebAssembly.instantiate(module, importObj);
```
As a nice side effect, this allows [JS API]-using toolchains that have no
need of a two-level namespace, like Emscripten, to avoid synthesizing
otherwise-superfluous first-level names like `env`.

For [ESM-integration], single-name imports of instance/module type (shown next)
have a natural interpretation. For any other import kind, it would be possible
for a single-name import to map to the [default export], which would actually
close a slight expressivity gap of current [ESM-integration].


### Module and instance imports

The first step toward first-class modules is to allow them to be *imported*.
As with other wasm imports, a module or instance import declares the type of
the imported thing, which means embedding a module or instance type inside
an `(import)` statement.

Starting with an example instance import:
```wasm
(module
  (import "i" (instance
    (export "x" (func))
    (export "y" (func))
  ))
)
```
we can see why a single-level import makes sense. This import is essentially
equivalent to the MVP wasm module:
```wasm
(module
  (import "i" "x" (func))
  (import "i" "y" (func))
)
```
where the first-level name has been factored out. Notice that the function
*imports* in the MVP wasm module turn into function *exports* of an
*imported* instance.

While not strictly necessary, for convenience, size and performance, the
individual function, memory, table, global fields of imported instances would
be added into their respective [index spaces]. This would allow, e.g., calls to
imported functions or stores to imported memories to work just like they do now.

Module imports are written symmetrically. For example:
```wasm
(module
  (import "m" (module
    (import "a" (func))
    (export "x" (func))
    (export "y" (func))
  ))
)
```
Here we define a module which imports another module. The difference with
an instance import is that the containing module can control how the imported
module is instantiated. Thus, the containing module can create 0, 1 or N
instances of this imported module, supplying a different `a` import and
receiving distinct `x` and `y` exports each time. In contrast, the prior
instance import is given exactly one instance, `x` and `y` at instantiation
time.

But how do we actually go about instantiating an imported module?


### Module and instance references

Symmetric to the [function references] proposal, module and instance types can
used in reference type that can be passed around as first-class values.
To do this, the module or instance type needs to be explicitly defined in the
type section, so that it can be referenced (by type index) in a `(ref $T)` type:
```wasm
(module
  (type $I (instance
    (export "run" (func (param i32) (result i32)))
  ))
  (func $callRun (param (ref $I))
    ...
  )
)
```

Given these new module and instance reference types, we can define 3
new instructions:

First, symmetric to [`ref.func`] in the function references proposal, there
are two new instructions for creating first-class module and instance references:
* `ref.instance $i : [] -> [(ref (instance $I))]`
  * iff `$i : (instance $I)` in the instance index space
* `ref.module $m : [] -> [(ref (module $M))]`
  * iff `$m : (module $M)` in the module index space

Second, given an instance reference, `instance.export` extracts a first-class
reference to one of its fields, identified statically by an index into the
ordered list of exports declared by the instance type:
* `instance.export <export-string> : [(ref (instance $I))] -> [(ref $T)]`
  * iff `$I` has an export named `<export-string>` with type `$T`

For example, we can now fill in the body of the above `$callRun` function:
```wasm
(module
  (type $I (instance
    (export "run" (func (param i32) (result i32)))
  ))
  (func $callRun (param $i (ref $I)) (result i32)
    local.get $i
    instance.export "run"
    i32.const 42
    call_ref
  )
)
```

Lastly, and central to the whole first-class modules feature, is the
`module.instantiate` instruction:
* `module.instantiate : [(ref (module M)) Tᵢⁿ] -> [(ref (instance I))]`
  * iff `M` has `n` imports matching the operand types `Tᵢ` and exports matching `I`

Similar to direct function calls, there is a "direct" form of
`module.instantiate` that replaces the first module reference operand with a
static module index:
* `module.instantiate $m : [Tᵢⁿ] -> [(ref (instance I))]`
  * iff `$m : (module M)` in the module index space

To see `module.instantiate` in action, this example imports a module once and
instantiates it with two different imported functions:
```wasm
(module
  (import "m" (module $m
    (import "i" (func))
    (export "run" (func))
  ))
  (func $f1 ...)
  (func $f2 ...)
  (func $runWithBoth
    (call_ref
      (instance.export "run"
        (module.instantiate $m
          (ref.func $f1))))
    (call_ref
      (instance.export "run"
        (module.instantiate $m
          (ref.func $f2))))
  )
)
```
In this example, the module `$m` imports an individual function, but whole
instances can be imported just as well:
```wasm
(module
  (type $I (instance
    ... exports
  ))
  (import "x" (instance $x (type $I)))
  (import "m" (module $m
    (import "i" (instance (type $I)))
    (export "run" (func))
  ))
  (func $runWithImport
    (call_ref
      (instance.export "run"
        (module.instantiate $m
          ref.instance $x)))
  )
)
```
Here, the `(type $I)` in the `import` statements is a [type use], allowing us to
reuse the type definition of `$I`.


### Nested modules

A nested module is simply a module written inside another module. Just as
function imports and definitions form a single function index space, module
imports and nested modules form a single module index space, so they can be
uniformly instantiated via `module.instantiate`. For example:
```wasm
(module
  (module $child
    (func (export "hi")
      ...
    )
  )
  (func (export "newChild") (result (ref (instance (export "hi" (func)))))
    module.instantiate $child
  )
)
```

Unlike most source-language nested functions/classes, nested modules have no
special access to their containing parent modules' state. As a syntactic
convenience, nested modules are allowed to use types (via `$identifer`) defined
in the parent module. Whether the binary format similarly allows type reuse
is TBD; type reuse could be a modest size savings, but binary decoding would be
simplified if each module was completely independent. Other than type reuse,
though, the parent and child modules' internals are completely encapsulated
from each other.

Because of this encapsulation, it is 100% equivalent to transform a module
import that is known to resolve to a module `X` into a nested module `X`
without changing either the text or binary format of `X`. The reverse direction
is also possible, with the exception that parent type definitions may need to
be duplicated into the child. This is an important and unique property of
module imports / nested modules owing to the fact that a wasm module is a
stateless function defined entirely in terms of its binary/text format.

Is there also a symmetric concept of "nested instance"? In theory, this could
be accomplished by allowing `(instance expr)` statements, where `expr` was a
[constant initializer expression] containing `module.instantiate`. However,
this would be complex to define and fairly limited in practice compared to the
second-class modules introduced later, so no nested instance concept is
proposed here.


### Module and instance exports

Modules and instances can also be (re)exported, just like everything else:
```wasm
(module
  (module $m1 ...)
  (import "m" (module $m2 ...))
  (import "i" (instance $i ...))
  (export "1" (module $m1))
  (export "2" (module $m2))
  (export "3" (instance $i))
)
```
Because of the lack of "nested instances", as described in the previous
section, instance exports can only be re-exports of instance imports.


### Constructors

One problem that we'll see come up below when the shared-everything linking and
link-time virtualiztion use cases are [revisited](#use-cases-revisited) is that
if a parent module P has a dependency on module D, then P usually wants to
instantiate D first, so that P can import D. However, if P can only call
`module.instantiate` from a function running from within a P instance, there is
an obvious ordering conflict. One option would be for P to move all of its code
into a nested module P', leaving P as a shell that for P'. Then the order of
instantiation could be: P &rarr; D &rarr; P'. However, this would mean that P's
exports would need to be thunks that make indirect calls to the real exports
of P', which is unfortunate.

What we'd really is the ability for P to run a special *constructor* function
*before* it is instantiated. This constructor function could mostly be a normal
wasm function, but validated in a stripped down environment with no memory,
tables, functions, etc (essentially, only types and imports). In this
restricted environment, the constructor would be able to call
`module.instantiate` on P's imported or nested modules and then
`module.instantiate` P itself. The constructor would also be able to call
initializtion exports (e.g., [`DllMain`]) on dependencies and P according to
whatever linking ABI.

TODO:
* how to write: fixed return type
* changes the module type according to ctor params
* `module.instantiate-self`
* relation to `start` function


### Determinate module import linking

To close out the first-class module feature, there is one more problem to be
solved: module imports force all private dependencies of a module to be imports
in its public module type, so that they much be explicitly supplied by clients.

To see the problem, let's look at the dependency diagram from the
[dynamic shared-everything linking use case](#dynamic-shared-everything-linking):

<p align="center"><img src="./shared-everything.svg" width="650"></p>

Let's assume for the moment that each of the dependency arrows between `.wasm`
modules become module imports in the client module (we'll see why when
revisiting the dynamic-shared-everything use case [below](#dynamic-shared-everything-linking-revisited)).
This means that, e.g., the module type of `zip.wasm` and `img.wasm` will include:
```wasm
(module
  (import "libc" (module ...))
  (import "libm" (module ...))
  ...
)
```
which means that the module type of `viz.wasm` will need to include:
```wasm
(module
  (import "zip" (module
    (import "libc" (module ...))
    (import "libm" (module ...))
    ...
  )
  (import "img" (module
    (import "libc" (module ...))
    (import "libm" (module ...))
    ...
  )
  ...
)
```
When `viz.wasm` calls `module.instantiate` to instantiate its imported `zip`
and `img` modules, `viz` will need to get `libc` and `libm` from *somewhere*.
One option is to embed `libc.wasm` and `libm.wasm` into `viz.wasm` as nested
modules, but now these shared libraries can't be shared with the broader app,
which may contain other uses of these same shared libraries. To enable maximal
sharing, `viz` needs to import `libc` and `libm` *itself*. But then we'd have a
design where ultimately the root module of a module dependency DAG ends up
importing *every single transitive dependency*!

To resolve this tension we need a mechanism that has qualities of both module
imports and nested modules: something that allows sharing of dependencies but
still keeps dependencies an encapsulated detail, not present in the module type.

The solution proposed here is to introduce a new **determinate module import
linking** phase in the WebAssembly specification.

First, we need categorize module imports into two formally-defined cases:
* **determinate** module imports, which refer to a specific module; and
* **indeterminate** module imports, which behave as above: they are explicit
  arguments supplied by the client and only required to have a matching
  module type.

To formally discriminate these two cases, we specify that any *single-level*
import whose string is a [valid URL] is a determinate import and all other
imports are indeterminate. Since single-level imports are new, this definition
ensures that no existing wasm module contains determinate imports, which is
necessary to ensure that this new phase is backwards compatible.

Next, the wasm spec's existing [`module_instantiate`] entry point is extended to
have an optional `modulemap` argument:
* `module_instantiate(store, module, externval*, modulemap?) : (store, moduleinst | error)`

where `modulemap` is a map from URLs to [module]s. As a precondition, the host
must ensure that:
1. all determinate import strings (URLs) in `module` and the module values of
   `modulemap` are present as keys of `modulemap`
2. there are no cycles in `modulemap`

The "determinate module import linking" phase is then added to the beginning of
`module_instantiate`, transforming the incoming `module` as defined by a new
`module_link` function with signature:
* `module_link(module, modulemap) : (module | error)`

The `module_link` function recursively replaces determinate module imports in
`module` with nested modules according to `modulemap` until there are no more
determinate module imports. Before each substitution, the new nested module is
checked to match the module import's declared type, ensuring that if
`module_link` does not fail, the resulting module is valid.

Since `module_link` is always performed first and since it removes all
determinate imports before instantiation, determinate module imports
effectively cease to be part of the module type from the perspective of the
`module.instantiate` instruction. Thus, determinate module imports do go into a
module's type, thereby achieving the initial goal of encapsulating private
dependencies.

During `module_link`, a host implementation is expected to reuse module code
whenever a particular URL is used more than once instead of blindly substituting
copies. However, it would also be possible, albeit more complex, to specify an
observably-equivalent version of `module_link` that explicitly avoids
duplication in the new module by hoisting nested modules to a common ancestor in
the dependency DAG. This "optimizing" transformation could then be literally
implemented by bundling/deployment tools (such as webpack) as an optimization,
allowing N linked modules to be efficiently collapsed down to 1.

Finally, to be clear, the resolution of URLs to modules is still host-defined;
the wasm spec is only extended to specify what happens *after* resolution is
complete for an *entire* module DAG. That being said, it is possible for a
toolchain to start making some portable assumptions about how particular URLs
(especially relative URLs) work.


## Use cases revisited

With first-class modules introduced, we can now revisit the motivating use cases
identified above and see how they can be addressed.


### Command modules revisited

TODO: fill in words; but the shape of the solution is:

```wasm
(module
  (type $WasiFileInstance (instance
    ...
  ))
  (import "wasi:file" (instance $file
    (type $WasiFileInstance)
  ))
  (module $core
    (import "wasi:file" (type $WasiFileInstance))
    (func (export "main") (param i32 i32) (result i32)
      ...
    )
  )
  (func (export "run") (param i32 i32) (result i32)
    (call_ref
      (instance.export "main"
        (module.instantiate $core
          (ref.instance $file)))
      (local.get 0)
      (local.get 1))
  )
)
```

* note that `wasi:file` is imported once by the outer module and used to instantiate `$core` each time the export is called


### Dynamic shared-everything linking revisited

TODO: fill in words; but the shape of the solution is:

```wasm
// libc.wasm
(module
  (memory (export "memory"))
  (func (export "malloc") (param i32) (result i32) ...)
  ...
)
```

* `libc.wasm` is shipped with the compiler as part of the sysroot

```wasm
// libm.wasm
(module
  (type $LibcInstance (instance
    (memory (export "memory"))
    (func (export "malloc") (param i32) (result i32))
    ...
  ))
  (import "libc" (type $LibcInstance))

  (func (export "sin") (param f64) (result f64) ...)
)
```

* note libc is imported as an instance, allowing a client to choose which libc instance
* this would be the convention for *all* `-shared` libraries
* just like functions of imported instances, the memory of an imported instance
  gets added to the memory index space, becoming, in this case, the default memory (memory 0)

```wasm
// zip.wasm
(module
  (type $LibcInstance (instance
    (memory (export "memory"))
    (func (export "malloc") (param i32) (result i32))
    ...
  ))
  (import "./libc.wasm" (module $Libc
    (exports $LibcInstance)
  ))

  (type $LibmInstance (instance
    (func (export "sin") (param f64) (result f64))
  ))
  (import "./libm.wasm" (module $libm
    (import "libc" (type $LibcInstance))
    (exports $LibmInstance)
  ))

  (import "libc" (type $LibcInstance))
  (import "libm" (type $LibmInstance))
  (func (export "run") (param i32 i32) (result i32 i32)
    ...
  )

  (ctor
    module.instantiate $Libc
    (let (local $libc (ref $LibcInstance))
      (module.instantiate $libm (local.get $libc))
      (let (local $libm (ref $LibmInstance))
        module.instantiate $main (local.get $libc) (local.get $libm)
        return
      )
    )
  )
)
```

* note we're using a syntactic sugar `(exports $FooInstance)` to inject the exports of an instance type into a module type, to avoid repeating it all. there may be better way to do this
* note that, as the main module, `zip.wasm` gets to choose *which* libc/libm via determinate module imports
* note that, because the `ctor` has no params, the resulting module type has no
  imports, only the one exported `run` as we see below:

```wasm
// viz.wasm
(module
  (import "./zip.wasm" (module
    (func (export "run") (param i32 i32) (result i32 i32))
  ))
  (import "./img.wasm" (module
    (func (export "run") (param i32 i32) (result i32 i32))
  ))
  ...
)
```

* note no mention of libc/libm; they are encapsulated by `zip.wasm` and `viz.wasm`
* whether or not `zip.wasm` and `img.wasm` share `libc.wasm` is determined by if their determinate module URLs resolve to the same module
* a build-time tool is responsible for writing the URLs in to the module imports
* here, the build-time tool has put everything into the same directory and pointed both `zip.wasm` and `viz.wasm` at the same `libc.wasm`
* it is the job of a higher-level build-time tool (e.g., make system, IDE or package manager) to unify or duplicate (based on semver or just byte equality)


### Link-time virtualization revisited

TODO: fill in words; but the shape of the solution is:

```wasm
// child.wasm
(module
  (type $WasiFileInstance (instance
    (func (export "write") ...)
  ))
  (import "wasi:file" (type $WasiFileInstance))
  
  (func (export "play") ...)
)
```

* `child.wasm` imports `wasi:file` with an instance import (it has no other choice)

```wasm
// attenuate.wasm
(module
  (type $WasiFileInstance (instance
    (func (export "write") ...)
  ))
  (import "wasi:file" (type $WasiFileInstance))

  (func (export "write") ...)
)
```

* `attenuate.wasm` imports `wasi:file` with an instance import and then defines exports that match
  the `wasi:file` instance type
* thus `attenuate.wasm` is a function mapping `wasi:file` instances to new (attenuated) `wasi:file` instances
* how this attenuation policy is passed to `attenuate.wasm` is in the `...`
  

```wasm
// parent.wasm
(module
  (type $WasiFileInstance (instance
    (func (export "write") ...)
  ))
  (import "wasi:file" (instance $file
    (type $WasiFileInstance)
  ))

  (import "./attenuate.wasm" (module $attenuate
    (import "wasi:file" (instance $WasiFileInstance))
    (exports $WasiFileInstance)
  ))

  (type $ChildInstance (instance
    (func (export "play") ...)
  ))                         
  (import "./child.wasm" (module $child
    (import "wasi:file" (instance $WasiFileInstance))
    (exports $ChildInstance)
  ))

  (import "child" (instance
    (type $ChildInstance)
  ))

  (ctor
    ref.instance $file
    module.instantiate $attenuate
    module.instantiate $child
    module.instantiate_self
  )
)
```

* by importing `child.wasm` as a module, `parent.wasm` has complete control
  over how it is instantiated and what gets passed in
* note that determinate module import linking doesn't violate the parent's
  control over the child: the child can only import code that it could've
  written itself (in a nested module). Importing a module never gives you
  a capability.


## Second-class modules

While the preceding section shows how the use cases can be functionally solved
by first-class modules, there remain several practical problems with first-class
modules:

First, first-class modules fundamentally depend on GC because, by containing
mutable reference cells (globals and tables), wasm instances can participate in
arbitrary cyclic graphs. As a general rule, WebAssembly standardization avoids
creating any hard dependencies on GC for anything outside the [GC proposal].
In particular, one of our motivating use cases is shared-everything dynamic
linking for C/C++, where there is otherwise no need for GC.

The second problem is that the purely runtime nature of first-class module
linking prevents various desirable optimizations and compilation techniques that
need to know the linking structure ahead-of-time. In particular, it should be
possible to reliably compile a linked set of modules into a single,
statically-linked, optimized blob of machine code which includes the compiled
interface adapter trampolines of [Interface Types].

Lastly, we need to say how Interface Types can be used with linking
*at all*, given that first-class modules are a core wasm feature and
Interface Types are designed to be layered on top of core wasm. 

Ultimately, the solution to all three boils down to the same thing: lifting
first-class modules into the second-class context of interface adapter
functions.


### The static def-use property of adapter functions

Interface adapter functions, as introduced by the [Interface Types] proposal
have a vaguely "declarative" property that is useful to define more precisely,
since this property is what we need for linking. While adapter functions are
composed of stack instructions, just like core wasm, the set of instructions
and validation rules ensure that **every definition of an interface value flows
into the same set of uses on every (non-trapping) execution and every use of an
interface value comes from exactly one definition**. Until a better name
appears, we call this the "static def-use property" of adapter functions.

How do the adapter function constraints ensure the static def-use property?
First, calls between adapter functions are required to be non-recursive and thus
adapter function bodies can always be statically inlined into their caller.
(Indeed, efficient [fusing] of lifting and lowering adapter instructions requires
this.) Next, adapter instructions do not include the normal core wasm control
flow instructions like `br_if` or `if`. Instead, adapter code is mostly
sequential with functional-style lifting/lowering instructions for sequences and
variants that specifically preserve the static def-use property.

Thus, if we consider instance references as *interface types* (not core wasm
types) and `module.instantiate` as an *adapter instruction* that is both a
definition (of the new instance) and a use (of the imported instances), the
static def-use property implies that there will be a static structure to any DAG
of instances produced by adapter functions. This static structure is most of
what we need to solve the problems introduced above.


### Naively lifting first-class modules into Interface Types

TODO: write

* Easy: just prefix instance/module imports, nested modules and ctors with `@interface`
* Add instance/module types only to the set of interface types, not core wasm types
* Example(s) from first-class to second-class


### Rebasing Interface Types onto Interface Modules

TODO: write

* It's kindof weird that all nested modules would go inside the interface
  adapter section. That will be 99% of the bytes of many modules.
* Ultimately, having a section that wraps the containing module in a
  containing module is weird and confusing to people.
* Initially interface adapter section just contained small annotations,
  but has now grown into practically its own mini module format with
  types section, imports, exports, functions subsections.
* With nested modules and linking, we can actually make interface adapters
  into actual real modules:

1. There are two kinds of modules core modules `(module ...)` and interface modules `(@interface module ...)`
2. Interface modules use the same structure as core modules, but:
  * Only support a subset of the sections (types, imports, exports, function, code)
  * Enriched set of Interface Types anywhere there is a core wasm type in core
    modules.
3. Interface modules are additionally allowed to:
  * Use all the first-class module features described above
  * Which means containing nested modules
  * A nested module can either be an interface module or a core module
  * Thus, a `.wasm` file forms a tree whose leaves are core modules and all
    non-leaves are interface modules
4. Interface modules use the same binary encoding as core modules, using a bit of the "version" word
   to discriminate. The goal is to reuse most of a wasm decoder/validator, with 
   extra branches to accept/reject various types/instructions/sections/... where they
   are decoded.

* From an implementation perspective, an interface module ultimately compiles
  down to trampolines for entering, exiting and connecting core wasm modules.
  An interface module does not need its own reified instance/module data structures.


### When can a module be destroyed?

TODO: write

* Host is going to instantiate a root module and keep that alive for a
  host-determined lifetime. E.g.:
  * ESM-integration will keep alive as long as global object (lifetime of tab)
  * CLI might keep alive for a single export call
  * Reverse-proxy might keep alive for only the duration of a request
* The root module's constructor (default or otherwise) will return an instance reference
* If that instance is produced by `module.instantiate(-self)`, any import is
  transitively of the same lifetime as the root instance.
* Static use-def property allows transitively reaching all imports, statically,
  so all reachable instances can be bunched into a single host-determined instance-DAG
  lifetime.
* What about an instance that is not transitively reached by the root instance?
* E.g., trivial case (instantiate -> call export)
* This is the command use case
* Can host simply destroy when the instance ref is dropped (again, a static property)?
* No: if the instance created references from an existing instance to itself
* E.g., funcref installed in imported table via elem section. Could expect for `dlopen()`
* However, host can statically detect this by analyzing the instantiated module
  type. *enumerate cases*
* Non-GC host has two choices:
  1. statically reject when type may keep alive
  2. assume has same lifetime as root instance DAG, keep alive until root instance DAG destroyed
* More conservative to do the first: could be standardized as part of Interface
  Module validation. Effectively defines commands to be those that can be immediately destroyed.


[Type section]: https://webassembly.github.io/spec/core/binary/modules.html#binary-typesec
[Type use]: https://webassembly.github.io/spec/core/text/modules.html#type-uses
[start functions]: https://webassembly.github.io/spec/core/syntax/modules.html#start-function
[decoded]: https://webassembly.github.io/spec/core/binary/index.html
[parsed]: https://webassembly.github.io/spec/core/text/index.html
[validated]: https://webassembly.github.io/spec/core/valid/index.html
[import-abbrev]: https://webassembly.github.io/spec/core/text/modules.html#id1
[index spaces]: https://webassembly.github.io/spec/core/syntax/modules.html#indices
[`module_instantiate`]: https://webassembly.github.io/spec/core/appendix/embedding.html#embed-module-instantiate
[module]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-module
[type use]: https://webassembly.github.io/spec/core/text/modules.html#type-uses
[constant initializer expression]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-global

[JS API]: https://webassembly.github.io/spec/js-api/index.html
[`instantiate`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiate
[Default Export]: https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export#Description
[ESM-integration]: https://github.com/WebAssembly/esm-integration
[HTML module loader]: https://html.spec.whatwg.org/#integration-with-the-javascript-module-system
[Valid URL]: https://url.spec.whatwg.org/#valid-url-string
[Import Object]: https://webassembly.github.io/spec/js-api/index.html#read-the-imports
[Namespace Object]: https://tc39.es/ecma262/#sec-module-namespace-objects

[Interface Types]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md
[Fusing]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md#lifting-lowering-and-laziness
[Abstract Types]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md
[Module Types]: https://github.com/WebAssembly/module-types/blob/master/proposals/module-types/Overview.md
[Multi-memory]: https://github.com/webassembly/multi-memory
[GC proposal]: https://github.com/WebAssembly/gc/blob/master/proposals/gc

[Shared-nothing]: https://github.com/WebAssembly/interface-types/blob/linking/proposals/interface-types/Explainer.md#motivation
[direct-copy]: https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md#lifting-lowering-and-laziness

[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#examples
[`ref.func`]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#functions

[daemons]: https://en.wikipedia.org/wiki/Daemon_(computing)
[Statically linked]: https://en.wikipedia.org/wiki/Static_library
[Dynamic linking]: https://en.wikipedia.org/wiki/Dynamic_loading
[Principle of Least Authority]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[ABI]: https://en.wikipedia.org/wiki/Application_binary_interface
[Virtualize]: https://en.wikipedia.org/wiki/Virtualization

[`dllexport`]: https://docs.microsoft.com/en-us/cpp/cpp/dllexport-dllimport
[`visibility("default")`]: https://gcc.gnu.org/wiki/Visibility
[Attenuated]: http://cap-lore.com/CapTheory/Patterns/Attenuation.html
