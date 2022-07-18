# Virtual Module Sources

## Synopsis

Extend the `Module` constructor such that it accepts virtual module sources:
objects that implement a protocol that is sufficient for virtualizing the
evaluation of modules in languages not anticipated by ECMA-262 or a host
implementations.

## Motivation

Hosts can add support for linking non-EcmaScript languages by creating
new kinds of ***Module Source Record***.
For example, a host could add a [[ModuleSource]] internal slot to
[`WebAssembly.Module`][web-assembly-module] and then link arbitrary WASM into
an EcmaScript module graph.

However, user code cannot emulate a host in this way.
To extend or experiment with new languages or new module-like features, user
code needs a way to emulate or *virtualize* a module source record.

Such a protocol would allow user code to integrate CommonJS, JSON, or WASM
modules on hosts that do not.
It would also open a field for experimentation with other kinds of resource
modules.

## Description

The first argument to the `Module` constructor is a source.
If that source is an object that does not have a [[Module Source]] internal
slot, we treat this object as a protocol for loading, binding, linking,
initializing, and executing the new `Module` instance.

## Interface

```ts
type VirtualModuleSource = {
  // Indicates the import and export bindings the module has
  // between its module environment record, module namespace exotic object,
  // and its dependencies.
  bindings?: Array<Binding>,

  // Executes the module if it is imported.
  // execute may return a promise, indicating that the module uses
  // the equivalent of top-level-await.
  execute?: (namespace: ModuleImportsNamespace, {
    import?: (importSpecifier: string) => Promise<ModuleExportsNamespace>,
    importMeta?: Object,
    globalThis?: Object,
  }) => void,

  // Indicates that execute needs to receive a dynamic import function
  // bound to a Module instance.
  needsImport?: boolean,

  // Indicates that initialize needs to receive an importMeta.
  needsImportMeta?: boolean,
};
```

Bindings must be one of the shapes proposed in [module source static
analysis][0], such that for each binding gets linked in a virtual module.

The `Module` constructor from [Module and ModuleSource](./0-module-and-module-source.md) extends to
accept virtual module sources instead of `ModuleSource` and reflects whatever
`source` its given as the `source` property of the returned instance.

## Examples

### JSON

This protocol allows user code to create new module source constructors.
For example, for a host that does not support JSON, this could be
accomplished with a small virtual module source.

```js
class JsonModuleSource {
  bindings = { export: 'default' };
  constructor(text) {
    // Throw SyntaxError if the source is invalid, from here.
    this.#object = JSON.parse(text);
  };
  execute(imports) {
    // Exports of multiple module instances backed by this
    // source should be referentially independent.
    imports.default = clone(this.#object);
  };
}

const source = new JsonModuleSource({ meaning: 42 });
const module = new Module(source);
const { default: { meaning } } = await import(module);
```

Asset modules of various kinds would largely follow this pattern.

### WASM

On a host that does not provide support for WebAssembly, a virtual module
source would suffice.

```js
class WasmModuleSource {
  constructor(buffer) {
    const module = new WebAssembly.Module(buffer);
    this.#imports = WebAssembly.Module.imports(module);
    this.#exports = WebAssembly.Module.exports(module);
    this.bindings = [
       ...this.#imports.map(({ module, name }) =>
         ({ import: name, from: module })),
       ...this.#exports.map(({ name }) =>
         ({ export: name })),
    ];
  };
  async execute(namespace) {
    const importObject = {};
    for (const { module, name, kind } of this.#imports) {
      importObject[name] = namespace[name];
    }
    const instance = await WebAssembly.instantiate(module, importObject);
    for (const { name } of this.#exports) {
      namespace[name] = instance[name];
    }
  };
}
```

### Pass-through module sources

This example illustrates how a virtual module source can simply
reexport another module with no special logic in an executor.

```js
import * as direct from 'real.js';
const source = { bindings: { exportAllFrom: 'real.js', as: 'real' } };
const module = new Module(source);
const { real: indirect } = await import(module);
direct === indirect; // true
```

## Dependencies

This proposal depends on [Module and ModuleSource][0] from the [Compartments
proposal](README.md) to introduce `ModuleSource`, and [ModuleSource
analysis][0] from the [Compartments proposal](README.md).

## Design

For every `Module` instance that has a virtual module source,
module import machinery constructs a real ***Module Imports Namespace***,
an exotic object that allows user code to get and set the values for each
binding.

Bindings must be one of the shapes proposed in [module source static
analysis][0], such that for each binding gets linked in a virtual module:

- `{ import: string, from: string }`

  Links the named `import` property of the ***Module Imports Namespace***
  to the same name of the `from` module's ***Module Exports Namespace***.

- `{ import: string, as: string, from: string }`

  Links the named `as` property of the ***Module Imports Namespace***
  to the `import` name of the `from` module's ***Module Exports Namespace***.

- `{ export: string }`

  Links the named `export` property of this module's ***Module Imports
  Namespace*** directly to its ***Module Exports Namespace***.

- `{ export: string, as: string}`

  Links the named `export` property of this module's ***Module Imports
  Namespace*** to the named `as` property of this module's ***Module Exports
  Namespace***.

- `{ export: string, as: string, from: string }`

  Links the named `export` property of the `from` module's ***Module Exports
  Namespace*** to the named `as` property of this module's ***Module Exports
  Namespace***, bypassing this module's ***Module Imports Namespace***
  entirely.

- `{ importAllFrom: string, as: string }`

  Links the ***Module Exports Namespace*** of the module `importAllFrom`
  to the name `as` in this module's ***Module Imports Namespace***.

- `{ exportAllFrom: string }`

  Links all of the names except `default` exported by the `exportAllFrom`
  module to this module's ***Module Exports Namespace***, bypassing this
  module's ***Module Imports Namespace*** entirely.

- `{ exportAllFrom: string, as: string }`

  Links the ***Module Exports Namespace*** of the module `exportAllFrom`
  to a property named `as` on this module's ***Module Exports Namespace***,
  bypassing this module's ***Module Imports Namespace*** entirely.

In the absence of a `bindings` property, module machinery presumes an empty
bindings array.

> Modules without bindings may still have side-effects in global scope.

Dynamic import of a `Module` with a virtual module source induces the memoized
`importHook` of the `Module` for each binding that has an `from` property.
When a `Module` instance exists for every transitive dependency, dynamic import
advances the linkage to all new namespaces, including all links between
***module [exports] namespaces*** and ***module import namespaces*** according
to each source's bindings.

In the absence of an `execute` property, dynamic import assumes an empty
execute function.

> Modules without execution behavior may still have useful export bindings from
> other modules.

Dynamic import will then execute the working set of modules according
to the existing rules of ordering, and will call `execute` for each virtual
module source, providing the linked ***Module Imports Namespace***,
a dynamic import function bound to the module instance only if the virtual
module source has a truthy `needsImport` property, and an `import.meta` object
only if the virtual module source has a `needsImportMeta` property.

## Design questions

[Should virtual module sources support emulated
JavaScript?](https://github.com/tc39/proposal-compartments/issues/70)
That will require, in some cases, separation of initialization from execution
as separate phases.
Should the virtual module source protocol support separate paths for modules
that do not require an initialization phase?

## Limitations

Module sources compiled by the `ModuleSource` constructor capture enough data
that engines can transfer them in many ways.
The internal representation of a module source can be an immutable record that
a JavaScript host can trivially share throughout an agent cluster.
Communicating JavaScript agent clusters can transfer module sources as data.

In the most naive case, a module source is just a record of the original source
and its type, which is not limited to JavaScript modules, since hosts can define
additional module source types.
For example, [`WebAssembly.Module`][web-assembly-module] might be trivially
extended to serve as a module source, by adding a [[ModuleSource]] slot
referring to a concrete ***Module Source Record***.
In this case, it is sufficient for a pair of JavaScript agent clusters
to transfer the original source and type.

In a more elaborate case, the module source does not even retain the original
text, but its compiled bytecode.
In this case, a pair of compatible JavaScript agent clusters might elect to
send and receive byte code.

Virtualized module sources are transmissible only between JavaScript agent clusters
if both the sender and receiver agree on a protocol for constructing a new
virtual module source from some serial representation.

To wit, it would not be possible for a host to transmit arbitrary virtual
module sources between agents using a general purpose algorithm like
[structured clone][structured-clone].
The protocol for communicating virtual module sources between hosts would
necessarily need to be implemented in user code.
Any general-purpose host-defined mechanism for transmitting module graphs would
necessarily fail when encountering a virtual module source.
For example, it may be possible to extend structured clone to transmit
instances of `ModuleSource` between any JavaScript hosts, or even
[`WebAssembly.Module`][web-assembly-module] when a host-defined extension is
present, but it would not be possible for this behavior to generalize to
virtual module sources.

[0]: ./0-module-and-module-source.md
[web-assembly-module]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module
[structured-clone]: https://developer.mozilla.org/en-US/docs/Web/API/structuredClone
