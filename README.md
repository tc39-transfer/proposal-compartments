# Module Loader

Load and evaluate modules.

**Stage**: 1

**Champions**:

* Bradley Farias, GoDaddy
* Mark S. Miller, Agoric
* Caridy Patiño, Salesforce
* Jean-Francois Paradis, Salesforce
* Patrick Soquet, Moddable
* Kris Kowal, Agoric
* Jack Works, Sujitech

Agoric's [SES shim][ses-shim], Sujitech's [ahead-of-time-SES][AOT-SES] and
Moddable's [XS][xs-compartments] are actively vetting this proposal
as a shim and native implementation respectively (2022).
Most activity toward advancing this proposal occurs on those projects.

## Synopsis

Provide a mechanism for evaluating modules from ECMAScript module source code
and virtualizing module loader host behaviors.

This proposal is a [SES Proposal][ses-proposal] milestone.

## Motivation

Many ECMAScript module behaviors are defined by the host.
The language needs a mechanism to allow programs running on one host to fully
emulate or virtualize the module loader behaviors of another host.

Module loader behaviors defined by hosts include:

* resolving a module import specifier to a full specifier,
* locating the source for a full module specifier,
* canonicalizing module specifiers that refer to the same module instance,
* populating the `import.meta` object.

For example, on the web we can expect a URL to be a suitable full module
specifier, and for every module specifier to correspond to a URL.
We can expect the canonicalized module specifier to be reflected as
`import.meta.url`.
In Node.js, we can also expect import specifiers that are not fully qualified
URLs to receive special treatment.
However, in Moddable's XS engine, we can expect a module specifier
to resemble a UNIX file system path and not have a corresponding URL.

We can also expect to have only one module instance per canonical module
specifier in a given Realm, and for `import(specifier)` to be idempotent
for the lifetime of a Realm.
Tools that require separate module memos are therefore compelled to create
realms either using Node.js's [VM context][vm-context] or `<iframes>` and
[content security policies][csp] rather than a lighter-weight mechanism,
and consequently suffer identity discontinuties between instances from
different realms.

Tools that will benefit from the ability to have multiple module graphs
in a single realm include:

* bundlers ([Browserify][browserify], [WebPack][webpack], [Parcel][parcel],
  &c), virtualize loading but not evaluation of module graphs and emulate other
  host environments, like a Node.js program emulating a web browser.
* import mappers ([import-map][import-map]) like bundlers need to be able to
  collect transitive dependencies according to ECMAScript language and specific
  host behaviors.
  A ECMAScript native module loader interface would expedite evolution of import map
  runtimes in JavaScript.
* hot module replacement (HMR) systems (WebPack, SnowPack, &c), which need the
  ability to instantiate new module graphs when dependencies change and the
  ability to bequeath subgraphs to new graphs.
  * Node.js [defers][node-hmr] to ECMAScript to provide a module loader
    interface to aid HMR.
* persistent testing apparatuses ([Jest][jest]), because a persistent service
  reinstantiates whole module graphs to reconstruct tests and test subjects.
  * Jest currently resorts to exploiting Node.js's [vm][vm-context] module to
    instantiate separate realms and attempts ([and
    fails][jest-ses-interaction]) to provide the illusion of a single realm by
    patching client realms with some of the intrinsics of the host realm.
* emulators ([JSDom][jsdom]) in which the emulated artifact may need a separate
  module memo from the surrounding realm.
* sub-realm sandboxes ([SES][ses] and [LavaMoat][lava-moat]) that virtualize
  evaluating guest modules and limit access to globals and built-in modules.
  This proposal prepares for the SES proposal to introduce `lockdown`, which
  isolates all evaluators, including `eval`, `Function`, and this `Loader`
  module evaluator. That proposal will introduce the concern of per-loader
  globals and hardened shared intrinsics.

Defining a module loader in the language also improves the language's ability
to evolve.
For example, a module loader interface that accounts for linking third-party
modules that are not JavaScript facilitates easier experimentation with linkage
against languages like WASM.
For another, a module loader interface allows for user space experimentation
with the notion of [import maps][import-map].

Defining a module loader in the language also provides valuable insight to the
design of every language feature that touches upon modules, and every new
module system feature adds uncertainty to the eventual inclusion of a module
loader to the language.

One such insight is that module blocks will benefit from the notion of a module
descriptor as defined by this proposal. Module blocks roughly correspond to
compiled sources and are consequently not coupled to a particular host environment.
A module descriptor is necessary to carry properties of a module not captured
in the source, like the module referrer specifier and how to populate `import.meta`.

Additionally, having a module loader interface is a prerequisite for shimming built-in
modules.

### Sketch

Below is a rough sketch of potential interfaces.

* A "module instance" consists of a "module exports namespace", a "module
  environment record", and a "static module record".
* A "module exports namespace" is an exotic object that represents the exported
  namespace of a module as already specified in ECMA 262.
  This proposal does not alter module exports namespaces.
* A "module environment record" is the scope into which a module imports names
  from other modules and exports names to other modules.
  This proposal reifies a module environment record as an exotic object that
  third-party modules can use to implement bindings.
  An `import name as alias` binding will have a property with the lexically bound alias.
  An `export name as alias` binding will have a property with the lexically bound name,
  whereas the module exports namespace will have a property with the alias.
  The environment record does not contain a property for any names that are
  imported and reexported without a lexical binding.

```ts
type ModuleExportsNamespace = Record<string, unknown>;
type ModuleEnvironmentRecord = Record<string, unknown>;

// Bindings reflect the `import` and `export` statements of a module.
type Binding =
  { import: '*' | string, as?: string, from: string } |
  { export: '*' | string, as?: string, from?: string };

// Loaders support ECMAScript modules and linkage to other kinds of modules,
// notably allowing for JSON or WASM.
// These must provide an initializer function and may declare bindings for
// imported or exported names.
// The bindings correspond to the equivalent `import` and `export` declarations
// of an ECMAScript module.
type ThirdPartyStaticModuleRecord = {
  bindings?: Array<Binding>,
  // Initializes the module if it is imported.
  // Initialize may return a promise, indicating that the module uses
  // the equivalent of top-level-await.
  // XXX The loader will leave that promise to dangle, so an eventual
  // rejection will necessarily go unhandled.
  initialize(environment: ModuleEnvironmentRecord, {
    import?: (importSpecifier: string) => Promise<ModuleExportsNamespace>,
    importMeta?: Object,
    // TODO: If module environment record also reflects the properties of
    // globalThis, we do not need to thread globalThis.
    // The module environment record must not fall through to the global
    // environment record or global contour since those capture top-level
    // declarations from Script mode eval.
    // If we add globalLexicals, to the proposal, they would need to be
    // threaded through here as well, or layered on the module environment record.
    globalThis,
  }),
  // Indicates that initialize needs to receive a dynamic import function that
  // closes over the referrer module specifier.
  needsImport: boolean,
  // Indicates that initialize needs to receive an importMeta.
  needsImportMeta: boolean,
};

// Static module records are an opaque token representing the compilation
// of a module that can be reused across multiple loaders.
interface StaticModuleRecord {
  // Static module records can be constructed from source.
  // XS allows third-party module records and source descriptors to
  // be precompiled as well.
  constructor(source: string);

  // Static module records reflect their bindings for information only.
  // Loaders use internal slots for the compiled code and bindings.
  bindings: Array<Binding>;
}

// A ModuleDescriptor captures a static module record and per-loader metadata.
type ModuleDescriptor =

  // Describes a module by referring to a *reusable* record of the compiled
  // source.
  // The static module record captures a static analysis
  // of the `import` and `export` bindings and whether the source ever utters
  // the keywords `import` or `import.meta`.
  // The loader will then asynchronously _load_ the shallow dependencies
  // of the module and memoize the promise for the
  // result of loading the module and its transitive dependencies.
  // If the loader _imports_ the module, it will generate and memoize
  // a module instance and initialize the module.
  // To initialize the module, the loader will construct an `import.meta` object.
  // If the source utters `import.meta`,
  // * The loader will construct an `importMeta` object with a null prototype.
  // * If the module descriptor has an `importMeta` property, the loader
  //   will copy the own properties of the descriptor's `importMeta` over
  //   the loader's `importMeta' using `[[Set]]`.
  // * If the loader has an `importMetaHook`, it will call
  //   `importMetaHook(fullSpecifier, importMeta)`.
  // The loader will then begin initializing the module.
  // The loader memoizes a promise for the module exports namespace
  // that will be fulfilled at a time already defined in 262 for dynamic import.
  | {
    record: StaticModuleRecord,

    // Properties to copy to the `import.meta` of the resulting module instance,
    // if the source mentions `import.meta`.
    importMeta?: Object,
  }

  // Describes a module by a *reusable* third-party static module record.
  // When the loader _loads_ the module, it will use the bindings array
  // of the third-party static module record to discover shallow dependencies
  // in lieu of compiling the source, and otherwise behaves identically
  // to { source } descriptors.
  // When the loader _imports_ the module, it will construct a
  // [[ModuleEnvironmentRecord]] and [[ModuleExportsNamespace]] based
  // entirely on the bindings array of the third-party static module record.
  // If the third-party static-module-record has a true `needsImport`, the
  // loader will construct an `import` that resolves
  // the given import specifier relative to the module instance's full specifier
  // and returns a promise for the module exports namespace of the
  // imported module.
  // If the third-party static-module-record has a true `needsImportMeta`
  // property, the loader will construct an `importMeta` by the
  // same process as any { source } module.
  // `importMeta` will otherwise be `undefined`.
  // The loader will then initialize the module by calling the
  // `initialize` function of the third-party static module record,
  // giving it the module environment record and an options bag containing
  // either `import` or `importMeta` if needed.
  // The loader memoizes a promise for the module exports namespace
  // that is fulfilled as already specified in 262 for dynamic import,
  // where the behavior if the initialize function is async is analogous to
  // top-level-await for a module compiled from source.
  | {
    record: ThirdPartyStaticModuleRecord,

    // Properties to copy to the `import.meta` of the resulting module instance,
    // if the `record` has a `true` `needsImportMeta` property.
    importMeta?: Object,
  }

  // Use a static module record from the loader in which this loader
  // was constructed.
  // If the loader _loads_ the corresponding module, the static
  // module record will be synchronously available only if it is already
  // in the parent loader's synchronous memo.
  // Otherwise, loading induces the parent loader to load the module,
  // but does not induce the parent loader to load the module's transitive
  // dependencies.
  // This loader then resolves the shallow dependencies according
  // to its own resolveHook and loads the consequent transitive dependencies.
  // If the loader _imports_ the module, the behavior is equivalent
  // to loading for the retrieved static module record or third-party static
  // module record.
  | {
    record: string,

    // Properties to copy to the `import.meta` of the resulting module instance.
    importMeta?: Object,
  }

  // FOR SHARED MODULE INSTANCES:

  // To create an alias to an existing module instance in this loader:
  | {
    // A full specifier for another module in this loader.
    instance: string,
  }

  // To create an alias to an existing module instance given the full specifier
  // of the module in a different loader:
  | {
    instance: string,

    // A loader instance.
    // We do not preclude the possibility that loader is the same loader,
    // as that is possible to achieve in a `loadHook`.
    loader: Loader,
  }

  // To create an alias to an existing module instance given a module exports
  // namespace object that can pass a brand check:
  | {
    namespace: ModuleExportsNamespace,
  }

  // If the given namespace does not pass a ModuleExportsNamespace brandcheck,
  // the loader will not support live bindings to the properties of that
  // object, but will instead produce an emulation of a module exports namespace,
  // using a frozen snapshot of the own properties of the given object.
  // The module exports namespace for this module does not reflect any future
  // changes to the shape of the given object.
  | {
    // Any object that does not pass a module exports namespace brand check:
    namespace: ^ModuleExportsNamespace,
  };

type LoaderConstructorOptions = {
  // Every Loader has a reference to a global environment record that in
  // turn contains a new globalThis object, global contour, and three
  // specialized intrinsic evaluators: eval, Function, and Loader instances.
  // The new globalThis object contains a subset of the JavaScript language
  // intrinsics (to be defined in this proposal) and other globals must be
  // "endowed" to the loader explicitly with the `globals` option.
  // All of these evaluators close over the loader's global environment
  // record such that they evaluate code in that global environment.
  // When borrowGlobals is false, a the new Loader gets a new global
  // environment record.
  // When borrowGlobals is true, the new Loader will have the
  // same global environment record as associated with the Loader
  // constructor used to construct the loader.
  // To borrow the globals of an arbitrary loader, use that loader's
  // Loader constructor, like
  // new loader.globalThis.Loader({ borrowGlobals: true }).
  borrowGlobals: boolean,

  // Globals to copy onto this compartment's unique globalThis.
  // Constructor options with globals and borrowGlobals: true would be incoherent and
  // effect an exception.
  globals: Object,

  // The loader uses the resolveHook to synchronously elevate
  // an import specifier (as it appears in the source of a StaticModuleRecord
  // or bindings array of a third-party StaticModuleRecord), to
  // the corresponding full specifier, given the full specifier of the
  // referring (containing) module.
  // The full specifier is the memo key for the module.
  // TODO This proposal does not yet account for import assertions,
  // which evidence shows are actually import type specifiers as they were
  // originally designed and may need to also participate in the memo key.
  resolveHook: (importSpecifier: string, referrerSpecifier: string) => string,

  // The loader's load and import methods may need to load or initialize
  // additional modules.
  // If the loader does not have a module on hand,
  // it will first consult the `modules` object for a descriptor of the needed module.
  modules?: Record<string, ModuleDescriptor>,

  // If loading or importing a module misses the loader's memo and the
  // `modules` table, the loader calls the asynchronous `loadHook`.
  // Note: This name differs from the implementation of SES shim and a
  // prior revision of this proposal, where it is currently called `importHook`.
  loadHook?: (fullSpecifier: string) => Promise<ModuleDescriptor?>

  // A ModuleDescriptor can have an `importMeta` property.
  // The loader assigns these properties over the true `import.meta`
  // object for the module instance.
  // That object in turn has a null prototype.
  // However some properties of `import.meta` are too expensive for a host
  // to instantiate for every module, like Node.js's `import.meta.resolve`,
  // which can be avoided for modules that never utter `import.meta`
  // in their sources.
  // This hook gets called for any module that utters `import.meta`,
  // so some work can be deferred until just before the loader
  // initializes the module.
  importMetaHook?: (fullSpecifier: string, importMeta: Object) => void,
};

interface Loader {
  // Note: This single-argument form differs from earlier proposal versions,
  // implementations of SES shim, and Moddable's XS, which accept three arguments,
  // including a final options bag.
  constructor(options?: LoaderConstructorOptions): Loader;

  // Accessor for this loader's globals.
  // If borrowGlobals is true, globalThis is object identical to the incubating
  // loader's globalThis.
  // If borrowGlobals is false, globalThis is a unique, ordinary object
  // intrinsic to this loader.
  // The globalThis object initially has only "shared intrinsics",
  // followed by loader-specific "eval", "Function", and "Loader",
  // followed by any properties transferred from the "globals"
  // constructor option with the semantics of Object.assign.
  globalThis: Object,

  // Evaluates a program program using this loader's associated global
  // environment record.
  // A subsequent proposal might add options to either the loader
  // constructor or the evaluate method to persist the global contour
  // between calls to evaluate, making loaders a suitable
  // tool for a REPL, such that a const or let binding from one evaluation
  // persists to the next.
  // Subsequent proposals might afford other modes, like sloppy globals mode.
  // TODO is sloppyGlobalsMode only sensible in the context of the shim?
  evaluate(source: string): any;

  // load causes a loader to load module descriptors for the
  // transitive dependencies of a specified module into its
  // memo, but does not initialize any modules.
  // The load function is useful for tools like bundlers and importmap
  // generators that load a module graph in an emulated host environment
  // but cannot and should not emulate evaluation.
  async load(fullSpecifier: string): Promise<void>

  // import induces the asynchronous load phase and then initializes
  // the given module and any of its transitive dependencies that
  // have not already begun initialization.
  async import(fullSpecifier: string): Promise<ModuleExportsNamespace>;

  // TC53: some embedded systems hosts exclude promises and so require these
  // synchronous variants of import and load.
  // On hosts where these functions cannot succeed synchronously,
  // they must throw an error without the side effect of initializing any
  // additional modules.
  // For modules that have a top-level await, these would return
  // after the module first awaits but not after any subsequent promise queue
  // job.

  // If a host supports both importNow and import, they must share
  // a memo of full specifiers to promises for the module exports namespace.
  // importNow and loadNow followed by import or load must not accidentally
  // reinitialize the underlying module or produce different namespaces.

  importNow(fullSpecifier: string): ModuleExportsNamespace;

  loadNow(fullSpecifier: string): void;
}
```

### Design Rationales

An exported value named `then` can be statically imported, but dynamic import
confuses the module namespace for a thenable object.
The resolution of the promise returned by dynamic import, in this case, is the
eventual resolution of the thenable module.
And the eventual resolution is unlikely to be an intended effect.

Consider `thenable.js`:

```js
export function then(resolve) {
  resolve(42);
}
```

A neighboring module might dynamically import this.

```js
import('./thenable.js').then((x) => {
  // x will be 42 in this case, not a module namespace object with a then
  // function.
})
```

This is the behavior of a dynamic import today, despite it being surprising.

We have chosen to embrace this hazard since it would be worse to have
dynamic import and loader import behave differently.

However, with `loader.importNow`, a program can mitigate this hazard.
With `importNow`, the following program will not invoke the `then`
function exported by `'./thenable.js'`.

```js
await loader.load('./thenable.js');
const thenableNamespace = loader.importNow('./thenable.js');
```

[browserify]: https://browserify.org/
[import-map]: https://github.com/WICG/import-maps
[jest-ses-interaction]: https://github.com/facebook/jest/issues/11952
[jsdom]: https://www.npmjs.com/package/jsdom
[lava-moat]: https://github.com/LavaMoat/LavaMoat
[node-hmr]: https://github.com/nodejs/node/issues/40594
[parcel]: https://parceljs.org/
[ses-proposal]: https://github.com/tc39/proposal-ses
[ses-shim]: https://github.com/endojs/endo/tree/master/packages/ses
[AOT-SES]: https://github.com/DimensionDev/aot-secure-ecmascript
[webpack]: https://webpack.js.org/
[xs-compartments]: https://blog.moddable.com/blog/secureprivate/
[vm-context]: https://nodejs.org/api/vm.html#vm_vm_createcontext_contextobject_options
