# Bare Module Specifier Resolution in node.js

**Contributors:** Guy Bedford, Geoffrey Booth, Jan Krems, Saleh Abdel Motaal

## Motivating Examples

* A package (`react-dom`) has a dedicated entrypoint `react-dom/server` for code that isn't compatible with a browser environment.
* A package (`angular`) exposes multiple independent APIs, modeled via import paths like `angular/common/http`.
* A package (`lodash`) allows to import individual functions, e.g. `lodash/map`.
* A package is exclusively exposing an ESM interface.
* A package is exclusively exposing a CJS interface.
* A package is exposing both an ESM and a CJS interface.
* A project wants to mix both ESM and CJS code, with CJS running as part of the ESM module graph.
* A package wants to expose multiple entrypoints as its public API without leaking internal directory structure.
* A package wants to reference an internally aliased subpath, without exposing it publicly.
* A package wants to polyfill a builtin module and handle this through a package fallback mechanism like import maps.

## High Level Considerations

* The baseline behavior of relative imports should match a browser's with a simple file server.
  This implies that `./x` will only ever import exactly the sibling file "x" without appending paths or extensions.
  `"x"` is never resolved to `x.mjs` or `x/index.mjs` (or the `.js` equivalents).
* The primary compatibility boundary are bare specifiers. Relative and absolute imports can follow simpler rules.
* Resolution should not depend on file extensions, allowing ESM syntax in `.js` files.
* The directory structure of a module should be treated as private implementation detail.
* Validations should apply similarly to import maps in supporting forwards-compatibility with possible future features.

## `package.json` Interfaces

We propose two fields in `package.json` to specify entrypoints and internal aliasing of bare specifiers - [`"exports"`](#1-exports-field) and [`"imports"`](#2-imports-field).

> **For both fields the final names of `"exports"` and `"imports"` are still TBD, and these names should be considered placeholders.**

These interfaces will only be respected for bare specifiers, e.g. `import _ from 'lodash'` where the specifier `'lodash'` doesn’t start with a `.` or `/`.

Package _exports_ and _imports_ can be supported fully independently, and in both CommonJS and ES modules.

### 1. Exports Field

#### Example

Here’s a complete `package.json` example, for a hypothetical module named `@momentjs/moment`:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "exports": {
    "./": "./src/util/",
    "./timezones/": "./data/timezones/",
    "./timezones/utc": "./data/timezones/utc/index.mjs",
    "./core-polyfill": ["std:core-module", "./core-polyfill"]
  }
}
```

Within the `"exports"` object, the key string after the `'.'` is concatenated on the end of the name field, e.g. `import utc from '@momentjs/moment/timezones/utc'` is formed from `'@momentjs/moment'` + `'/timezones/utc'`. Note that this is string manipulation, not a file path: `"./timezones/utc"` is allowed, but just `"timezones/utc"` is not.

Keys that end in slashes can map to folder roots, following the [pattern in the browser import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes): `"./timezones/": "./data/timezones/"` would allow `import pdt from "@momentjs/moment/timezones/pdt.mjs"` to import `./data/timezones/pdt.mjs`.

- Using `"./"` maps the root, so `"./": "./src/util/"` would allow `import tick from "@momentjs/moment/tick.mjs"` to import `./src/util/tick.mjs`.

- Mapping a key of `"./"` to a value of `"./"` exposes all files in the package, where `"./": "./"` would allow `import privateHelpers from "@momentjs/moment/private-helpers.mjs"` to import `./private-helpers.mjs`.

- When mapping to a folder root, both the left and right sides must end in slashes: `"./": "./dist/"`, not `".": "./dist"`.

- Unlike in CommonJS, there is no automatic searching for `index.js` or `index.mjs` or for file extensions. This matches the [behavior of the import maps proposal](https://github.com/WICG/import-maps#packages-via-trailing-slashes).

The value of an export, e.g. `"./src/moment.mjs"`, must begin with `.` to signify a relative path (e.g. "./src" is okay, but `"/src"` or `"src"` are not). This is to reserve potential future use for `"exports"` to export things referenced via specifiers that aren’t relatively-resolved files, such as other packages or other protocols.

There is the potential for collisions in the exports, such as `"./timezones/"` and `"./timezones/utc"` in the example above (e.g. if there’s a file named `utc` in the `./data/timezones` folder).
Rough outline of a possible resolution algorithm:

1. Find the package matching the base specifier, e.g. `@momentjs/moment` or `request`.
1. Load its exports map.
1. If there is an exact match for the requested specifier, return the resolution.
1. Otherwise, find the longest matching path prefix. If there is a path prefix, return the resolution by applying the prefix.
1. Return an error - no mapping found.

In the future, the algorithm might be adjusted to align with work done in the [import maps proposal](https://github.com/domenic/import-maps).

#### Validations and Fallbacks

The following validations are performed for an exports start to resolve:

- The exports target must be a string and start with _"./"_ (URLs and absolute paths are not currently supported but may be supported in future).
- The exports target cannot backtrack below the package base.
- Exports targets cannot map into a nested node_modules path.

For directory resolutions the following validations also apply:

- Directory exports targets must end in a trailing _"/"_.
- Directory exports targets may not backtrack above the package base.
- Directory exports subpaths may not backtrack above the target folder.

Whenever there is a validation failure, any exports match must throw a Module Not Found error, and any validation failure context can be included in the error message.

Fallback arrays allow validation failures to continue checking the next item in the fallback array providing forwards compatiblitiy for new features in future based on extending these validation rules to new cases.

#### Usage

For a consumer, the above `@momentjs/moment` and `request` packages can be used as follows, assuming the user’s project is in `/app` with `/app/package.json` and `/app/node_modules`:

```js
import request from 'request';
// Loads file:///app/node_modules/request/request.mjs

import request from './node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import request from 'file:///app/node_modules/request/request.mjs';
// Loads file:///app/node_modules/request/request.mjs

import utc from '@momentjs/moment/timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import request from 'request/';
// Error: trailing slash not mapped

import request from 'request/request.mjs';
// Error: no such mapping

import moment from '@momentjs/moment/';
// Error: trailing slash not allowed (cannot import folders, only files)

import request from 'file:///app/node_modules/request';
// Error: folders cannot be imported, package.json only considered for bare imports

import request from 'file:///app/node_modules/request/';
// Error: folders cannot be imported, package.json only considered for bare imports

import utc from '@momentjs/moment/timezones/utc/'; // Note trailing slash
// Error: folders cannot be imported (there is no index.* magic)
```

### 2. Default Field

The `default` package.json field can be thought of as the "new main" for Node.js.

Like exports, the target value can be a string or array, must start with './' and be an exact file path - no extension searching applies.

When there is no "default" field, the traditional "main" and its associated lookup will be used, including the default `index.js` main.

"default" can also be set to _false_ to indicate that there is no main entry point for the package.

#### Example

```js
{
  "main": "./index.js",
  "default": "./main.js"
}
```

will load `pkg/main.js` from both CommonJS and ESM importers `require('pkg')` or `import 'pkg'`.

```js
{
  "main": "./index.js",
  "default": false
}
```

will give an error - "No main entry point found for pkg" when trying to `require('pkg')` or `import 'pkg'`.

### 3. Conditional Mapping

Conditional mapping is an extension of the `"exports"` and `"default"` features that allows defining different mappings between different environments, for example having a different module path between Node.js and the browser.

Conditional mappings are defined as _objects_ in the target slot for both `"exports"` and the `"default"` field.

The object has keys which are _condition names_, and values which correspond to the mapping target.

Condition names are matched in a priority order. In Node.js the following conditions apply in priority order:

1. `"require"`: Indicates we are resolving from a CommonJS importer.
2. `"node"`: Indicates we are in a Node.js environment.

> Note: Using a "require" condition opens up the dual specifier hazard in Node.js where a package can have different instances between CJS and ESM importers. There is an argument that this condition is an opt-in behaviour to the hazard which is less risky than the main concerns of the hazard which were non-intentional cases. It is still not clear if this condition will get consensus, and it may still be removed.

> Node: It could also be worthwhile considering object order over priority order. This would shift the source of truth of the entry priority from the resolver to the package author, which could be beneficial, but should be discussed further.

If no condition is matched, the package fallback applies. If the matched condition target is invalid, the fallback will still not apply, although individual condition targets can themselves use array fallbacks.

Other resolvers are free to define their own conditions to match. Eg it is expected that users will use a `"browser"` condition name for browser mappings.

#### Example

Taking the previous moment example we can provide browser / Node.js mappings for some modules with the following:

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "default": {
    "node": "./dist/index.js",
    "browser": "./dist/index-browser.js"
  },
  "exports": {
    "./": {
      "node": "./src/util/",
      "browser": "./src/util-browser/"
    },
    "./timezones/": "./data/timezones/",
    "./timezones/utc": "./data/timezones/utc/index.mjs",
    "./core-polyfill": {
      "node": ["std:core-module", "./core-polyfill"],
      "browser": "./core-polyfill-browser"
    }
  }
}
```

Note we are able to share some exports with both Node.js and the browser and others split between the environments.

#### Dual package example

For an example of a package that wants to support legacy Node.js, `require()` and `import` (noting that this is an opt-in to the instancing hazard, and pending consensus), we can use the `"require"` condition which will always beat the `"node"` condition as it is a more specific condition:

```js
{
  "type": "module"
  "main": "./index-legacy.cjs",
  "default": [{
    "require": "./index.cjs"
  }, "./index.js"],
  "exports": {
    "./features/": [{
      "require": "./features-cjs/"
    }, "./features/"]
  }
}
```

#### Combined dual package browser example

To show how conditions handle combined scenarios, here is another example of a package that supports `require()`, `import()` and a separate browser entry:

```js
{
  "type": "module",
  "main": "./index.cjs"
  "default": {
    "require": "./index.cjs",
    "browser": "./index-browser.js",
    "node": "./index.js"
  }
}
```

Similarly, the above could apply to any exports as well.

### 4. Imports Field

Imports provide the ability to remap bare specifiers within packages before they hit the node_modules resolution process.

The current proposal prefixes all imports with `#` to provide a clear signal that it's a _symbolic specifier_ and also to prevent packages that use imports from working in any environment (runtime, bundler) that isn't aware of imports.

> **Whether this restriction is maintained in the final proposal, or what exact symbol is used for `#` is still TBD.**

#### Example

For the same example package as provided for `"exports"`, consider if we wanted to make the `timezones` implementation something that is only referenced internally by code within `@momentjs/moment`, instead of exposing it to external importers.

```js
{
  "name": "@momentjs/moment",
  "version": "0.0.0",
  "type": "module",
  "main": "./dist/index.js",
  "imports": {
    "#timezones/": "./data/timezones/",
    "#timezones/utc": "./data/timezones/utc/index.mjs",
    "#external-feature": "external-pkg/feature",
    "#moment/": "./"
  }
}
```

As with package exports, mappings are mapped relative to the package base, and keys that end in slashes can map to folder roots.

The resolution algorithms remain the same except `"imports"` provide the added feature that they can also map into third-party packages that would be looked up in node_modules, including to subpaths that would be in turn resolved through `"exports"`. There is no risk of circular resolution here, since `"exports"` themselves only ever resolve to direct internal paths and can't in turn map to aliases.

The `"imports"` that apply within a given file are determined based on looking up the package boundary of that file.

#### Usage

For the author of `@momentjs/moment`, they can use these aliases with the following code in any file in `@momentjs/moment/*.js`, provided it matches the package boundary of the `package.json` file:

```js
import utc from '#timezones/utc';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from './data/timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from '#timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs

import utc from '#moment/data/timezones/utc/index.mjs';
// Loads file:///app/node_modules/@momentjs/moment/data/timezones/utc/index.mjs
```

The following don’t work - please note that **error messages and codes are TBD**:

```js
import utc from '#timezones/utc/';
// Error: trailing slash not mapped

import unknown from '#unknown';
// Error: no such import alias

import timezones from '#timezones/';
// Error: trailing slash not allowed (cannot import folders, only files)

import utc from '#moment';
// Error: no mapping provided (folder mappings require subpaths)
```

### Prior Art

* [`package.json#browser`](https://github.com/defunctzombie/package-browser-field-spec)
* [Import Maps](https://github.com/domenic/import-maps)
* [`package.json#mimes`](https://github.com/nodejs/modules/pull/160)
* [node.js ESM resolver spec](https://github.com/nodejs/ecmascript-modules/pull/12)
